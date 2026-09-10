# Changelog

All notable changes to config-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
