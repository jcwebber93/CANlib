# Extending a generic-message parameter table

Background for the constraint list at the end of `src/CanMessageGenericTables.h`. All three limits below
were hit while adding the FOC current-loop parameter to `M569Point1Params`, and **none of them announces
itself** — two produce silently wrong behaviour and one produces a misleading error.

Read this before adding a parameter to any `*Params` table.

---

## <a name="parammap"></a>1. A table may not exceed 20 parameters

`CanMessageGeneric` is:

```cpp
uint32_t requestId : 12,
         paramMap  : 20;      // one bit per table entry
uint8_t  data[60];
```

`paramMap` is a **20-bit bitfield with one bit per table entry**. `CanMessageGenericConstructor` sets
`msg.paramMap |= paramBit` walking the table in order, and `CanMessageGenericParser::FindParameter()`
consumes it with `paramMap >>= 1`.

Exceeding 20 **neither fails to build nor fails at run time**. The surplus entries' bits are truncated
out of the bitfield, so those parameters are accepted by the sender and simply never arrive.

`M569Point1Params` was already at exactly 20 when this was discovered, which meant the first attempt —
three separate letters for the current-loop gains — was unsendable from the moment it was written. The
length check happened to fire first and mask it.

A `static_assert` now makes this a compile error. Of the 22 tables in the header, `M569Point1Params` is
the only one anywhere near the limit.

**If a table is full:** pack related values into one `FLOAT_ARRAY_PARAM` (one bit, not three), or
displace a retired entry — see [3](#retired-letters).

---

## <a name="gm-letters"></a>2. `G` and `M` cannot be used at all, in any table

RepRapFirmware's `StringParser` treats them as the start of a new command and **stops scanning parameters
there**:

```cpp
else if (c2 == 'G' || c2 == 'M')
{
    break;
}
```

So `M569.1 P52.0 ... G25` parses as an `M569.1` followed by a `G25`, and reports:

```
Warning: G25: Command is not supported
```

Found on hardware, not by inspection — the misleading part is that the `M569.1` itself succeeds.

**`T` is fine** despite also being a command letter; the parser special-cases it, which is why
`M569Point1Params`' existing `T` parameter works. That exception is what makes command letters look safe.

---

## <a name="retired-letters"></a>3. A retired letter is not a free letter

When a parameter is withdrawn, its entry is normally kept in place **in lowercase** so that every later
parameter keeps its bit position. `M569Point1Params` carried `FLOAT_PARAM('h')` for exactly this reason —
lowercase specifically so it can never match again.

Reviving such a slot with a new meaning would silently give any stale letter in an existing config file
the new behaviour. For `H` on `M569.1` that would have turned a leftover value into a maximum phase
current.

Reusing the *slot* (rather than the letter) is legitimate and is what was done here: `FLOAT_ARRAY_PARAM('F', 3)`
now occupies the retired `h` position, so every later parameter keeps its bit. The cost is that firmware
built before the change misreads an `F` as the old 4-byte `h`, which is why **both ends must be flashed
together**.

---

## <a name="message-length"></a>Message length is a separate limit

`4` bytes of header plus `data[60]` is exactly the 64-byte CAN-FD frame, so it **cannot grow**.
`CanMessageGenericConstructor::StoreValue()` throws `"CAN message too long"` past that.

Sizes: `uint8`/`localDriver` 1 byte, `uint16` 2, `float` 4, `FLOAT_ARRAY_PARAM(n)` **1 + 4n** (a leading
element count, then the elements), string up to and including its null terminator.

A full closed-loop configuration is already about 52 of the 60 bytes, so the current-loop settings need
their own `M569.1` line. That is safe: the handler applies only the parameters present, and only `T` (or
a change of `U`) drops the driver out of closed loop.

---

## How the FOC current-loop parameter resolved all three

```cpp
FLOAT_ARRAY_PARAM('F', 3)     // {proportional gain, integral gain, max phase current in amps}
```

One **array** parameter (one bit, not three, satisfying [1](#parammap)), named **`F`** (avoiding `G` per
[2](#gm-letters)), placed **in the retired `h` slot** (preserving later bit positions per
[3](#retired-letters)).
