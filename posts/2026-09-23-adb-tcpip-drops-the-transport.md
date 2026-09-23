# `adb tcpip` drops the transport you're about to read from

## The gotcha

`adb tcpip 5555` restarts `adbd` on the tablet so it listens on TCP. Restarting it drops the USB transport you were talking over. Any `adb` command you send on that transport in the next few seconds comes back empty, and if you are reading a value rather than checking an exit code, empty looks like an answer.

I hit this on a rack of test tablets. Each tablet sits on a USB switcher so the host can hand it over to other hardware, and once a tablet moves off the host's USB bus we talk to it over Wi-Fi instead. The sequence is: arm network adb, flip the switch, reconnect over the network. Reconnecting needs the tablet's IP address, so we read the address off the tablet while USB still works.

We read it one line too late.

```sh
adb -s "$serial" tcpip 5555
sleep 2
ip=$(adb -s "$serial" shell ip -o -4 addr show \
    | grep -v ' lo ' | awk '{print $4}' | cut -d/ -f1 | head -1)
```

On one tablet `ip` came back empty every time. The report said "tablet reported no IP", the code took that as "this tablet has no network address", and it skipped the hardware tests. That tablet had been sitting on the office Wi-Fi the whole time.

## Proving it

The same read, seconds apart, on a tablet with a working Wi-Fi connection:

```
--- before tcpip ---
<the tablet's address>

--- adb tcpip 5555 ---
restarting in TCP mode port: 5555

--- after tcpip + sleep 2 ---
<nothing>
```

Two seconds was not long enough for the USB transport to come back. On another tablet in the same rack it was long enough, which made it look like a fault in one tablet rather than a fault in the order of the code.

## The fix

Read the address first.

```sh
ip=$(adb -s "$serial" shell ip -o -4 addr show \
    | grep -v ' lo ' | awk '{print $4}' | cut -d/ -f1 | head -1)
adb -s "$serial" tcpip 5555
sleep 2
```

Six lines moved, no new logic. The tempting fix is to raise the `sleep`, but a longer sleep only widens the window you are racing in. Moving the read out of the window removes the race.

## Empty is a value

`adb shell` on a transport that has gone away doesn't fail, it just prints nothing and exits. Capture its stdout into a variable and you get an empty string, which is a perfectly good value for the next `if` to act on.

That is the same shape as the SharedPreferences post further down this list: the call works, and the assumption about **when** it works is wrong. Here it cost a hardware test suite that skipped itself while reporting a reason that was not true.
