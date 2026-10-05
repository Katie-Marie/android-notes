# `flock` shares its lock with everything your command starts

## The gotcha

Our test tablets each sit on a USB switcher, and one controller drives every switcher through a single serial port. Two flips at the same time could interleave their commands and press the wrong switcher, so the flip script takes a lock on the port first. It re-runs itself under `flock`, so the whole flip happens inside the lock:

```sh
if [ -z "${LOCK_HELD:-}" ]; then
    LOCK_HELD=1
    export LOCK_HELD
    flock -w "$LOCK_WAIT" -E 99 /tmp/switch.lock "$0" "$@"
    rc=$?
    [ "$rc" -eq 99 ] && echo "FAILED: another flip still holds $PORT"
    exit "$rc"
fi
```

A second flip waits its turn. If it never gets the lock, `-E 99` gives it an exit code of its own, so a flip that never started doesn't read as one that went wrong.

One morning every hardware test in the nightly run had errored before it started, and every flip had printed the same line:

```
FAILED: another flip still holds /dev/ttyACM0
```

The run works through the tablets one at a time, so there was no other flip to be holding it.

## Who had it

Every file a process has open shows up as a symlink under `/proc/<pid>/fd`, so you can ask the kernel who has the lock file open:

```sh
$ find /proc/[0-9]*/fd -lname /tmp/switch.lock 2>/dev/null
/proc/<pid>/fd/3
```

It was the adb server. Its parent was pid 1, so whatever started it had already exited.

The flip script had started it. When a tablet is on the USB bus but missing from `adb devices`, the flip restarts the adb server and tries again:

```sh
adb kill-server
adb start-server
```

That runs inside the lock, like the rest of the flip. It's there to make flips more reliable, and it's what broke them.

## Why it kept the lock

A lock taken with `flock` belongs to the **open file**, not to the process that took it. Every copy of the file descriptor shares the same lock, and the lock is only released when the last copy is closed.

Copies are easy to make without meaning to. `fork` gives the child a copy of every descriptor its parent has open, and `exec` keeps them unless they're marked close-on-exec, which `flock` doesn't do. So `flock <file> <command>` takes the lock, forks, and waits, and the child runs your command with its own copy of the descriptor. Your command's children get one too, and so do theirs.

`adb start-server` starts the server in the background and returns. The server carries on after the client exits and after the flip ends, and it got a copy of descriptor 3 like everything else. When the flip finished and `flock` exited, the lock stayed with a process that had no idea it was holding anything. Every flip after that waited for the lock, gave up, and blamed another flip. The message named the only holder the script could imagine.

## The fix

`-o` closes the descriptor in the child before it runs your command:

```sh
flock -w "$LOCK_WAIT" -o -E 99 /tmp/switch.lock "$0" "$@"
```

The flip still holds the lock from start to finish, because `flock` keeps its own copy and waits for the command to exit. The man page says as much under `-F`, which runs the command without forking: it can't be combined with `-o` "as there would otherwise be nothing left to hold the lock".

## Proving it

Each `sh -c` below starts a `sleep` in the background and exits straight away, and `flock` exits with it:

```sh
$ flock /tmp/a.lock sh -c 'sleep 60 &'
$ flock -n /tmp/a.lock echo free || echo "still locked"
still locked

$ flock -o /tmp/b.lock sh -c 'sleep 60 &'
$ flock -n /tmp/b.lock echo free || echo "still locked"
free
```

Without `-o`, the `sleep` has the lock. The `find` from earlier turns it up, holding the lock file open the same way the adb server was.

## The lock outlives the script

This is the same family as the [`adb tcpip` post](2026-09-23-adb-tcpip-drops-the-transport.md): the lock works, and the assumption about **who** holds it is wrong. The script treated the lock as its own, released when it exited. It belongs to a descriptor, and every process the script starts gets a copy. Most of them exit with the script, so you never notice. A daemon doesn't, and adb starts one whenever it finds no server running, so any adb call inside a lock can take the lock with it.
