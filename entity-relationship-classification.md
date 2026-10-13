# How we classify each entity's relationship to the target

Source: `acct-independence-rules.skill` and `acct-audit-rule-conflicts.skill` (same model; the conflicts skill adds a `createSearch` parameter and currently has jurisdiction override groups disabled).

## The one idea to hold onto

**Every relationship is relative to a single target company, and the target does not have to be the owner.**

The target is the company being screened. It is *the engagement*. It can be:

- the ultimate parent (GUO) of the tree, or
- a mid-tree subsidiary, or
- a leaf.

Nothing in the model assumes the target is at the top. The skill is run with `get_party(targetPartyId)`, and the tree it fetches is the target's whole corporate family. The target is just one node inside it.

Consequences:

1. **The same entity gets a different relationship depending on which company is the target.** Pick a different target and the labels are recomputed from scratch.
2. **"GUO" is a role, not a property of the tree.** For a mid-tree target, the GUO is simply the furthest ancestor. It is a parent, and it is restricted by control, but it is not the engagement.
3. **Nothing is inherited from parents.** The target's own record sets the governing framework for the whole run. A parent's framework does not flow down.
4. **Siblings and cousins are not "outside the tree".** Anything in the tree that is not the target, an ancestor or a descendant is a common-control affiliate of the target.

Worked example from the skill's fixtures (SoftBank tree, `JP1010401056795`):

| Target | What it makes of other nodes |
|---|---|
| **Arm Limited** (mid-tree) | 8 ancestors, three of them direct parents (`ARM HOLDINGS PLC`, `SVF HOLDCO (UK)`, `KRONOS I (UK)`); 19 descendants; everything else is common control |
| **Ampere Computing** (mid-tree) | 2 ancestors, 1 descendant |
| **Arm Holdings** (mid-tree) | 5 ancestors, 85 descendants |

`ARM HOLDINGS PLC` is a direct parent when Arm Limited is the target and is itself the engagement when Arm Holdings is the target. `SOFTBANK GROUP CORP` is the GUO in all three runs, but it is never the target.

---

## Step 1: structural classification (tree entities)

Run against the flattened tree, with the target identified by ID.

### Flatten with a recursion-stack guard

```
flatten(node, depth, parentId, stack):
    if node.externalId in stack: return          # real cycle only
    push { id, name, depth, parentId }
    for child in node.subCompanies:
        flatten(child, depth+1, node.externalId, stack ∪ {node.externalId})
```

Do not use a global `visited` set. On SoftBank it returns 506 occurrences / 453 distinct instead of the correct 666 / 549, because it drops entities that only appear beneath a duplicated placement.

### Build the sets relative to the target

| Set | How to compute it |
|---|---|
| `targetIds` | the target's own node(s) |
| `ancestors` | transitive closure over **all** parent edges, upward from the target |
| `directKids` | nodes with an edge from the target |
| `descendants` | transitive closure over **all** edges, downward from the target |

Two traps the skill calls out:

- **Ancestry must use all parent edges.** A single-parent walk finds one chain and misses other controlling entities (Arm Limited has three direct parents).
- **Descendants must use all edges, not a first-occurrence parent map.** `IN0011746231` MORNINGSTAR INDIA appears as a direct child of the root and as the target's only subsidiary. A first-occurrence map calls it common control when it is a control relationship. The tell is a tree with a known subsidiary that emits only one control row.

### Classify: first match wins, catch-all mandatory

| Order | Test | Relationship |
|---|---|---|
| 1 | in `targetIds` | Entity under audit (EUA) |
| 2 | in `ancestors` | GUO / direct parent / indirect parent |
| 3 | in `directKids` | Direct subsidiary |
| 4 | in `descendants` | Indirect subsidiary |
| 5 | **otherwise** | **Common control affiliate** (siblings *and* sibling descendants) |

Any version whose last branch is "skip" is wrong. An earlier draft tested only direct siblings and silently dropped descendants of siblings.

---

## Step 2: non-tree sections (also relative to the target)

These come from the target's own `get_party` response. An entity's tier comes from **which section it appears in**, not from `shareHoldingPercentage` (blank on 47% of Santander's and 59% of Dior's close-affiliate nodes).

| Section | Relationship | Rule |
|---|---|---|
| Shareholders | Shareholder | tier on `max(directSharesPercentage, totalSharesPercentage)` of the **target**. Over 50% or `"CTP"` is control. 20–50% is significant influence. Under 20%, or both fields sentinel, is judgment |
| Management / board | Board member | flat list of `P`-prefixed persons. `[]` means no population. A `null` wrapper may be a failed fetch, so flag it |
| Beneficial owners | UBO | recursive. May duplicate the shareholder register: dedup on ID but keep both entries with distinct `section` |
| Close affiliates | Significant influence (close-affiliate-only) | recursive, with cycles. Flatten fully, keep the hierarchy, diff against the tree |

Sentinels (`"-"`, `"n.a."`, `""`) are **not zero**. `"CTP"` means control with the total undisclosed. AGACHE holds 1.51% of CHRISTIAN DIOR directly but 98.63% in total, so it is a >50% holder.

Close-affiliate-only entities have no `country`. Set `local: "UNKNOWN"` and never infer a jurisdiction from a BvD ID prefix.

---

## Step 3: one entity, many relationships

