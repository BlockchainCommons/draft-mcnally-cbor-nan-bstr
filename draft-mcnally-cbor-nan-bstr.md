---
title: A CBOR Tag for Lossless Transport of IEEE-754 NaN Bit Patterns
abbrev: nan-bstr
docname: draft-mcnally-cbor-nan-bstr-latest
category: std
ipr: trust200902
area: ART
workgroup: CBOR
keyword: [CBOR, IEEE-754, NaN, determinism, interoperability]
stand_alone: yes
pi: [toc, sortrefs, symrefs]
venue:
  group: CBOR
  mail: cbor@ietf.org
  github: BlockchainCommons/draft-mcnally-cbor-nan-bstr
author:
  - name: Wolf McNally
    org: Blockchain Commons
    email: wolf@wolfmcnally.com
    role: editor
normative:
  RFC8610:
  RFC8949:
informative:
  IEEE754:
    title: IEEE Standard for Floating-Point Arithmetic
    seriesinfo:
      IEEE: Std 754-2019
    target: https://ieeexplore.ieee.org/document/8766229
  IANA-CBOR-TAGS:
    title: Concise Binary Object Representation (CBOR) Tags Registry
    target: https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml
  I-D.mcnally-deterministic-cbor:
  WASM-NaN:
    title: WebAssembly issue #1463 - What is the motivation for NaN canonicalization?
    target: https://github.com/WebAssembly/design/issues/1463
  JSC-NaNBox:
    title: JavaScriptCore NaN Boxing exercise
    target: https://browser.training.ret2.systems/content/module_1/8_jsc_internals/exercises/jsc_nan_boxing
  CBOR-WG-NaN-Threads:
    title: CBOR WG mailing list threads on NaN and determinism
    target: https://mailarchive.ietf.org/arch/browse/cbor/?q=NaN
date: 2025-10-24
---

--- abstract

IEEE-754 NaN formations are not numbers and have no equivalence class. They are not comparable to anything, including reflexively, and therefore each attribute - sign bit, signaling/quiet bit, payload bits, and representation width (binary16, binary32, binary64) - can carry meaning for an implementation. CBOR permits encoding NaNs as floating-point values, but generic processing, preferred serialization, and deterministic profiles may canonicalize or otherwise alter NaN encodings, losing information. This document defines a CBOR tag, colloquially "nan-bstr", whose content is a byte string carrying the exact IEEE-754 NaN bit pattern so that all attributes are preserved across encode/decode, enabling use cases like NaN boxing, platform-specific error signaling, diagnostics, and forensics.

--- middle

# Introduction

{::boilerplate bcp14-tagged}

CBOR defines an extensible binary format with semantic tags (major type 6) that can ascribe additional meaning to enclosed data items ({{RFC8949}}). While CBOR's floating-point types can encode NaN values, encoders and application profiles commonly canonicalize NaNs or collapse them to a single preferred representation. For applications that rely on specific NaN formations, this behavior is unacceptable. This specification defines a single CBOR tag that wraps a CBOR byte string (bstr) containing 2, 4, or 8 octets representing the big-endian bit pattern of a single IEEE-754 NaN (binary16/32/64). The tag enables exact round-tripping of NaN attributes independent of the policies that an ecosystem applies to floating-point numbers.

# Motivation and Rationale

IEEE-754 purposefully leaves NaNs incomparable and allows implementations to use sign, signaling/quiet, and payload bits for implementation-defined purposes ({{IEEE754}}). When CBOR encoders perform preferred serialization or when deterministic profiles constrain encodings for predictability (see {{RFC8949, Section 4.2}}), the precise NaN formation can be lost. A narrowly scoped tag that treats a NaN as an opaque bit pattern avoids interfering with numeric semantics while providing an explicit, interoperable mechanism to preserve all attributes. This mechanism addresses concrete needs: (1) NaN boxing schemes use NaN payload space to embed tagged values and pointers; preserving those bits across transports is essential ({{JSC-NaNBox}}). (2) Some profiles, such as dCBOR, intentionally allow only a single canonical NaN and reject others to maximize determinism; those ecosystems still benefit from an explicit escape hatch when a precise NaN must be preserved ({{I-D.mcnally-deterministic-cbor}}). (3) Other communities canonicalize NaNs for determinism (for example, WebAssembly discussions), illustrating the tension between reproducibility and information retention ({{WASM-NaN}}). See also prior CBOR WG threads that discussed NaN handling and determinism ({{CBOR-WG-NaN-Threads}}). A tag solves the transport problem without weakening profile policies for ordinary numeric values.

# The nan-bstr Tag

## Semantics

