# TODO

- [ ] choice of `_` as path component separation character, replace with?
  - conflicts with special symbols
- [ ] nomenclature, is this document defining an _encoding_, _mapping_, _representation_?
  - _encoding_
    - YANG RFC talks about XML _encoding_
    - same for JSON RFC: "JSON Encoding of Data Modeled with YANG"
  - XML and JSON encodings are complete and lossless
    - tag-centric metrics
- [ ] can we use `:` as namespace prefix separation character or do we need like `__`?
- [ ] _time series_ vs _metric_ nomenclature, while the final data is a time series, the exported data is a snapshot so a metric?
- [ ] namespaces
  - we have to be 100% deterministic
  - use short prefixes in path and put xmlns map as tags?
  - or can we put all namespace stuff as tags?
- [ ] what are actual name length limits of some common TSDBs and what will we reasonably reach?
  - check common IETF, OpenConfig and vendor proprietary models from IOS XR, JUNOS & Nokia - what's the longest path name we end up with?
- [ ] _tag_ vs _label_ in nomenclature
  - is one better than the other? which one is more popular? gives better intuition?
