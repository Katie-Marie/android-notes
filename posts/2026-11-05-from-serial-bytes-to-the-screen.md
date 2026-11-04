# From serial bytes to the screen

## The gotcha

One of the controllers I talk to over USB serial sends plain ASCII text, one message per line, each ending in a carriage return. I'd added a log line for any byte that wasn't printable text, to see what else the device was sending, and it reported a byte with the code 65480.

A byte only goes up to 255. The byte on the wire was 200 (`0xC8`), and my code turned it into a character like this:

```kotlin
val char = byte.toInt().toChar()
```

Kotlin's `Byte` is **signed**. It holds -128 to 127, so the bits of 200 read as -56. `toInt()` keeps the value -56 by filling the top 24 bits with the sign bit, which gives `0xFFFFFFC8`. `toChar()` then keeps the low 16 bits, `0xFFC8`, and that's 65480.

```text
                                bits, low byte on the right    value
on the wire                                       1100 1000    200
as a Kotlin Byte                                  1100 1000    -56
toInt()             1111 1111 1111 1111 1111 1111 1100 1000    -56
toInt().toChar()                        1111 1111 1100 1000    65480
toInt() and 0xFF    0000 0000 0000 0000 0000 0000 1100 1000    200
```

The 8 bits never change. What changes is what fills the space above them.

The fix is to drop the sign before the byte is used for anything:

```kotlin
val code = byte.toInt() and 0xFF   // 0 to 255
val char = code.toChar()
```

`and 0xFF` keeps the low 8 bits and zeroes the rest, so 200 stays 200. Kotlin's own `UByte` does exactly this: the standard library writes `UByte.toInt()` as `data.toInt() and 0xFF`. So `byte.toUByte().toInt()` is the same thing with a name on it.

What caught me out is that `toInt().toChar()` looks like the careful version. `Byte.toChar()` is deprecated, and the replacement its deprecation note suggests is `toInt().toChar()`. That compiles without a warning and still gives the wrong code for any byte above 127.

## A read is a chunk, not a message

The serial library hands over whatever bytes have arrived, which can be half a line or a line and a half. Say the device sends two temperature readings:

```text
the device sends   TEMP,21.5\rTEMP,21.6\r
read 1             TEMP,2
read 2             1.5\rTEMP,21
read 3             .6\r
lines handed on    "TEMP,21.5"   "TEMP,21.6"
```

So the parser keeps a buffer and only passes a line on when its carriage return arrives:

```kotlin
private const val NUL = 0x00
private const val ACK = 0x06
private const val CR = 0x0D

class LineParser(private val onLine: (String) -> Unit) {
    private val buffer = StringBuilder()

    fun read(data: ByteArray) {
        data.forEach { byte ->
            val code = byte.toInt() and 0xFF
            when (code) {
                CR -> {
                    if (buffer.isNotEmpty()) onLine(buffer.toString())
                    buffer.clear()
                }
                ACK -> onLine("ACK")
                in 32..126 -> buffer.append(code.toChar())
                NUL -> {}
                else -> logUnexpected(code)
            }
        }
    }
}
```

`32..126` is a Kotlin `IntRange`, the printable ASCII characters from space to `~`. Once the byte is 0 to 255, `in 32..126` reads straight off an ASCII table.

## Every byte gets a job or a log line

Printable text goes into the buffer. Control bytes have jobs: a carriage return ends a line, and ACK (`0x06`) is the device acknowledging a command, so it's passed on as a message of its own. NUL bytes are ignored. Anything else has no place in a text protocol, so `logUnexpected` writes it in hex and it stays out of the message. Throttle that log, because a device that sends a burst of binary can bury everything else in it.

## Parse at the boundary

A complete line is still just text. Check the field count, then convert each field with `toIntOrNull()` or `toDoubleOrNull()`, so a corrupted field gives you a `null` to deal with instead of an exception. Nothing past this point sees raw text, only typed values.

## Cross to the main thread on purpose

The USB serial library I use calls back on its own reader thread, not the main thread. If the value goes into LiveData, use `postValue`, because `setValue` throws if you call it off the main thread. `postValue` hands the value over to the main thread, and if several arrive before it runs, only the last one is delivered. For a device that streams readings, that's what you want: the screen shows the latest reading, not a queue of old ones.

Skip a value that equals the current one, too, so the UI doesn't redraw for a reading that didn't change. A `StateFlow` does both for you: its methods are thread-safe, and it drops an update that equals the current value.

## A wrong number can start at the bottom

Each step promises the next one something: real byte values, then whole lines, then typed fields, then the latest value on the main thread.

```text
serial bytes, on the library's reader thread
  |  and 0xFF              real byte values, 0 to 255
  |  buffer to \r          whole lines
  |  split, toIntOrNull    typed fields
  |  postValue             the latest value, on the main thread
  v
the screen
```

When something on screen looks wrong, I walk back through those promises in order. My 65480 was the very first one broken, and I only saw it because I'd logged the bytes that didn't fit.
