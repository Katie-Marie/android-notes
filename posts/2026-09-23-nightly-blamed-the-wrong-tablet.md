# A nightly test run that stopped early and blamed the wrong tablet

## The gotcha

We run the app's instrumented tests overnight on a rack of real Android tablets. A shell script walks the roster one tablet at a time: install, smoke test, run the suite, write a row into a summary, move on.

One morning the run was red. The summary had exactly one line in it:

```
tablet 1: UNREACHABLE - unable to connect
```

That tablet had been unplugged for a while and everyone knew it. So the run looked like a known problem, and the reason it gave was true. It just wasn't the reason the run failed.

What had actually happened: tablet 2 ran its whole suite and passed, the script then died partway through tidying up after it, and tablets 3, 4 and 5 were never touched. The passing results were thrown away with everything else. The one line in the summary was there because it happened to be written before the script died.

Digging into it turned up four more bugs. They fall into two groups, and the second group is why the first group was expensive.

## Nothing bounded the failure

**`grep` finding nothing is not an error, but the shell thinks it is.** The script runs under `set -euo pipefail`. It flips a tablet onto a different USB bus, then reads the tablet's address out of the log of that flip:

```sh
addr="$(grep -oE '[0-9.]+:5555' "$switch_log" | tail -1)"
if [ -z "$addr" ]; then
    # report it, put the tablet back, carry on
fi
```

When the flip can't report an address, `grep` exits 1. `pipefail` promotes that to a failed pipeline even though `tail` succeeded, `errexit` sees a failed command, and the script is gone. The `if` on the very next line was written for exactly that case, and it had never once run. `|| true` on the pipeline fixes it.

**One tablet's death ended the night.** The loop body ran inline, so any unhandled failure in one tablet's turn took the whole run with it. The fix is to give each tablet its own subshell, and the obvious way to do that is wrong:

```sh
# does NOT contain the failure: errexit is disabled inside the subshell
( run_tablet "$device" ) || rc=$?
```

Bash suppresses `errexit` for a command on the left of `||`, and that suppression reaches inside the subshell. Commands after a failing one keep running. Putting an explicit `set -e` as the subshell's first statement doesn't override it either, which I only believed after testing it. This is the form that contains the failure and keeps `errexit` live inside:

```sh
set +e
( set -e; run_tablet "$device" )
rc=$?
set -e
```

The subshell isn't in a checked context, so the parent survives, and the explicit `set -e` re-arms `errexit` for the body. I checked it on bash 3.2 and 5.3 and they behave the same.

**A wait with no clock on it.** When a test leg fails, the script reboots the tablet and retries. It waited like this:

```sh
adb -s "$device" reboot
adb -s "$device" wait-for-device || true
while [ "$waited" -lt "$limit" ]; do ...
```

`adb wait-for-device` blocks forever. `|| true` can't rescue a command that never returns to fail, and the loop with the timeout on it sits underneath, unreachable. A tablet that didn't come back parked the run on that line for hours. The fix is `timeout` around it.

## The report said things that weren't true

**A skipped test counted as a failed one.** The suite skips its hardware tests with `assumeTrue` when the hardware isn't attached. The Android Gradle Plugin writes those skips into the JUnit XML as `<failure>` elements carrying an `AssumptionViolatedException`, not as `<skipped>`. Code that counts raw `<failure>` elements reads three skipped tests as three failures. A tablet that simply had nothing plugged into it would report a red run for hardware that was absent, which is the opposite of what a skip means.

**An explanation that couldn't be true.** An unreachable tablet was always told it might be "on the other side of its switch". Only tablets on a switch can be, and a bare tablet that's gone is just gone. I went looking at switch wiring for a tablet that didn't have any, because the message offered me that as an option.

## Which group cost more

The unbounded failures were the real bugs, and each fix is one or two lines. On their own they'd have been found in a day, because a run that dies in an obvious place gets looked at.

What made them expensive is that the reporting absorbed them. A run that died mid-loop still produced a summary, a report page and an exit code, so it looked like a run that had finished and failed. A skip counted as a failure reads as a test problem, so you go and look at the test. An unreachable row that names a cause sounds like a diagnosis, so you believe it. The rack was testing one tablet out of five and the reports never said so, because nothing in them was false enough to notice.

The thing I'd do differently is not about `set -e` at all. Every one of those reports was assembled from whatever the script had got to before it stopped, and none of them had any way to say "this is partial". A summary that counted its own rows against the roster would have caught all of it on the first night.
