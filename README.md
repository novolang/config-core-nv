# config-core-nv

A program's settings usually come from more than one place: values compiled
into the program, a file on disk, the process environment, and the command
line. **Layered configuration** is the practice of treating each of those as a
named source with a precedence, and merging them into one tree that a program
reads. The model is Rust's
[figment](https://docs.rs/figment), where "a provider is a named source of
values" and every provider is merged into one common value type.

This package is the arithmetic of that model. It merges layers, walks dotted
paths, and answers which layer set a value. It opens no file, reads no
environment and has no clock. The package that does those things is
[config-nv](https://novo-lang.org/packages/config-nv), which depends on this
one.

## What it is

A **configuration value** is one node of a tree. `ConfigValue` has seven arms:
a table of key-and-value pairs, a list of values, a string, an integer, a
float, a boolean, and null. Every format a program reads is adapted into this
one tree, because a merge has to compare two values that came from different
formats before it can say which one wins.

A **layer** is three things: a name, a rank, and a tree. The name is what a
provenance answer reports and what a refusal blames, so it should read as the
place a person would go to change the setting: `/etc/app.toml`, `APP_
environment`, `--set`. The rank is the precedence. A higher rank wins.

A **merge** folds the layers into one tree under five rules, listed below.
Alongside the merged tree it records, for each path that holds a value, the
name of the layer that set it. That record is the **provenance**, and
`cfgmerge.origin_of` is how a caller reads it.

A **dotted path** names a place in the tree: `server.port` is the key `port`
inside the table `server`. A segment that is the decimal spelling of an index
indexes a list.
`\.` inside a segment is a literal dot and `\\` is a literal backslash. There
are no quotes, so `log.my\.app.level` is how a key containing a dot is spelled.

**Null is a value, and absence is not.** A key no layer mentioned has no node
in the tree at all. A key a layer set to null has a `CfgNull` node. A program
clearing an inherited setting needs the difference, and
`cfglookup.presence` is where a caller asks for it.

## Install

```
novo pkg add config-core-nv
```

## Example

```novo
use cfgvalue
use cfglayer
use cfgmerge
use cfglookup
use cfgenvkeys
use cfgfault

fn main() [io]
    // The values the program ships with. Rank 0 is the lowest precedence.
    let defaults = cfglayer.layer("defaults", 0, cfgvalue.table([
        cfgvalue.pair("server", cfgvalue.table([
            cfgvalue.pair("port", cfgvalue.int_value(80))]))]))

    // The environment, as pairs the host looked up. APP_SERVER__PORT is server.port.
    let from_env = cfgenvkeys.layer_from(cfgenvkeys.rule("APP_"),
                                         [("APP_SERVER__PORT", "8080")],
                                         "APP_ environment", 30)

    // Fold the layers into one tree. The higher rank wins every path it sets.
    let m = cfgmerge.merge_all([defaults, from_env])

    // Read the port. The getter refuses a value of the wrong kind rather than coercing it.
    match cfglookup.get_int(m, "server.port")
        Ok(n)  => println("${n}")
        Err(e) => println(cfgfault.fault_kind(e))

    // Which layer set it. This is the question a surprising value leads to.
    match cfgmerge.origin_of(m, "server.port")
        Some(o) => println("set by ${o.layer}")
        None    => println("nothing set it")
```

The program prints `8080` and `set by APP_ environment`.

## What the package contains

| Module | Contents |
| --- | --- |
| `cfgvalue` | The value tree that every format is adapted into, its constructors and accessors, and the one rule that turns text into a typed value. |
| `cfglayer` | A layer as a name, a rank and a tree, with the order the merge will apply them exposed as a function. |
| `cfgmerge` | The five merge rules, the merged tree, and the record of which layer set each path. |
| `cfglookup` | The dotted-path grammar, the typed getters, the three-answer presence question, and building a tree from flat pairs. |
| `cfgenvkeys` | The correspondence between environment-variable names and dotted paths, in both directions, as a function over pairs the caller supplies. |
| `cfgfault` | The refusal type. Every arm carries the path, and every arm that had a value to look at carries the layer that supplied it. |

## How to choose an entry point

**A program assembling its settings starts at `cfglayer` and `cfgmerge`.**
Build one layer per source, hand the list to `merge_all`, and keep the result
for the life of the program.

**A program reading a value starts at `cfglookup`.** `get_int`, `get_str`,
`get_bool`, `get_float`, `get_list`, `get_str_list` and `get_table` each answer
a value or a refusal that names the layer. `presence` is the question to ask
before applying a default.

**A program with two trees and no layers calls `cfgmerge.merge_values`.** It is
the five rules on their own, with no provenance recorded. `merge_all` is this
folded over a sorted list of layers.

**A program explaining itself calls `cfgmerge.origin_of` and `history_of`.**
`origin_of` is a lookup in the record the merge already kept. `history_of`
re-walks the layers to list everyone who set a path, lowest rank first, so it
costs a walk and is called only when somebody asks.

## The rules a user needs

1. **Layers apply in ascending rank, and ties keep list order.** A later layer
   wins. `cfglayer.in_order` returns the exact order `merge_all` will use.
2. **Two tables merge deep, key by key.** A key only the lower layer has
   survives the merge.
3. **Two lists do not merge. The later one replaces the earlier one whole.**
   There is no append, no union and no index-wise merge. This is the rule
   figment, Rust's `config` and Go's Viper all settle on. A program that wants
   to accumulate puts each contribution at its own key and concatenates them
   itself.
4. **Anything else replaces.** A scalar over a table, a table over a scalar and
   a scalar over a scalar all leave the later value standing alone.
5. **Null replaces and does not delete.** A layer that sets a key to null
   leaves null in the merged tree at that path, and
   `cfglookup.presence` reports `KeyWasNull`. The key does not disappear.
6. **`presence` has three answers, not two.** `KeyAbsent` means no layer
   mentioned the path, and a default is right. `KeyWasNull` means a layer
   cleared it on purpose, and a default would undo that. `KeyHeld` means there
   is a value, of a named kind, from a named layer.
7. **The typed getters do not coerce.** `get_int` on a string is
   `CfgWrongType`, naming the layer that supplied the string. The one exception
   is `get_float`, which accepts an integer and widens it.
8. **`get_int` refuses a float, even one whose fraction is zero.** A layer that
   wrote `8080.0` where a port was wanted is a mistake worth hearing about.
9. **Text becomes a typed value exactly once, at `cfgvalue.infer_scalar`, and
   only where a layer is built from text.** `"true"` and `"false"` become
   booleans, a decimal integer that fits 64 bits becomes an integer, a decimal
   with a point or an exponent that fits a 64-bit float becomes a float,
   `"null"` becomes null, and everything else stays a string unchanged,
   `"True"` and `"99999999999999999999"` included.
10. **A numeric path segment indexes a list, and reads a table's key.**
    `server.hosts.0` is the first host when `hosts` is a list. Against a table,
    `0` is the key spelled `"0"`, because a TOML table may have one. Only the
    plain decimal spelling indexes: `server.hosts.00` is missing.
11. **Lookup is case-sensitive.** The one place anything is lowercased is the
    environment mapping in `cfgenvkeys`, where it is part of the documented
    correspondence.
12. **A layer whose root is not a table contributes nothing and is reported.**
    `merge_all` does not raise. `cfgmerge.refused_layers` lists what it would
    not take, so one bad adapter does not cost a program its other sources.
    `cfgenvkeys.skipped_names` does the same for environment names the rule
    does not map, with the reason for each.
13. **Nothing here reads the environment.** `cfgenvkeys` is a function over a
    list of `(name, value)` pairs the caller supplies. The default rule maps
    `APP_SERVER__PORT` to `server.port`: the prefix is `APP_`, the separator is
    `__`, segments are lowercased, values are typed, and values are not split
    into lists.
14. **`cfgenvkeys.name_for` is the direction a host needs.** The standard
    library reads one environment variable by name and cannot enumerate the
    environment, so a program turns each path it understands into a name and
    looks that up. `map_name` is the inverse.
15. **Duplicate keys in a table are the caller's to avoid.** `cfgvalue.table`
    does not deduplicate, because which of two same-named pairs a format means
    is the format's rule. A lookup finds the first.

## Running on a microcontroller

A firmware image carries a settings table built at compile time: a radio
channel, a sample interval, calibration constants. An update received over the
air is a second layer on top of it. Reading a setting is the same question a
server asks of a file.

novo-lang lets a package state which of its modules can run on a device, and
the claim is checked by compiling a probe program for a Cortex-M4. Here the
claim covers `cfgvalue`, `cfglayer`, `cfgmerge`, `cfglookup` and `cfgfault`.
Their functions are `@tier(rt)`: the tree is made of lists and strings, so they
need the device's heap, and they take nothing else from the host.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe merges a built-in layer with an update, reads two integers through
the typed getters, checks that a cleared key reads as null, checks that a
refusal names its layer, and prints `PASS: config-core-embedded` under QEMU's
`mps2-an386` machine.

`cfgenvkeys` is outside the claim, because a device has no environment to
read.

## What is not included

- **Reading anything.** No file is opened, no environment variable is read and
  no clock is consulted. Every function is arithmetic over values the caller
  already holds. [config-nv](https://novo-lang.org/packages/config-nv) is the
  package that reads.
- **The format parsers.** TOML, YAML and JSON are parsed elsewhere, and the
  functions that turn their output into a `ConfigValue` live in config-nv. A
  program that only wants to merge two tables it built in code resolves no
  parser to do it.
- **Dates and times.** TOML has both, and this tree holds them as strings in
  RFC 3339 spelling. A configuration merge never compares two instants, and a
  program that wants the date parses the string it gets back.
- **Deserialising into a struct.** That belongs with whichever package owns the
  deserializer, not with the merge.
- **Watching a file for changes.** [watch-nv](https://novo-lang.org/packages/watch-nv)
  is the package for that.
- **Interpolation.** A value containing `${OTHER_KEY}` is left alone. Resolving
  one needs an order across layers, and getting that order wrong silently is
  worse than not having the feature.
- **Case-insensitive keys.** Viper folds key case, which surprises anyone whose
  keys differ only by case. Lookup here is case-sensitive.
- **A table key that is not a string.** YAML permits a collection as a mapping
  key. A dotted path cannot name one, so config-nv's YAML adapter refuses such
  a document rather than flattening it.

## Related packages

- [config-nv](https://novo-lang.org/packages/config-nv) is the half that reads:
  it opens a file, dispatches on the extension, looks environment variables up,
  parses a `.env`, and builds the layers this package merges.
- [toml-edit-nv](https://novo-lang.org/packages/toml-edit-nv) edits a TOML file
  in place and keeps everything it did not change. This package only reads
  values that are already parsed.
- [watch-nv](https://novo-lang.org/packages/watch-nv) reports that a file
  changed, which is what a program reloading its configuration needs.
- `std.env` in the standard library reads one environment variable by name. It
  cannot enumerate the environment, which is why `cfgenvkeys.name_for` exists.
- `std.toml` and `std.json` in the standard library parse those two formats.
  Their output is adapted into this package's tree by config-nv.

## Tests

```bash
novo test tests/merge_tests.nv     # the value tree, the layers, the five rules
novo test tests/laws_tests.nv      # the same rules as properties over random trees
novo test tests/lookup_tests.nv    # the path grammar and the typed getters
novo test tests/envkeys_tests.nv   # the environment mapping, both directions
novo test tests/edges_tests.nv     # every refusal, rendering and grammar edge
bash tests/coverage.sh             # line coverage over src/, merged across suites
```

There is no published test data for a configuration merge, so the suites
assert the rules this page lists. `merge_tests.nv` writes each merge rule down
as one case. `laws_tests.nv` checks the rules over a few hundred layers of
random tables, lists, scalars and nulls from a fixed pseudo-random sequence:
that the merged tree is the rules folded over the layers in order, that every
leaf comes from the last layer that set its path, that a lower table's leaf
survives unless the upper table holds its path or a prefix of it, that a list
replaces whole, and that the empty table changes nothing. `lookup_tests.nv`
asserts the shape of each refusal: that a wrong type is `CfgWrongType` and not
`CfgMissing`, that a malformed path never reaches the tree, and that an explicit
null is its own answer. `envkeys_tests.nv` asserts the mapping with nothing
exported, which is possible because the mapping is a function over pairs.

The behaviour the suites compare against is figment's and Rust's `config`'s
for the layering and the getters, and dynaconf's for a settings object that can
say where a value came from.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
