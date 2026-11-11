# Collection and conversion contracts

The following staged architecture is a reusable **design proposal** informed by experiments. Portable arbitrary-map replay was not completed by those experiments. Do not present these stages as supplied executable commands.

| Stage | Output and boundary |
| --- | --- |
| Collect | Original resources, typed scene data, defaults, dependencies and unresolved raw evidence |
| Convert base | Source-derived scene with supported geometry, materials and environmental state; explicit unsupported records |
| Add replacements | Separate derived artifact with recorded UE4/C2 substitutions and rationale |
| Add gameplay | Separately authored FFA or objective rules, spawns, navigation and state transitions |
| Package/test | Selected artifact, archive checks and runtime validation |

Users should be able to consume a collection or base directly. Collection does not implicitly install a PAK or choose a game mode. Artistic tuning remains distinct from automatic conversion or recorded replacement rules.

## Bundle requirements

- Use package/group/object identities, not short names or exported filenames, as keys. Keep identities separate from filesystem-safe filenames.
- Record schema, source/tool/profile hashes, outputs, coverage and completion state. All consumer file references should be relative to the bundle root; omit personal absolute paths from shared provenance.
- Retain original bytes and typed evidence, including unsupported payloads. Raw collection does not mean semantic understanding.
- Include streamed levels, component-local transforms, class defaults, material dependencies, collision policies and relevant environmental state links.
- Discover visual mesh components across owner classes, including gameplay actors. Do not discard scenery merely because its owner implements an objective. Do not blindly place nested particle payloads as ordinary scenery.
- When conversion discovers missing inputs, emit a typed request with source identity, field, attempted rule and required collector capability. Recollect into a new revision; do not silently read the installed source from a supposedly portable stage.
- Prove independence with an offline replay that denies access to the original installation. Valid hashes alone do not establish independence.

## Starting state

Preserve saved actor/component values and resolved source defaults. Missing inheritance evidence is unresolved; do not substitute UE4 defaults or guessed zero/false values.

Closed gates and intact barriers are legitimate source-state output. Keep alternate states, keys, bindings and attachments for later editing. Optional opening requires an explicit profile, verified source poses and clearance checks. Disabling collision while leaving a visible closed gate is not a faithful opening.

Use source scale 1.0 in original records. Any target-player scale adaptation belongs in an explicit profile affecting the full scene, not selected mesh positions only.

## Fidelity accounting

Track each identity by facet: geometry, attributes, material assignment, shader channels, collision, lighting, audio/effects and state. Use converted, approximated, unsupported, excluded or unverified dispositions with evidence.

An object can have correct geometry and unsupported motion. A complete actor inventory does not establish shader fidelity. Aggregate nested limitations even when a top-level queue is empty. Separate source coverage, available conversion inputs, target fidelity and gameplay acceptance; avoid a single misleading completeness percentage.

Choose conversion methods from explicit prerequisites. Reject ambiguous correspondence instead of trying transformations until something looks plausible. Invalidate cached decisions when sources, tools or profiles change. Validate saved output independently of the authoring code.