The nan-bstr tag denotes that its content is a CBOR byte string whose bytes are, in network byte order (big-endian), the bit pattern of a single IEEE-754 NaN in one of the three interchange widths: 2 bytes (binary16), 4 bytes (binary32), or 8 bytes (binary64). The tag does not change IEEE-754 NaN semantics: it does not define equality or ordering for NaNs. A tagged NaN is not a CBOR floating-point number; it is an opaque bit pattern that an application MAY convert to a native NaN when desired, acknowledging that the conversion could lose information if the platform cannot represent signaling NaNs or payload bits.

## Validity

A decoder that understands this tag MUST enforce all of the following: (1) The enclosed bstr length is exactly 2, 4, or 8. (2) The bytes are interpreted in network byte order, consistent with CBOR's encoding of multi-byte values ({{RFC8949}}). (3) The bit pattern is a NaN for the indicated width (that is, exponent is all ones and the significand/fraction field is non-zero per {{IEEE754}}). (4) No normalization or canonicalization of the payload, sign bit, signaling/quiet bit, or width is performed by the tag processing itself. If any check fails, the decoder MUST treat the tagged value as invalid for this tag and handle it per application error policy.

## Deterministic Encoding and Preferred Serialization

CBOR's preferred serialization and deterministically encoded CBOR rules ({{RFC8949, Section 4.2}}) apply to the tag and to the bstr's container (for example, definite length), not to the content bytes themselves. The content of the bstr is application-defined and MUST be preserved exactly. When an application needs exact preservation of a NaN, the sender MUST use this tag in place of a floating-point NaN literal.

## Interoperability with Deterministic Profiles and dCBOR

Deterministic profiles often restrict or canonicalize floating-point representations to ensure byte-for-byte stability. For example, the dCBOR profile allows only a single canonical NaN formation (half-width value with CBOR representation 0xf97e00) and rejects others ({{I-D.mcnally-deterministic-cbor}}). These profiles can still carry non-canonical NaNs by allowing this tag: encoders emit nan-bstr when exact NaN preservation is required and emit the profile's canonical NaN otherwise. This keeps deterministic rules for numbers intact while providing a precise transport for exceptional cases.

# Examples and Diagnostic Notation

The requested tag number for this specification is 102. Diagnostic notation shows tags in decimal by default. (a) Half-precision quiet NaN (0x7E00): `102(h'7E00')`. (b) Single-precision quiet NaN with payload 0x000001: `102(h'7FC00001')`. (c) Double-precision signaling NaN with minimal payload 0x00000000000001 and sign bit set: `102(h'FFF0000000000001')`. CBOR encodings (hex) for the above are, respectively: (a) `D8 66 42 7E 00` (tag(102), bstr len 2, 0x7E00), (b) `D8 66 44 7F C0 00 00`, (c) `D8 66 48 FF F0 00 00 00 00 00 01`. In all cases, the content preserves sign, signaling/quiet, payload bits, and width exactly; applications that cannot natively represent a formation still retain the bit pattern for pass-through or later analysis.

# CDDL

The following CDDL ({{RFC8610}}) defines the tagged value.

```
nan-bstr = #6.102 (bstr .size (2 / 4 / 8))
```

# Security Considerations

Treat the bstr as untrusted input. A decoder MUST validate NaN well-formedness for the indicated width. Implementations MUST avoid triggering floating-point exceptions when merely transporting the bit pattern. Because payload bits can carry platform-specific metadata, applications should consider confidentiality and integrity requirements when transporting tagged NaNs. The tag restricts size to 2/4/8 bytes and introduces no variable-length resource concerns beyond those of CBOR bstr handling ({{RFC8949}}).

# IANA Considerations {#iana}

IANA is requested to register one new entry in the CBOR Tags registry ({{IANA-CBOR-TAGS}}) using the template of {{RFC8949, Section 9.2}}.

* Tag: 102 (Specification Required range).
* Data item: byte string.
* Semantics: IEEE-754 NaN encoded as a byte string (nan-bstr). Preserves sign, signaling/quiet bit, payload bits, and width (2/4/8 bytes in network byte order).
* Reference: This document.

# Implementation Notes

Encoders SHOULD use definite-length bstrs. Decoders SHOULD expose APIs that surface width, sign, signaling/quiet, and payload without mutating bits. Implementations SHOULD avoid using nan-bstr values as map keys because NaN equivalence is undefined; generic CBOR guidance on key equivalence applies ({{RFC8949}}). Profiles that otherwise canonicalize floating-point NaNs can retain those rules and treat nan-bstr as the explicit mechanism for exact preservation when needed.

# Acknowledgments

Thanks to CBOR WG participants for discussion of determinism and floating-point edge cases, and to implementers who documented NaN canonicalization behavior across platforms.

--- back
