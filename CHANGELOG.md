# Changelog

All notable changes to config-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `cfgvalue` — one configuration value tree, `ConfigValue`, that every
  format's output is adapted into, and `infer_scalar`, the package's
  single coercion from text to a typed value.
- `cfgfault` — `ConfigFault`, whose every arm carries the path and,
  where there was one, **the layer to blame**; and `KeyPresence`, which
  separates "no layer mentioned it" from "a layer set it to null".
- `cfglayer` — a layer as a name, a rank and a tree, with the order the
  merge will apply exposed rather than hidden.
- `cfgmerge` — the five merge rules, the merged tree, and `origin_of`:
  for any key, which layer set it. `history_of` re-walks the layers to
  say who else tried.
- `cfglookup` — the dotted path grammar with one escape, the typed
  getters that do not coerce, and `with_path` for building a tree from
  flat pairs.
- `cfgenvkeys` — the environment key mapping in **both** directions, as
  a pure function over `(name, value)` pairs the host supplies.
  `name_for` is the direction a host on this toolchain actually needs,
  because the standard library reads the environment by name and cannot
  enumerate it.

### Known

- `tests/embedded_probe.nv` makes a device claim and the audit's
  `core-embedded` row is green — it links for `--target=nrf52-qemu`.
  What the probe does NOT do is carry a `Result` across a function
  boundary: `Result<T, E>` does not build at `@tier(embedded)` today
  while the `Error` trait is absent from that prelude, and every getter
  here returns one on purpose, so the probe folds each answer to a
  `Bool` instead. The filing is
  `result-is-unusable-at-tier-embedded-no-error-trait`.

### Design notes

- One common value tree rather than an adapter per format. A merge has
  to compare two values that came from different formats before "later
  layer wins" has an answer, and three trees make that comparison a
  matrix.
- The format adapters live in config-nv rather than here. That package
  depends on the parsers anyway. Putting them here would give every
  consumer that only wanted to merge two tables a closure containing
  toml-nv, yaml-nv and calendar-nv.
- `std.json`'s value is an opaque handle whose accessors return an
  optional `Any`, so its adapter has to discover an arm by trying the
  table accessor, then the list accessor, then the scalar casts, in
  that order. That is written down in config-nv's `cfgadapt`.
- The type names carry a prefix because bare names collide across a
  registry and enum variants collide across an assembly: `Error` is a
  standard-library trait, `Config`, `Provenance` and `Source` are
  already taken, and `str` is a standard-library module.
- Interpolation is a candidate for 0.2, with the resolution order
  across layers written down before any code.
