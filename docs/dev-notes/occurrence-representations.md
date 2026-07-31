<!-- This file was generated with the assistance of an AI coding tool. -->

# Occurrence representations — normalize occurrence-local reps onto the type

> **Living dev note** for the `occurrence-representations` branch/PR. Read before working
> on the feature; append decisions and findings as the PR is refined. This is *not* user
> documentation — at merge it is removed or its durable parts promoted to code comments.
> See [README.md](README.md) for the convention.

Tracking issue: [#8788](https://github.com/IfcOpenShell/IfcOpenShell/issues/8788). Supersedes
the closed PR #8789 (see "History / pivot"). Stacked on `dev-notes-system` (#8201) →
`opening-template-on-type` (#8200) → `select-by-representation-type` (#7916), because it shares
the Representations panel and `RepresentationsData`.

## Position (maintainer-aligned)

Per Dion Moult (project lead), **if a type has representations, its occurrences should share
them** — an occurrence carrying a representation the type lacks is an *anomaly to normalize up
to the type*, not something to preserve. This follows the MVD concept-template intent (mapped
representations mirror the type relationship) and the `IfcTypeProduct` text that typed
occurrences "have to reference the representation maps", even though EXPRESS has no WHERE rule
enforcing it. See the buildingSMART thread Moult started:
<https://forums.buildingsmart.org/t/must-mappedrepresentations-come-from-the-corresponding-ifc-type/3361>.

This branch therefore provides the **normalization path**, and deliberately does *not* try to
make occurrence-local reps a first-class, persisted thing.

## Scope

**In:**

1. **Promote to Type** (`bim.promote_representation_to_type`) — lift an occurrence-local rep
   onto its type as a `RepresentationMap`, so occurrences inherit it. The migration tool for
   imported/legacy models (Revit et al. emit occurrence-only / partial-from-type reps).
2. **Type / Occurrence panel split** — `BIM_PT_representations` groups rows under **Type**
   (mapped/inherited) vs **Occurrence** (local) headers, so an anomalous occurrence-local rep
   is *surfaced* instead of silent.

**Deliberately out (dropped from the earlier draft):**

- **Copy-time preservation** — `copy_class` is left as-is (occurrence-only reps are not
  re-added on duplicate). Preserving them perpetuates the anomaly; normalize first, then copy.
- **"Add to Occurrence" toggle** — removed. `add_representation` keeps stock behaviour
  (`geometry.assign_representation` already redirects a new rep onto the type when the type has
  maps). No force-local override.

## Design

### Promote to Type

`bim.promote_representation_to_type` (`EXPORT` icon on Occurrence rows, only when
`element_has_type`) → `core.geometry.promote_representation_to_type`. Copies the local rep onto
the type as a new `RepresentationMap` (`tool.Geometry.add_type_representation_map`), then for
each occurrence of the type in the same context:

- **identical** local rep → removed (`core.remove_representation`), occurrence inherits the
  type's mapped rep (`map_representation` + `assign_representation`, which does *not* redirect
  because the rep is now a `MappedRepresentation`);
- **divergent** local rep → left untouched (kept as-is);
- **no** local rep → inherits the mapped rep.

"Identical" = `tool.Geometry.representations_are_identical`: canonical serialization ignoring
STEP ids and the shared context; styles (inverse `IfcStyledItem`) are not compared. Compared
against the **type copy**, not the original — the original is removed mid-loop when the source
occurrence is processed.

### Panel split

`RepresentationsData` (geometry/data.py) exposes `is_mapped`
(`resolve_representation(rep) != rep`), `element_is_type`, and `element_has_type`.
`draw_representation_row` is shared and carries the stack's `RepresentationIdentifier` column +
`select_by_representation_type` button; the Occurrence-group rows additionally show the promote
button. A type element shows a flat list.

## The unresolved case (needs a maintainer decision)

Two occurrences of one type that legitimately need **different** geometry in the same context
**cannot both push to the shared type** (one mapped rep per context). Promote encodes the only
physical resolution — identical → push, divergent → keep — so the tooling *is* the open
question: either those occurrences should be **distinct types**, or an occurrence-level
override must be **permitted to exist**. Intrinsic per-instance geometry (voids/joins;
`IfcRelVoidsElement` is occurrence-only) is the concrete reason the strict "never diverge" rule
can't be absolute. Pending Moult's answer on #8788.

## Status — implemented (Promote + panel verified in live Blender before the re-scope)

- `tool/geometry.py`: `copy_representation_deep`, `add_type_representation_map`,
  `representations_are_identical`.
- `core/geometry.py`: `promote_representation_to_type`.
- `core/tool.py`: interface decls for the three new `Geometry` methods.
- `bim/module/geometry/operator.py`: `PromoteRepresentationToType`.
- `bim/module/geometry/{data,ui}.py`: `is_mapped` / `element_is_type` / `element_has_type`;
  Type/Occurrence grouping merged with the stack's panel columns; old `*` suffix removed.
- `bim/module/geometry/__init__.py`: register `PromoteRepresentationToType`.
- `core/root.py`, `tool/root.py`, `core/geometry.py::add_representation`: reverted to base
  (copy-preservation + add-to-occurrence removed).

## History / pivot

Originally four pieces incl. a `copy_class` fix that re-added occurrence-only reps on duplicate,
and an "Add to Occurrence" toggle. PR #8789 was closed by Moult as "based on the wrong premise
— there shouldn't be representations on occurrence and not on type if the type has
representations." Re-scoped to the normalization-only subset above; copy-preservation and the
toggle removed.

## Things to test / verify

- Promote: identical siblings consolidate (and drawings still show the plan); a **divergent**
  sibling keeps its own geometry and does *not* pick up the type's; the source occurrence ends
  with only the mapped rep; exercises `remove_representation`'s Blender mesh/data-link side
  effects.
- `representations_are_identical` on real authored geometry (float exactness): separately
  authored "same" plans may compare unequal → treated as divergent (safe direction, no delete).
- Panel: Type vs Occurrence grouping correct for occurrence, typed occurrence with no local
  reps (only Type header), typeless element (only Occurrence), and a type element (flat list);
  columns still align with the stack's header row.
- Confirm copy/add behave as stock v0.8.0 (no regression from the removed pieces).