An entity can be a tree sibling, a >50% shareholder and a board member at the same time. The model records **all** of them in `relationships[]`, strongest first, and applies `relationships[0]`.

- An entity in both the tree and the close-affiliate list gets two entries (28 such rows on Arm Limited).
- `SOFTBANK GROUP CORP` as seen from Arm Limited: applied `GUO / automatic`, plus a `SignificantInfluence / materiality` entry from the close-affiliate section.

The mirrored fields `basis`, `code`, `cite` and `reason` must equal `relationships[0]`. Do not rename them.

---

## Step 4: assign a restriction basis

| Relationship (to the target) | `restrictionBasis` | Code |
|---|---|---|
| Entity under audit | `automatic` | `<FW>-RF6-<O>` |
| Ancestor: GUO, direct or indirect parent | `automatic` | `<FW>-R<ctrl>-<O>` |
| Descendant: direct or indirect subsidiary | `automatic` | `<FW>-R<ctrl>-<O>` |
| Management / board | `automatic` | `<FW>-RC2-<O>` |
| Beneficial owner | `automatic` | `<FW>-RC1-<O>` |
| Shareholder >50% or `"CTP"` | `automatic` | `<FW>-R<ctrl>-<O>` |
| Shareholder 20–50% | `materiality` | `<FW>-M44-<O>` |
| Shareholder <20%, or both fields sentinel | `judgment` | `<FW>-JGN-<O>` |
| Common control (sibling and sibling descendants) | `materiality` | `<FW>-M<cc>-<O>` **plus `dualTest`** |
| Close-affiliate-only | `materiality` | `<FW>-M43-<O>` down / `<FW>-M44-<O>` up |

- `<FW>` is the **engagement's** framework family (the target's), not the entity's.
- `<O>` is `D` where the entity's own (local) framework equals the engagement framework, and `I` where it differs. It varies inside a bucket (Ampere materiality rows: 138 `-D`, 308 `-I`).
- `materiality` is set from the relationship type alone. The outcome never comes back. The screening application runs the test.
- **There is no `X`.** If the skill emits one, that is the bug.

### Dual materiality for common control

A sibling is only restricted if the test passes on **two legs**, recorded separately:

```json
"dualTest": { "legs": [
  { "leg": "affiliate",       "of": "<the sibling>",   "materialTo": "<common parent>", "outcome": null },
  { "leg": "entityUnderAudit","of": "<the target>",    "materialTo": "<common parent>", "outcome": null } ] }
```

Assert `legs.length === 2`. One boolean cannot carry two determinations.

### Strictness, when an entity has several relationships

```
automatic > materiality > judgment > undetermined

automatic:    EntityUnderAudit > tree ancestor/descendant > BoardMember > UBO > Shareholder >50%/CTP
materiality:  Shareholder 20–50% > CommonControl (dual) > SignificantInfluence
same prong:   tree membership > shareholder register > close affiliates
```

### Exceptions override everything

If any applies, `basis` is `undetermined`, no code is emitted, `undeterminedWhy` is mandatory, and every `relationships[]` entry is marked `blocked: true` but kept.

| Exception | Applies to |
|---|---|
| Investment company complex (prong 45, **not implemented**, detected by name pattern) | materiality rows only, never control rows |
| IESBA non-listed engagement needing listing confirmation | materiality rows on that engagement |
| Unresolvable entity (no country, no provider record) | any row |
| Chain gap (intermediate node unresolved) | the gap node only |

Example: `SVF HOLDCO (UK)` is an ancestor of Arm Limited, so it stays `automatic` even though it matches the SVF name heuristic. Prong 45 adds nothing where control already applies. A sibling with the same name pattern would be `undetermined`.

---

## Step 5: which framework governs (this is also target-relative)

- The **target's** framework governs the whole run. It sets the code prefix, the prong tokens and the citations.
- Each other entity also gets a **local** framework (jurisdiction, listing, sector). That feeds only the `D`/`I` origin and the "local attention" flag.
- Local attention fires when `STRICT[local] > STRICT[engagement]`.
- Strictness convention (not a sourced legal hierarchy): `SEC/PCAOB 6 > EU-537 5 > UK-FRC 4 > IESBA-PIE 3 > IESBA 2 > AICPA 1 > UNKNOWN 0`.

Whether the firm audits an entity is read from its **matters** (`_serviceLine` key `audit`, with a labelled name-prefix fallback), never from party flags or the client record. Client status does not change scope. Control does.

---

## Quick decision flow for one entity, given a target

```
1. Is it the target?                              → EUA, automatic
2. Is it an ancestor of the target (any path)?    → parent/GUO, automatic
3. Is it a descendant of the target (any path)?   → subsidiary, automatic
4. Otherwise, in the tree?                        → common control, materiality + dualTest
5. Also in shareholders / board / UBO / close affiliates?
                                                  → add an entry per section (tiers in Step 2)
6. Sort entries strongest first; [0] is applied.
7. Does an exception block it?                    → undetermined, no code, all entries blocked
```

## Known limits to keep in mind

- Materiality is not computable from provider data. The `M` code says a test applies, not that it was passed.
- The 20% and 50% bands are conventional proxies. Rule 2-01 does not define "control" or "significant influence".
- Prong 45 is a name heuristic (about 90 entities per run).
- `UKF` shares the `EU5` token, so UK engagements cite `Reg 537/2014 Art. 5`.
- `GAO` and `DOL` affiliate rules are not modelled.
