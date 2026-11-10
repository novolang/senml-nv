# Changelog

All notable changes to senml-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`senmlunit`**, the vocabularies RFC 8428 restricts, with no allocation:
  the 58 symbols of the SenML Units registry (`unit_is_registered`), the
  characters a name may have (`name_is_valid`, `name_start_ok`,
  `name_char_ok`) and the line at 2^28 seconds between a relative and an
  absolute time (`time_is_relative`).
- **`senmlpack`**, a pack of fixed capacity, 8 records and 256 bytes of
  strings, that a microcontroller builds with no heap: a builder, the
  resolved name, unit, time, value and sum of each record, `validate`, and
  writing as JSON (`to_json`) and as CBOR (`to_cbor`) into a 512-byte buffer.
  The CBOR writer takes the shortest of an integer, a half, a single and a
  double that gives the value exactly, and its output for the voltage and
  current example of RFC 8428 section 5.1.2 is the RFC's own 195-byte dump.
- **`senmlread`**, reading a pack from CBOR into a `SenmlPack` with no
  allocation: integers, floats of three widths and decimal fractions,
  definite and indefinite arrays and maps, labels it does not know skipped,
  and a code for each way a document can be wrong.
- **`senmllist`**, the host half, named in `host_modules`: a list of
  `SenmlRecord`, reading and writing JSON through the standard library and
  CBOR through `cbor-nv`, `resolve` for section 4.6 with the checks of
  sections 4.4 and 4.5, base64url for the data value, and `SenmlError`.
- **`tests/embedded_probe.nv`**, a firmware program for a sensor node, built
  for `--target=nrf52-qemu`. It prints `PASS: senml-embedded` under QEMU. The
  image is 35,180 bytes of flash and 520 bytes of static RAM.
- Five test suites with vectors from the RFC and worked out by hand.

### Not included

- Reading JSON on a device. A device that receives SenML receives CBOR.
- The data value (`vd`) in the fixed pack. `senmllist` carries it.
- A base field stated again after the first record, in the fixed pack.
  `senmlread` refuses such a document with `SENML_READ_LATE_BASE`.
- XML and EXI, SenSML streams as an output, sorting into chronological order,
  and unit conversion.
