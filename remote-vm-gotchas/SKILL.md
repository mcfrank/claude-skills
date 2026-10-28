---
name: remote-vm-gotchas
description: Common footguns when running long jobs on remote VMs over SSH (GCP, AWS, Sherlock, any cloud). Use as a sanity check before kill/relaunch operations, before scp'ing into a running script, or when an SSH session keeps dropping unexpectedly.
---

# Remote VM workflow gotchas

Bite-sized rules from running long jobs over `ssh` and `gcloud compute ssh`.

## Don't scp a script that's currently executing

`gcloud compute scp run_thing.sh vm:path/run_thing.sh` while a `run_thing.sh` process is mid-execution will corrupt the running shell. bash re-reads its source at line boundaries; if the file changed (even just byte length), the next read can land mid-token and throw "unexpected EOF" or "syntax error".

This will crash a long-running launcher right between phases (e.g., between sampling and post-processing).

Workarounds:
- Wait for the current invocation to finish before patching
- Patch into a new filename (`run_thing_v2.sh`) and switch the caller
- Make the change only in the next launch's environment

## pkill -9 of session ancestors kills the SSH session

If you `gcloud compute ssh ... --command="pkill -9 -f some_pattern"` and the SSH session itself is somewhere in the process tree being walked, ssh dies. You'll see:

```
ERROR: (gcloud.compute.ssh) [/usr/bin/ssh] exited with return code [255].
```

This is harmless — the pkill ran, just before SSH could send a clean exit. Reconnect to verify the kill worked:

```
gcloud compute ssh <vm> --command="ps -eo cmd | grep <pattern> | grep -v grep || echo clean"
```

## Detached background launches

The robust pattern for "fire and forget" on a remote VM:

```bash
gcloud compute ssh <vm> --command="cd ~/path && \
  ENV_VAR=val nohup bash run_thing.sh > thing.log 2>&1 < /dev/null & disown"
```

All four pieces matter:

| Piece | Why |
|---|---|
| `nohup` | survives `SIGHUP` when ssh disconnects |
| `> log 2>&1` | captures all output (else SSH waits on stdout/stderr drain) |
| `< /dev/null` | closes stdin (else SSH waits on stdin) |
| `& disown` | backgrounds AND removes from bash's job table |

Without `< /dev/null` and `& disown`, the SSH session can still hang waiting for IO, or the process can die when ssh closes — depending on shell version.

## Background tasks vs. harness notifications

If your runner harness has a "background task" feature (like Claude Code's `Bash(..., run_in_background: true)`), use it for **commands that return useful output**. For pure "fire on remote and forget" launches, use the inline `nohup ... & disown` pattern above — the wrapper exits in <1s anyway, so backgrounding the wrapper itself is pointless.

For "watch this thing until done" patterns where the thing takes hours, prefer scheduled wakeups (`ScheduleWakeup` or similar) over background-bash-polling. You get fresh process/log state each time.

## scp --recurse with multiple sources flattens

```
gcloud compute scp file1.sh file2.sh file3.R vm:dest/ --recurse
```

flattens all sources into `dest/`, losing the local path hierarchy. To preserve `gcp/run_fit.sh → vm:standard_model_2/gcp/run_fit.sh`, do one scp per file with the explicit destination path. Annoying but reliable.

## Symlinks vs mkdir -p order

Code that does:
```
mkdir -p $X/sub
[ -L $X/sub ] || ln -sfn $TARGET $X/sub
```

will **never** create the symlink — `mkdir -p` always creates `$X/sub` as a real directory first, so the `[ -L ]` check is always false (it's a directory, not a symlink) and `ln -sfn` then refuses to clobber a non-empty dir.

Robust version (bash):
```
mkdir -p "$X" "$TARGET"
if [ -e "$X/sub" ] && [ ! -L "$X/sub" ]; then
  # Move contents into $TARGET without overwriting anything (dotfiles too)
  (shopt -s dotglob nullglob; mv -n "$X/sub"/* "$TARGET"/) || true
  # Remove the old dir only if it is now empty. Never rm -rf here: anything
  # left is a name collision with $TARGET and needs a human decision.
  if ! rmdir "$X/sub"; then
    echo "$X/sub still has entries that collide with $TARGET; resolve by hand" >&2
    exit 1
  fi
fi
ln -sfn "$TARGET" "$X/sub"
```

This idiom catches the case where a previous run created a real dir: it moves the contents over, refuses to delete anything that didn't move (`mv -n` skips names already in `$TARGET`, and an `rm -rf` fallback would silently destroy those), then sets up the symlink.

## Disk space silently fills /tmp

Many tools write intermediate output to `/tmp` (in GB) without warning. Stan, R, Python's tempfile, etc. /tmp is often on the boot disk, which is small by default.

Before launching long jobs: `df -h /tmp /`. If the boot disk is < 100 GB and you're about to write log_lik or similar per-obs data: stop and provision more disk. Online-resize is supported on most cloud disks:

```
# GCP
gcloud compute disks resize <name> --size=250GB --zone=<zone>
# Then on the VM
sudo growpart /dev/sda 1 && sudo resize2fs /dev/sda1
```

## Mac binaries leak to Linux VMs via rsync

If you rsync a project directory from a Mac to a Linux VM, **compiled binaries (Stan models, native extensions) get copied** and will segfault on Linux x86_64 because they were built for Mac arm64.

Always exclude `.o`, `.so`, model-binary files in your rsync, or recompile after sync. Better: rsync only source files and let the remote rebuild.
