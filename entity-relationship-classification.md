# Classifying an entity's relationship to a company in a corporate tree

## The one idea to hold onto

**Every relationship is relative to one chosen company (the "target"), and the target does not have to be the owner.**

The target can be the root of the tree, a company in the middle, or a leaf. The same tree gives a different set of labels for every target you pick, and nothing about the target needs to be special.

- "Ultimate Parent" is a **role in the tree** (the root). It is not the target unless the target happens to be the root.
- If the target *is* the root, there are simply no parents. Everything else is a subsidiary.
- If the target is a leaf, there are no subsidiaries.

Example: in `ROOT → P1 → (T, X)`, with `T` as the target, `P1` is the Immediate Parent and `X` is a Sibling. Choose `X` as the target instead and `T` is the Sibling. Choose `P1` and `T` and `X` are both Immediate Subsidiaries.

## The relationship vocabulary

| Relationship | Meaning, relative to the target |
|---|---|
| **Ultimate Parent** | The root of the tree |
| **Immediate Parent** | One level directly above the target |
| **Indirect Parent** | Two or more levels above the target, short of the root |
| **Immediate Subsidiary** | One level directly below the target |
| **Indirect Subsidiary** | Two or more levels below the target |
| **Sibling** | Shares the target's immediate parent |
| **Cousin** | Shares a lowest common ancestor with the target but is on a different branch, and is not a sibling. This also covers aunt/uncle-shaped and niece/nephew-shaped relations, which have no category of their own |

## An entity can have more than one relationship

The relationship is a **set**, not a single value. An entity holds every relationship that any of its positions in the tree supports.

This happens whenever an entity appears more than once in the tree, or has more than one parent:

- Under one parent it sits beside the target (**Sibling**), and under another parent it is one branch over (**Cousin**).
- It is the target's parent through one path and its grandparent through a longer one (**Immediate Parent** and **Indirect Parent**).
- It is a direct and an indirect subsidiary of the target through different paths.

The root also counts: a root that sits directly above the target is both **Ultimate Parent** and **Immediate Parent**.

Output shape per entity:

```json
{
  "externalId": "X",
  "name": "X",
  "relationships": ["Sibling", "Cousin"]
}
```

Keep the list de-duplicated and give it a stable order (see below). An entity with an empty list is not connected to the target in this tree and is left out.

## How to compute it

### 1. Flatten every placement

Walk the tree and record **every occurrence** of every entity with its parent. Use a recursion-stack guard (skip a node only if it is already in its own ancestry), not a global "visited" set, which would silently drop entities that appear beneath a duplicated placement.

### 2. Build the edge sets from the target

- `ancestors`: follow **all** parent edges upward from the target, keeping the path length for every path.
- `descendants`: follow **all** child edges downward, keeping the path length for every path.

Do not use a map of "one parent per entity". It keeps only one placement and loses the rest.

### 3. Collect the relationships for each entity E

| Condition | Adds |
|---|---|
| E is the root and an ancestor of the target | Ultimate Parent |
| E is an ancestor at path length 1 | Immediate Parent |
| E is an ancestor at path length 2 or more, and is not the root | Indirect Parent |
| E is a descendant at path length 1 | Immediate Subsidiary |
| E is a descendant at path length 2 or more | Indirect Subsidiary |
| E and the target share a parent (pair of placements with the same immediate parent) | Sibling |
| E and the target have a lowest common ancestor, E is not an ancestor or descendant on that path, and the distance pair is not (1, 1) | Cousin |

Notes:

- Apply the table to **every placement pair**, not once per entity. Each row that matches adds its label.
- Use the *lowest* common ancestor for sibling and cousin. Otherwise every sibling would also count as a cousin through the shared grandparent.
- Distance means steps up from each node to the lowest common ancestor. A pair of (1, 1) is a sibling. Anything else on a different branch is a cousin, including (1, 2), (2, 1) and (2, 2).
- The target is never reported as related to itself. If the target appears in several places, that does not create a relationship with itself.

### 4. Order and de-duplicate

Emit labels in this fixed order so output is stable:

```
Ultimate Parent, Immediate Parent, Indirect Parent,
Immediate Subsidiary, Indirect Subsidiary, Sibling, Cousin
```

## Worked example

```
ROOT
├── P1
│   ├── T        ← target
│   └── X
└── P2
    └── X        (same entity as the X above, placed again)
```

Relative to `T`:

| Entity | Relationships | Why |
|---|---|---|
| ROOT | Ultimate Parent | It is the root. It is two levels above `T`, but "Indirect Parent" excludes the root, so only Ultimate Parent applies |
| P1 | Immediate Parent | One level above `T` |
| X | **Sibling, Cousin** | Under `P1` it shares `T`'s immediate parent. Under `P2` its lowest common ancestor with `T` is `ROOT`, one branch over |
| P2 | Cousin | Its lowest common ancestor with `T` is `ROOT`. It is aunt/uncle-shaped, which is a cousin |

## What the existing `corp-tree-to-relationships` script does differently

The `corp-tree-to-relationships` skill uses the same seven labels and the same definitions, but it assigns **one** category per entity. I ran it on the example above with `T` as the target. It returned `X` as `Cousin` twice (once per placement) and never reported `Sibling`. It builds a single parent map keyed by entity ID, so the later placement overwrites the earlier one.

So to support multiple relationships per entity, the script needs to compute over all placements as described in step 3 rather than over one parent map. The labels and their definitions do not change.

One deliberate difference: the script reports the root as Ultimate Parent only, even when the root is directly above the target. This doc reports **both** Ultimate Parent and Immediate Parent in that case, because both are true. If you want strict parity with the script, drop that rule.

## Things to watch for

- **Match on ID, never name.** The same company can appear under different names in different places.
- **Multiple trees.** If the target is in more than one tree, classify within each tree and merge the label sets per entity.
- **Root versus top of the target's branch.** Ultimate Parent is always the tree's root. A parent that is the top of the target's own line, but has its own parent, is an Indirect Parent.
