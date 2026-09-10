# config-core-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The arithmetic of a layered configuration: one value tree, layers with
a precedence, a merge whose rules are written down rather than
discovered, dotted-path lookup whose refusals say **which layer** they
are about, and a provenance answer for any key — who set this, and who
else tried.

There is no file here, no environment and no clock. Everything is a
function of values the caller already holds. The half that opens
`/etc/app.toml`, reads `APP_PORT` out of the process environment and
parses a `.env` is [`config-nv`](https://github.com/novolang/config-nv),
which depends on this one.

```
novo pkg add config-core-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use cfglayer
use cfgvalue
use cfgmerge
use cfglookup

fn port(from_file: ConfigValue, from_env: ConfigValue) -> Result<Int, ConfigFault>
    let m = cfgmerge.merge_all([
        cfglayer.layer("/etc/app.toml", 10, from_file),
        cfglayer.layer("APP_ environment", 30, from_env)
    ])
    cfglookup.get_int(m, "server.port")
```

And the call the whole package exists for, when that comes back with
the wrong number:

```novo ignore
match cfgmerge.origin_of(m, "server.port")
    Some(o) => println("set by ${o.layer}")
    None    => println("nothing set it")
```

## The load-bearing interface

```novo ignore
pub fn merge_all(ls: [ConfigLayer]) -> MergedConfig
pub fn origin_of(m: MergedConfig, path: Str) -> ?KeyOrigin
```

Not a type — a pair, and the second one is why the first one keeps a
record. **Five rules, applied in this order:**

1. Layers apply in ascending rank; ties keep list order. A later layer
   wins.
2. Two **tables** merge deep, key by key. A key only the lower layer has
   survives.
3. Two **lists** do not merge: the later one **replaces** the earlier
   one whole.
4. Anything else replaces — a scalar over a table, a table over a
   scalar, a scalar over a scalar.
5. `CfgNull` is a value and follows rule 4. It replaces what is under
   it and it **does not delete the key**.

Rules 3 and 5 are the two most configuration libraries leave implicit.

**Why lists replace.** Every alternative needs an identity for an
element, and a configuration list has none: two `["-O2"]` flags are not
the same flag, and merging `["a"]` under `["b"]` into `["a", "b"]` means
no layer can ever *remove* an inherited entry. Replacement is what
figment, Rust's `config` and Viper all settle on, and it is the rule a
person can predict without reading the source.

**Why null does not delete.** Deleting would make the merged tree depend
on the order of two layers in a way no lookup could explain afterwards:
`origin_of` would point at a layer whose value is not there. A layer
that wants a setting gone sets it to null, and the caller sees
`KeyWasNull` — a decision the program makes with a reason in hand,
rather than one this package makes silently.

**Absent is not null.** `cfglookup.presence` has three answers, not two:

| | means | what a caller does |
| --- | --- | --- |
| `KeyAbsent` | no layer mentioned it | apply the default |
| `KeyWasNull(layer)` | a layer set it to nothing, on purpose | do *not* apply the default |
| `KeyHeld(kind, layer)` | there is a value, of this kind | use it |

## How a `TomlValue` gets in

The question the package had to answer: a program's layers come from
toml-nv, from yaml-nv and from `std.json`, and those are three
different trees. Two shapes were available — an adapter function per
format taking each package's own type, or **one common `ConfigValue`
tree with `from_toml` and friends**.

**One common tree, and the adapters live in `config-nv`, not here.**

The tree is common because a merge has to compare two values that came
from different formats: the file said `port = 8080` in TOML and the
environment said `APP_PORT=8080` as text, and "later layer wins" has no
answer unless both are the same kind of thing first. One tree makes the
merge total; three trees and a pairwise comparison make it a matrix,
and adding ini-nv later would make it a bigger one.

The adapters are in the host half for a reason that is about the
dependency closure and nothing else. `config-nv` depends on toml-nv and
yaml-nv already — it is the package that opens the file and dispatches
on the extension. If the adapters were here, **this** package would
depend on both parsers, and calendar-nv behind toml-nv (whose
`TomlValue` carries civil date-times), so a program that only wanted to
merge two tables it built in code would resolve three packages it never
calls. Three total functions are not worth that to a `core` package
whose whole argument is that it costs a caller nothing.

Two consequences, stated because they are the cost of the choice:

- A caller holding a `TomlValue` in a pure context cannot convert it
  without `config-nv`. The conversion functions in `config-nv` declare
  **no effects** — they are in a `host` package because that is where
  the format dependency is, not because they perform anything.
- `std.json`'s value is `JsonValueH`, an opaque handle whose accessors
  return `?Any`, so its adapter has to *discover* an arm by trying
  `json.keys`, then `to_list`, then the scalar casts, in that order.
  That is written down in `config-nv`'s `cfgadapt` rather than here,
  where it would be the only adapter with a caveat.

## The layer, and why

`core` — no effects, and no argument required to get there. There is
no file, no environment read and no clock; every function is arithmetic
over a tree the caller supplies. The environment mapping is here too,
as a pure function over a list of `(name, value)` pairs, which is what
lets a test check `APP_SERVER__PORT` → `server.port` without exporting
anything.

**`@tier(embedded)` is claimed, and the claim is built.** A device
reading a configuration table baked into its image wants exactly
`cfglookup` — the dotted-path walk and the typed getters are integer
and string arithmetic with no allocation beyond the tree it was handed.
`tests/embedded_probe.nv` makes the claim and links for
`--target=nrf52-qemu`.

What the probe leaves out, and why, is worth reading before adding to
it: `Result<T, E>` does not build at `@tier(embedded)` today — the
`Error` trait is not in that tier's prelude, so a package's own error
type cannot implement it, and `Str` does not implement it there either.
Every getter here returns `Result<_, ConfigFault>` on purpose, because
a refusal that cannot say which layer it is about is not worth
returning, so the probe exercises the constructors, the accessors and
the path grammar and folds each answer to a `Bool` rather than carrying
one across a function boundary. The design is not being routed around;
the probe is written to the tier the language has.

## The names, and the ones that were taken

| here | the obvious name | why not |
| --- | --- | --- |
| `ConfigValue` | `Value` | too general to be unique across a registry; `TomlValue`, `YamlValue` and `DbValue` all took the prefixed form first |
| `ConfigLayer` | `Layer` | same, and `LayerNorm` is already a standard-library struct |
| `ConfigFault` | `Error`, `ConfigError` | `Error` is a standard-library **trait**; `Fault` says "a refusal with a reason" rather than "something broke" |
| `MergedConfig` | `Config` | `Config` is a struct in the orbit tree already |
| `KeyOrigin` | `Provenance`, `Source` | `Provenance` is an orbit struct and `Source` a published enum |
| `CfgTable`, `CfgStr`, … | `Table`, `Str`, … | enum **variants** collide by bare name across the whole assembly, so every arm carries a prefix |
| `KindInt`, `KeyAbsent`, … | `Int`, `Absent` | the same rule; `None` and `Other` are standard-library variants already |
| `str_value` | `str` | `str` is a standard-library module, and a function shadowing it makes every other call in the file ambiguous to read |

## The reference implementation

Rust's **figment** for the layering model — a provider is a named source
with a precedence, and the merge is over one common value tree — and
Rust's **config** for the dotted-path getters and the environment
mapping with a prefix and a separator. Python's **dynaconf** for the
insistence that a settings object can say where a value came from.
Go's **Viper** for the negative example: its `MergeConfigMap` and its
key case-folding are both decisions this package makes differently and
writes down.

Deliberately left out, and where it went instead:

- **Reading anything.** `config-nv`.
- **The formats.** toml-nv, yaml-nv, `std.json`, and ini-nv when it
  lands; this package sees their output.
- **Deserialising into a struct.** That is serde-nv's shape, and the
  bridge belongs in whichever package owns the `Deserializer`, not in
  the merge.
- **Watching for changes.** watch-nv's row on the plan.
- **Interpolation** (`${OTHER_KEY}` inside a value). It needs a
  resolution order across layers, and getting that wrong silently is
  worse than not having it; it is a candidate for 0.2 with the
  algorithm written down first.
- **Case-insensitive keys.** Viper folds them and it surprises people
  who have a key that differs only by case. Lookup here is
  case-sensitive, and `cfgenvkeys` is the one place that lowercases,
  where it is a documented part of the mapping.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: config-core-nv.<fn>` — the
expected result until the bodies land. Run it with `--isolate` for one
verdict per test naming the function it stopped at.

| module | `pub` items | implemented |
| --- | --- | --- |
| `cfgvalue` | 3 types, 18 functions | no |
| `cfgfault` | 2 types, 6 functions | no |
| `cfglayer` | 1 type, 6 functions | no |
| `cfgmerge` | 2 types, 8 functions | no |
| `cfglookup` | — , 14 functions | no |
| `cfgenvkeys` | 1 type, 8 functions | no |
