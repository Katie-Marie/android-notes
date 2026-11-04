# `0xFF` means 'not available', and it's also the top byte of every small negative number

## The gotcha

Say a device on a CAN bus sends a weight in kilograms as two bytes with a factor of 1, and the row in its spec, the line in the protocol document that describes this one field, says that and nothing else. The factor is what you multiply the raw number by to get the unit, so with a factor of 1 the two bytes are the kilograms. The weight can go below zero, because a scale that's zeroed with a load on it (tared) reads negative once the load comes off.

The decoder, the code that turns those bytes into a number, did what J1939 expects. It joined the two bytes into a number from 0 to 65535, and before using the number it checked the top byte, because J1939 says a top byte of `0xFE` means "error" and `0xFF` means "not available". For either one it showed the placeholder, the dash a screen uses for a value it doesn't have.

Run a small negative weight through that and it disappears. -1 is `0xFFFF`, so the check says not available. -100 is `0xFF9C`, same again. A large negative gets past the check and comes out large and positive: -30000 is `0x8AD0`, which reads as 35536 kg.

I didn't catch this on a screen. I caught it reading the spec row next to the decoder and asking what a negative would do, then printing a table before I said anything.

## Two bytes, two ways to read them

A byte is 8 bits and holds one of 256 patterns. Hex writes a byte as two digits from `00` to `FF`, which is 0 to 255, so a two-byte value is four hex digits, like `0xFF9C`. The top byte is the first two of those digits, `FF`, and the top bit is the leftmost bit of the top byte. Two bytes give 16 bits and 65536 patterns, and the whole question is what each pattern means. There are two common answers.

**Unsigned** says the patterns are 0 to 65535 in order. `0x0000` is 0, `0xFFFF` is 65535, and nothing is negative.

**Two's complement**, the usual way to store a signed number, splits them in half. Patterns with the top bit clear, `0x0000` to `0x7FFF`, are 0 to 32767, the same as before. Patterns with the top bit set, `0x8000` to `0xFFFF`, are negative, and the value is the unsigned reading minus 65536. So `0xFFFF` is 65535 - 65536 = -1, `0xFF9C` is 65436 - 65536 = -100, and `0x8000` is -32768, the most negative value there is. To negate a number, flip every bit and add one: 100 is `0x0064`, flipped is `0xFF9B`, plus one is `0xFF9C`. Plain addition then just works, and any carry falls off the top.

Same 16 bits, two numbers. It's the mirror of [the serial bytes post](2026-11-05-from-serial-bytes-to-the-screen.md), where one byte of 200 was read as -56 because Kotlin's `Byte` is signed. Nothing in the bytes says which reading the sender meant. Only the spec can.

## What J1939 reserves

CAN is the two-wire bus that vehicles use to pass short messages between controllers, and J1939 is SAE's standard for using it on trucks and heavy equipment. J1939-71, the part that lays out the data, keeps the top of every field's range for signalling rather than values. For a two-byte field:

```text
top byte    two-byte value     meaning
00 to FA    0 to 64255         a valid value
FB          64256 to 64511     parameter-specific indicator
FC to FD    64512 to 65023     reserved
FE          65024 to 65279     error
FF          65280 to 65535     not available
```

J1939 also sends the low byte first, so `0xFF9C` arrives as `9C` then `FF`, and the top byte is the second one you read.

A decoder's availability check is right to look at the top byte. It only works because a real reading never gets up there: a field with a factor of 1 tops out at 64255 kg, and the sender keeps the rest for signalling. The whole scheme assumes unsigned.

## Why a signed field can't use it

In two's complement, every number from -1 to -256 has `FF` as its top byte, and every number from -257 to -512 has `FE`. The small negatives, the ordinary ones a tared scale produces, land exactly on the two bytes J1939 keeps for "error" and "not available". The convention and the sign want the same bits.

So a field is one or the other. If it's unsigned, the J1939 markers apply and a negative is impossible. If it's signed, the markers can't apply, and the field needs a marker of its own, a pattern outside any real reading, and the spec has to name it. A row that gives a unit and a factor has told you neither.

## Print the table before you claim it

Both readings fit in a few lines of Python, so you can print what each value does instead of arguing about it. `lo` and `hi` are the two bytes in wire order. `hi << 8 | lo` moves the high byte up 8 bits and drops the low byte in underneath, which is how a decoder joins them, and `raw & 0x8000` is true when the top bit is set:

```python
def unsigned(lo, hi):
    return hi << 8 | lo

def signed(lo, hi):
    raw = hi << 8 | lo
    return raw - 65536 if raw & 0x8000 else raw

def j1939_available(hi):
    return hi < 0xFE

print("  value  wire   unsigned  J1939 says      signed")
for value in (100, -1, -100, -256, -257, -512, -513, -30000):
    lo, hi = value & 0xFF, (value >> 8) & 0xFF
    says = "ok" if j1939_available(hi) else "not available"
    print(f"{value:>7}  {lo:02X} {hi:02X}  {unsigned(lo, hi):>9}  {says:<14}{signed(lo, hi):>8}")
```

```text
  value  wire   unsigned  J1939 says      signed
    100  64 00        100  ok                 100
     -1  FF FF      65535  not available       -1
   -100  9C FF      65436  not available     -100
   -256  00 FF      65280  not available     -256
   -257  FF FE      65279  not available     -257
   -512  00 FE      65024  not available     -512
   -513  FF FD      65023  ok                -513
 -30000  D0 8A      35536  ok              -30000
```

Every value from -1 to -512 vanishes behind the placeholder. From -513 down the check passes, and the unsigned column is what reaches the screen. "I think it reads unsigned" is an opinion. The table is numbers you can quote.

## Read the same bits as signed

The bytes were right. The fix is a reinterpretation, not a conversion: read the same 16 bits as signed.

```kotlin
val raw = (hi shl 8) or lo            // 0 to 65535, as the decoder already builds it
val weight = raw.toShort().toInt()    // the same 16 bits, read as two's complement
```

`Int.toShort()` keeps the least significant 16 bits and reads them as a signed `Short`, so 65436 becomes -100 and 35536 becomes -30000. `toInt()` widens it back to an `Int` and keeps the sign. The J1939 top-byte check is skipped for this field, because for a signed field `0xFFFF` is -1 and nothing else.

Any test that pinned the old rule goes with it. A test that says `FF FF` means not available is now wrong for this field, so it's deleted.

## A factor isn't a type

A row with a unit and a factor and nothing else is incomplete for any field that can go negative. Ask the owner of the protocol three things: how the bytes encode the number, unsigned or two's complement; what range of values is real; and which pattern means "no value". A factor tells you how to scale a number. It doesn't tell you which number you have.
