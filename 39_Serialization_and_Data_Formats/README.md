# Serialization & Data Formats

Serialization is how you turn an in-memory C++ object into bytes that can be written to disk, sent over a network, or handed to another process - and then read back later, possibly by a completely different program. This folder covers binary protocols, JSON, MessagePack, FlatBuffers, schema evolution, zero-copy deserialization, and cross-platform wire formats.

**Topics:** 12

## Contents

- [Design schema evolution strategies for field addition, removal, and renumbering](Design_schema_evolution_strategies_for_field_addition_removal_and_renumbering.md)
- [Design self-describing wire formats with type tags and schema fingerprints](Design_self-describing_wire_formats_with_type_tags_and_schema_fingerprints.md)
- [Design versioned binary protocols with backward and forward compatibility](Design_versioned_binary_protocols_with_backward_and_forward_compatibility.md)
- [Handle endianness and alignment in cross-platform binary formats](Handle_endianness_and_alignment_in_cross-platform_binary_formats.md)
- [Implement compile-time-checked serialization with reflection (C++26)](Implement_compile-time-checked_serialization_with_reflection_Cpp26.md)
- [Implement JSON serialization with nlohmann-json, simdjson, and glaze](Implement_JSON_serialization_with_nlohmann-json_simdjson_and_glaze.md)
- [Implement XML and HTML parsing with streaming SAX-style and DOM approaches](Implement_XML_and_HTML_parsing_with_streaming_SAX-style_and_DOM_approaches.md)
- [Implement zero-copy deserialization with FlatBuffers and Cap'n Proto](Implement_zero-copy_deserialization_with_FlatBuffers_and_Capn_Proto.md)
- [Serialize and deserialize std::variant and polymorphic types](Serialize_and_deserialize_stdstdvariant_and_polymorphic_types.md)
- [Use MessagePack and CBOR for compact binary serialization](Use_MessagePack_and_CBOR_for_compact_binary_serialization.md)
- [Use std::bit_cast and std::start_lifetime_as for safe binary parsing](Use_stdstdbit_cast_and_stdstdstart_lifetime_as_for_safe_binary_parsing.md)
- [Use std::spanstream and memory buffers for in-memory serialization](Use_stdstdspanstream_and_memory_buffers_for_in-memory_serialization.md)

## Notes

- Protocol Buffers (protobuf) provide schema-evolving binary serialization with cross-language support.
- FlatBuffers enable zero-copy access - no deserialization step, ideal for performance-critical paths.
- JSON libraries (nlohmann/json, simdjson, glaze) range from maximum convenience to raw speed.
- Cap'n Proto provides zero-copy like FlatBuffers but with an RPC framework built in.
- `std::bit_cast` (C++20) is the safe way to type-pun for binary serialization of trivial types.
- MessagePack is a compact binary alternative to JSON - smaller payloads, faster parsing.
- Schema validation should happen at system boundaries, not on every internal access.
- Endianness must be handled for binary formats - use network byte order or explicit conversion.
- Versioning and backward compatibility should be designed from the start (field numbers, optional fields).
- Reflection (C++26) will eventually enable automatic serialization without boilerplate.
