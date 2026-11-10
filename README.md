# senml-nv

SenML, the Sensor Measurement Lists format of
[RFC 8428](https://www.rfc-editor.org/rfc/rfc8428), is how a small device
reports what it measured. A reading is a record with a name, a unit, a value
and a time. A pack is an array of records, with base fields up front so that a
hundred readings from one sensor do not repeat its name a hundred times. It is
written as JSON for a person or a web service and as CBOR for a radio link.
This package brings SenML to novo-lang, in two halves: a pack of fixed capacity
that a microcontroller builds, writes and reads with no heap, and a list of
records that a gateway or a server reads, resolves and writes.

## What it is

A sensor node holds a `SenmlPack`, a `@value` of about a kilobyte. It adds
readings to it, and writes it as JSON or CBOR into a buffer it owns. A node
that receives a pack reads the CBOR into a second pack and asks for the
resolved name, unit, time and value of each record.

A gateway holds a list of `SenmlRecord`. It reads a pack from JSON or CBOR,
resolves it by the algorithm of RFC 8428 section 4.6, and writes it back out.

**The device half allocates nothing.** The pack is a pool of 256 bytes for
the strings and room for 8 records. A reading that does not fit is refused and
the pack is marked, not cut. Resolution does not join strings: a name is the
base name and the name, read as one run of bytes through `rec_name_byte`.

**The two halves agree on the bytes.** `tests/senmlpack_tests.nv` writes the
voltage and current example of RFC 8428 section 5.1.2 and compares the CBOR
with the RFC's own dump of section 6, byte for byte, and
`tests/senmllist_tests.nv` reads that dump back into a list.

**Numbers are `Float`.** CBOR carries a number exactly, as the shortest of an
integer, a half, a single and a double that gives the same value. JSON text
carries up to six decimal places.

## Install

```
novo pkg add senml-nv
```

## Example

A sensor node's report. The names and the units are string literals, so the
pack holds copies in its own pool, and `to_json` and `to_cbor` write into a
buffer the caller owns. On a device the buffer is what goes to the radio; here
the program prints it.

```novo
use std.str
use senmlpack

fn main() [io]
    var p = senmlpack.pack()
    // Every name below is joined to this base name.
    let _ = senmlpack.set_base_name(var p, "urn:dev:ow:10e2073a01080063:")
    let t = senmlpack.add(var p, "temp")
    let _ = senmlpack.set_unit(var p, t, "Cel")
    senmlpack.set_value(var p, t, 23.1)
    let h = senmlpack.add(var p, "humidity")
    let _ = senmlpack.set_unit(var p, h, "%RH")
    senmlpack.set_value(var p, h, 48.5)

    var out: senmlpack.SenmlBuf = Vec.new()
    if senmlpack.to_json(var p, var out)
        var text = ""
        for i in 0..out.len()
            text = text + str.from_byte(out[i] as Int)
        println(text)
    if senmlpack.to_cbor(var p, var out)
        println("${out.len()} bytes of CBOR")
    println("valid: ${senmlpack.validate(p) == senmlpack.SENML_OK}")
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `senmlunit` | The three vocabularies RFC 8428 restricts: the 58 symbols of the SenML Units registry of section 12.1, the characters a name may have, and the line at 2^28 seconds between a time relative to now and an absolute one. |
| `senmlpack` | The fixed-capacity pack: a builder, the resolved name, unit, time, value and sum of a record, validation, and writing as JSON and as CBOR. |
| `senmlread` | Reading a pack from CBOR into a `SenmlPack`, with no allocation: integers, floats of three widths and decimal fractions, definite and indefinite lengths, and a code for each way a document can be wrong. |
| `senmllist` | The host half: a list of records with strings and floats, reading and writing JSON through the standard library and CBOR through `cbor-nv`, resolution by section 4.6 with the checks of sections 4.4 and 4.5, base64url for the data value, and the error type. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/senml-nv) is that
reference.

## The rules a user needs

1. **One set of base fields in the fixed pack.** The pack has a base name, a
   base time, a base unit, a base value, a base sum and a version, and they
   are written with the first record and apply to all of them. RFC 8428 lets
   any record state a base field again; `senmlread` refuses a document that
   does (`SENML_READ_LATE_BASE`), and `senmllist` reads and resolves it.
2. **Resolution is by section 4.6.** The name is the base name and the name,
   the unit the record's or the base unit, the time the base time plus the
   record's, and the value and the sum the base added to the record's. A
   missing time counts as zero. A time below 2^28 seconds, 268435456, is
   relative to now, and one at or above it is seconds since the Unix epoch.
3. **A record has exactly one value, or a sum.** `validate` and `resolve`
   refuse a record with more than one of `v`, `vs`, `vb` and `vd`, and one
   with none and no sum. They refuse a version above 10, and, in the list
   form, a version that differs from the first record's.
4. **A name has a restricted alphabet.** A to Z, a to z, 0 to 9, and `-`,
   `:`, `.`, `/` and `_`, starting with a letter or a digit. The restriction
   is applied to the base name and the name together.
5. **The field order written is the RFC's.** Base name, base time, base unit,
   base value, base sum and version, then name, unit, time, update time,
   value, string value, boolean value and sum. A reader accepts any order.
6. **JSON text carries six decimal places.** From 4.5 billion up, three. A
   magnitude of 10^12 or more and NaN are refused when written. CBOR has no
   such limit.
7. **CBOR floats are the shortest that give the same value.** A whole number
   below 2^53 is an integer. Otherwise a half if one holds the value, then a
   single, then a double. The fixed pack reads a decimal fraction (tag 4) and
   writes none; `senmllist` does the same.
8. **A label this package does not know is skipped.** A text label that ends
   in an underscore is an error, because section 4.4 makes it one the reader
   must understand.
9. **A document that does not fit is refused, with a code.** `senmlread`
   answers 8 records, 256 bytes of strings and 15 levels of unknown nesting;
   the buffer that `to_json` and `to_cbor` write into is 512 bytes. A pack
   that failed to build has `p.ok` false and writes nothing.
10. **The data value is on the host.** `vd` is base64url in JSON and a byte
    string in CBOR. `senmllist` carries it as `Bytes`; the fixed pack does
    not, and `senmlread` refuses a document that has one.

## Running on a microcontroller

`senmlunit`, `senmlpack` and `senmlread` are `@tier(embedded)` throughout and
allocate nothing, and the package declares no effects. `senmllist` is the one
module that does not build for a device: it allocates and it uses the
standard library's JSON, so the manifest names it in `host_modules`.

`tests/embedded_probe.nv` is a firmware program for a sensor node. It builds
the RFC's voltage and current pack, writes it as CBOR and checks its length and
a checksum of its 195 bytes, reads it back into a second pack and checks the
resolved name, unit, time and value of two records, writes it as JSON, and
checks the registry and the name rule. It is compiled for a Cortex-M as part
of this package's checks, so the claim is built rather than asserted, and under
QEMU it prints `PASS: senml-embedded`.

The image is 35,180 bytes of flash and 520 bytes of static RAM, including the
runtime and the board's startup code. Of the flash, 17,572 bytes are the
program, 5,654 the runtime and 7,994 the software floating point that the
double arithmetic needs on a part with a single-precision unit. A
`SenmlPack` is 1,056 bytes and a `SenmlBuf` 520, both in the caller's frame.
The probe holds two packs and two buffers.

The device effects are the caller's. The package reads no register and no
clock, and the time of a reading is a number the caller puts in.

## What is not included

- **Reading JSON on a device.** Writing JSON is here; parsing it takes a
  tokenizer, and `senmllist` does it on a host through the standard library.
  A device that receives SenML receives CBOR.
- **The data value on a device.** See rule 10.
- **XML and EXI.** RFC 8428 sections 7 and 8 define them; this package does
  JSON and CBOR.
- **SenSML streams.** `senmlread` reads an indefinite-length array and
  indefinite-length maps; nothing writes one, and a pack is written with
  definite lengths as section 6 asks for a pack that is not a stream.
- **Chronological order.** Section 4.6 asks for resolved records in time
  order; `resolve` keeps the order it was given.
- **Unit conversion.** `unit_is_registered` says whether a symbol is in the
  registry. Scaling `mV` to `V` is the caller's.
- **The registry of extensions.** Labels beyond the fifteen of Table 1 are
  skipped, not interpreted.
- **A clock.** Whether a time is "now" is the application's.

## Related packages

- [cbor-nv](https://novo-lang.org/packages/cbor-nv) is the CBOR of the host
  half: the value tree `senmllist` encodes and decodes. The fixed pack writes
  and reads the few CBOR heads a SenML record needs itself, because the tree is
  a heap value a device does not build.

## Tests

The suite is five files under `tests/`. What they assert comes from RFC 8428
rather than from an implementation.

`tests/senmlpack_tests.nv` compares what the pack writes with the RFC: the
single data point of section 5.1.1 as JSON, the voltage and current example as
JSON, and the same example as CBOR against the 195-byte dump of section 6. The
shortest-float rule is held to vectors worked out by hand for each width, and
the buffer limits and the resolution accessors are checked at their edges.

`tests/senmlread_tests.nv` reads the RFC's dump and writes it back to the same
bytes, and holds every label, every number form and every refusal to
hand-worked vectors.

`tests/senmlunit_tests.nv` lists all 58 units of the registry and the edges of
the name rule and the time line.

`tests/senmllist_tests.nv` reads the RFC's JSON and CBOR examples into lists,
resolves them, round-trips every field through both forms, and checks the
base64url alphabet against RFC 4648's own vectors.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.
