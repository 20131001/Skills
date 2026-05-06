# Benchmarks

Place small benchmark projects or benchmark descriptions here.

Good benchmark cases isolate one or two real TON audit risks across FunC, Tolk, or Tact, such as:

- payload-derived sender authorization
- missing exact-layout proof on a fixed-layout body or storage parser
- optimistic accounting before a fallible downstream send
- counterfeit Jetton wallet deposit spoofing
- external-message replay or gas-drain paths
- Tact fallback receiver accepting unsupported value-bearing messages
- Tolk lazy parser or typed storage decoder failing after state mutation
