---
name: "acct-independence-rules"
description: "Build an accounting independence screening list from a target company's corporate family. The target party IS the engagement. Reads party tree fields, resolves each entity's framework from global jurisdiction override groups (IND-SEC-PCAOB, IND-EU-537, IND-UK-FRC, IND-IESBA-PIE, IND-IESBA, IND-AICPA) falling back to 75 real cached Orbis records then a live Moody's call then deterministic synthesis, and derives audit-client status from matters via _serviceLine, and classifies EVERY relationship each entity has to the target into a restrictionBasis — automatic, materiality, judgment or undetermined — emitting a screening code per relationship with the strongest applied. Materiality is classified from structure; the screening application runs the test. Also emits the plain-language reasons and Moody's inputs the display popup shows. Trigger on \"independence screen\", \"independence list\", \"screening code\", \"restrictionBasis\", \"who needs screening for independence\", or \"run acct-independence-rules\"."
---


# Accounting Independence Screener

## Purpose
Given a target company and its corporate family, produce the list of entities requiring independence screening. Every entity receives **one row** and carries **every relationship it has** to the target, each classified into a `restrictionBasis`. The strongest is applied.

**The skill classifies relationships. The screening application runs materiality tests.** Those are separate jobs, and conflating them was the defect this model fixes.

## Three constraints
**The target party IS the engagement.** Its own `_auditIndRule` sets the governing framework. Nothing is inherited from parents.

**Party fields only.** Read `_treeIdOne`, `_treeIdTwo`, `_auditClient`, `_auditIndRule` from the party record. Do **not** read from the client record — measured on the six clients linked to parties in the SoftBank tree, `independenceRule` is **null on all six** and `auditClient` is **false on all six**, including on Arm Holdings and Ampere whose *party* records say `true`.

**A code asserts a classification, not an outcome.** `M` says "this prong applies and carries a test". It does not say the test passed. No code says "the relationship could not be classified, or an exception blocks it".

***

## `restrictionBasis`
| Value          | Code                | Meaning                                                                  |
| -------------- | ------------------- | ------------------------------------------------------------------------ |
| `automatic`    | `<FW>-R<prong>-<O>` | Control-based. No further test                                           |
| `materiality`  | `<FW>-M<prong>-<O>` | The prong carries a test. **The screener runs it**                       |
| `judgment`     | `<FW>-JGN-<O>`      | General standard only                                                    |
| `undetermined` | **none**            | Not classifiable, or an exception blocks it. `undeterminedWhy` mandatory |

`materiality` is set from **relationship type alone**. The outcome never comes back.

> **There is no `X`.** A negative outcome is the screener's record. `AUDAFF-*-X` stays provisioned but is populated downstream. **If this skill emits an `X`, that is the bug.**

> **Why an earlier version gated `M` behind overrides.** It was a *display* failure: 127 of 130 Morningstar rows came out `M`, and a badge coloured by *framework* made 130 open questions look like 130 settled restrictions. The display now encodes treatment in the pill's left edge bar — `R` solid dark, `M` solid amber — so Ampere reads "4 restricted, 446 pending". **Do not restore the gate without checking the display still separates the two.**

***

## Data model — `relationships[]` per entity
An entity can be a sibling in the tree **and** a >50% shareholder **and** a board member. Record them all, **strongest first**, so `[0]` is the applied one.

```json
{
  "id": "JP1010401056795", "name": "SOFTBANK GROUP CORP",
  "relationships": [
    { "relationshipType": "GUO", "section": "corporateTree",
      "restrictionBasis": "automatic", "ruleProng": "EU5",
      "screeningCode": "UKF-REU5-I", "citation": "Reg 537/2014 Art. 5" },
    { "relationshipType": "SignificantInfluence", "section": "closeAffiliates",
      "restrictionBasis": "materiality", "ruleProng": "43",
      "screeningCode": "UKF-M43-I", "citation": "Reg 537/2014 Art. 5" }
  ],
  "relationshipCount": 2, "appliedRelationship": "GUO",
  "basis": "automatic", "code": "UKF-REU5-I", "cite": "Reg 537/2014 Art. 5",
  "reason": "GUO — control prong, no materiality test. Also found: SignificantInfluence."
}
```

`basis` / `code` / `cite` / `reason` are the applied values, mirroring `relationships[0]`. **Do not rename them to `applied*`** — the display reads these names.

***

## Strictness ordering
```
1. automatic   >   2. materiality   >   3. judgment   >   4. undetermined
```

Within a tier, by specificity:

```
automatic:    EntityUnderAudit > tree ancestor/descendant > BoardMember > UBO > Shareholder >50%/CTP
materiality:  Shareholder 20–50% (single) > CommonControl (dual) > SignificantInfluence (single)
same prong:   tree membership > shareholder register > close affiliates
```

> **An exception outranks every tier.** If one applies, `basis` is `undetermined` regardless. Mark every entry `blocked: true` and keep them — the trail must show what would have applied.

***

## Rule tokens and citations — per FAMILY, never shared
| Framework     | Control | Common control | EUA  | Sig. influence | General |
| ------------- | ------- | -------------- | ---- | -------------- | ------- |
| `SEC`         | `41`    | `42`           | `F6` | `43`/`44`      | `GN`    |
| `AIC`         | `E22`   | `E22`          | `F6` | `43`/`44`      | `GN`    |
| `IES`         | `I20`   | `I20`          | `F6` | `43`/`44`      | `GN`    |
| `EU5` / `UKF` | `EU5`   | `EU5`          | `F6` | `43`/`44`      | `GN`    |
| `UNK`         | `GN`    | `GN`           | `F6` | `GN`           | `GN`    |

**Citations are keyed on the ENGAGEMENT FAMILY, not the token.**

| Family        | EUA                               | Control                | Common control     | Sig. influence (down / up) | General              |
| ------------- | --------------------------------- | ---------------------- | ------------------ | -------------------------- | -------------------- |
| `SEC`         | 17 CFR 210.2-01(f)(6)             | (f)(4)(i)              | (f)(4)(ii)         | (f)(4)(iii) / (iv)         | 2-01(b)              |
| `AIC`         | AICPA ET 0.400.02 — attest client | 0.400.02(a),(c)        | 0.400.02(e)        | 0.400.02(b) / (d)          | Conceptual Framework |
| `IES`         | IESBA Code — audit client         | related entity (a),(c) | related entity (e) | related entity (b) / (d)   | R120                 |
| `EU5` / `UKF` | Reg 537/2014 Art. 5               | Art. 5                 | Art. 5             | Art. 5                     | Art. 5               |

> **KNOWN DEFECT — `F6` used to cite `17 CFR 210.2-01(f)(6)` on every engagement**, so an AICPA or UK-FRC run put an SEC regulation on its entity-under-audit row. Confined to one row and therefore easy to miss. Caught by the citation-matches-family assertion. Fixed by the table above.

> **KNOWN DEFECT — `UKF` shares the `EU5` token and citation.** A UK-FRC engagement prints `Reg 537/2014 Art. 5` where an FRC Ethical Standard reference belongs. Live on the Arm Limited fixture: all 28 automatic and all 425 materiality rows. The display's popup *label* says Financial Reporting Council correctly; the tooltip citation does not. **They will not agree until the token is split.**

Also `45` (investment company complex — **not implemented**), `C1` (investments / UBO), `C2` (employment / board).

***

## Assignment
| Relationship                                     | Basis         | Code                                    |
| ------------------------------------------------ | ------------- | --------------------------------------- |
| Entity under audit                               | `automatic`   | `<FW>-RF6-<O>`                          |
| Ancestor — GUO / direct / indirect parent        | `automatic`   | `<FW>-R<ctrl>-<O>`                      |
| Descendant — direct / indirect subsidiary        | `automatic`   | `<FW>-R<ctrl>-<O>`                      |
| Management / board                               | `automatic`   | `<FW>-RC2-<O>`                          |
| Beneficial owner                                 | `automatic`   | `<FW>-RC1-<O>`                          |
| Shareholder > 50% or `"CTP"`                     | `automatic`   | `<FW>-R<ctrl>-<O>`                      |
| Shareholder 20–50%                               | `materiality` | `<FW>-M44-<O>`                          |
| Shareholder < 20%, or both fields sentinel       | `judgment`    | `<FW>-JGN-<O>`                          |
| Common control — sibling and sibling descendants | `materiality` | `<FW>-M<cc>-<O>` **+ `dualTest`**       |
| Close-affiliate-only                             | `materiality` | `<FW>-M43-<O>` down / `<FW>-M44-<O>` up |

Origin `<O>` is `D` where the entity's local framework equals the engagement framework, `I` where it differs. **It varies inside a bucket** — never write a fixture expectation assuming one value. Measured on Ampere's materiality rows: **138 `-D`, 308 `-I`**.

### Dual materiality — two legs, recorded separately
```json
"dualTest": {
  "note": "Both legs must be recorded; one outcome cannot assert two determinations.",
  "legs": [
    { "leg": "affiliate", "of": "PANTHER II 2021 HOLDINGS LIMITED",
      "materialTo": "SOFTBANK GROUP CORP", "outcome": null },
    { "leg": "entityUnderAudit", "of": "ARM LIMITED",
      "materialTo": "SOFTBANK GROUP CORP", "outcome": null } ] }
```

> **A single boolean cannot carry two determinations.** An `M42` with one leg answered is not a completed test. Assert `legs.length === 2`.

### Exceptions — these produce `undetermined`
| Exception                                                                | Applies to                                        |
| ------------------------------------------------------------------------ | ------------------------------------------------- |
| **Investment company complex** — prong `45` applies and is unimplemented | materiality rows matching the name heuristic      |
| **IESBA non-listed** — sibling scope requires listing confirmation       | materiality rows on a non-listed IESBA engagement |
| **Unresolvable entity** — no country, no provider record                 | any row                                           |
| **Chain gap** — intermediate node unresolved                             | the gap node only                                 |

> **§45 detection is a name-pattern heuristic, not a classification.** `SVF`, `SBIA`, `SB Global`, `SB Investment`, `HoldCo`, `TopCo`, `MidCo`, trailing `L.P.` Say so in `undeterminedWhy`. Measured: **92 on Ampere, 89 on Arm Limited**.

> **§45 blocks materiality rows only, not control rows.** An SVF entity that is an *ancestor* is restricted by control regardless — prong 45 adds nothing where control already establishes affiliate status unconditionally. `SVF TOPCO (CAYMAN)` and `SVF HOLDCO (UK)` are ancestors of Arm Limited and stay `automatic`.

> **Never assume non-listed for IESBA.** `LISTED` is absent from the default Orbis response and must be named in `select`.

***

## Data source

### Party fields
`_treeIdOne` / `_treeIdTwo` (tree roots). Empty second values return `""`, not `null`. Both must be queried and unioned — a tree carried only in the second field is invisible to a first-field query.

> **`_auditIndRule` and `_auditClient` are no longer read.** The framework comes from jurisdiction override groups; audit status comes from matters. Both fields were free-form or unvalidated: `SEC-PCAOB` with a hyphen fails exact match and degrades silently, and a party flagged as audited may hold no engagement at all. Neither failure is possible now.

### Fetch path
```
1. get_party(targetPartyId)          → trees, node externalIds, all four non-tree sections
2. get_component_group(treeGuoId)    → which parties, clients and matters exist in this tree
3. get_component_group(IND-<fw>)     → jurisdiction overrides, 8 global groups, union
4. get_matter(matterId)              → _serviceLine and description, per matter
5. get_client(clientId)              → clientId → partyId, per client
6. Provider data                     → only where no override group applies.
                                       Cache first, then a live Orbis call, then synthesis.
```

> **`get_party` responses are large** — 450–740KB, 12,000–19,500 lines. Process the saved result file programmatically.

> **Fetch clients in ONE call and match locally.** Per-party filtering would be 20 round trips.

> **Query BOTH tree fields and union — a correctness requirement.** CHRISTIAN DIOR and AGACHE both carry `_treeIdOne` `WW*926388678` and `_treeIdTwo` `FR921583266`. Screening `FR921583266` with `filter_treeIdOne` alone returns **zero parties**.

> **Query by tree root, never node-by-node** — 260 calls for Morningstar, 1,098 for SoftBank, 2,350 for Santander.

> **Only parties can be audit clients** — 20 of 549 nodes in SoftBank. `custom-` and `autogenerated:` nodes have no provider record and stay `UNKNOWN`.

### What the feed does NOT carry
`sales` **0 on 549/549** · `isPublicEquity` **null on 549/549** · `isFixedIncomeIssuer` **null on 549/549** · `country` the only other field.

**Materiality is still not computable from provider data.** What changed is that the skill no longer waits on a determination it cannot make.

> **`country` is not always an ISO code.** SoftBank gives two-letter codes; Morningstar gives region strings and null on 46 of 130. A fallback assuming ISO silently produces `UNKNOWN` for a whole tree. Detect the shape first.

***

## Structural classification

### Flatten with a recursion-stack guard, NOT a visited guard
```
flatten(node, depth, parentId, stack):
    if node.externalId in stack: return          # self in own ancestry — real cycle
    push { id, name, depth, parentId }
    for child in node.subCompanies:
        flatten(child, depth+1, node.externalId, stack ∪ {node.externalId})
```

> A global `visited` set returns **506 occurrences / 453 distinct** on SoftBank. Correct is **666 / 549 / 665 edges / depth 7**, dropping ~95 entities that only appear beneath a duplicated placement. The tree is **acyclic** — the gap is re-expansion of duplicated subtrees, not cycle suppression. Duplicate placement and cycles are separate problems.

### Ancestry — transitive closure over ALL parent edges
> **Three fixtures, increasing severity.** Arm Holdings **5 ancestors**; Ampere **2**; **Arm Limited 8, three of them DIRECT parents** (`ARM HOLDINGS PLC`, `SVF HOLDCO (UK)`, `KRONOS I (UK)`). A single-parent walk finds one chain and misses two controlling entities.

### Descendants — use ALL edges, not a first-occurrence parent map
> **The highest-frequency bug.** `IN0011746231` MORNINGSTAR INDIA appears as a direct child of the root **and** as the target's only subsidiary. A first-occurrence map calls it common control when the answer is control. **The tell: a tree with a known subsidiary that emits only one control row.** Keep first-occurrence only for render position.

### Classify — first match wins, catch-all mandatory
`targetIds` → Entity Under Audit · `ancestors` → GUO/Parent · `directKids` → Direct Subsidiary · `descendants` → Indirect Subsidiary · **otherwise → Common Control Affiliate**.

> An earlier draft tested only direct siblings and dropped **descendants of siblings**. Any classification whose final branch is "skip" is wrong.

### Non-tree sections
**Shareholders** — tier on `max(directSharesPercentage, totalSharesPercentage)`. AGACHE holds 1.51% directly but **98.63% in total** of CHRISTIAN DIOR. `"-"`, `"n.a."`, `"CTP"`, `""` are sentinels, not zero — `"CTP"` is control with total undisclosed. All **65** SoftBank Group Corp shareholders are sentinel-only → every one `judgment`.

**Board** — flat `P`-prefixed persons. **Empty `[]` = no population. `null` wrapper may be a failed fetch — flag it, never skip silently.**

**Beneficial owners** — recursive; may duplicate the shareholder register, dedup on ID but keep both entries with distinct `section`.

**Close affiliates** — recursive with cycles. **Flatten fully, KEEP the hierarchy, diff against the tree.** Arm Limited 34 of which **5 outside**; SoftBank Group Corp 418 with 95 outside; Santander 1,304 with 284 outside. **Entities in both get two `relationships[]` entries** — 28 such rows on Arm Limited.

> **All four sections can be empty wrappers.** On Ampere every one returns `{companyId: null, <list>: []}`. Report as `emptySections`.

### Tier comes from section membership, never `shareHoldingPercentage`
Tree → control 50%+. Close-affiliate-only → significant influence 20–50%. The percentage field is blank on **47% of Santander's** and **59% of Dior's** close-affiliate nodes.

**Close-affiliate-only entities have no `country`.** Set `local: "UNKNOWN"` with `localSource: "not resolved — entity absent from the tree feed, no Orbis lookup"`. **Never infer a jurisdiction from a BvD ID prefix.**

***

## Framework resolution
**The framework is set by jurisdiction override group membership.** One global group per framework; a party in a group takes that framework, a party in none is derived from provider data.

| Group ID        | Framework                              | Strictness |
| --------------- | -------------------------------------- | ---------- |
| `IND-SEC-PCAOB` | SEC / PCAOB                            | 6          |
| `IND-EU-537`    | EU Regulation 537/2014                 | 5          |
| `IND-UK-FRC`    | UK FRC Ethical Standard                | 4          |
| `IND-IESBA-PIE` | IESBA, public interest entity          | 3          |
| `IND-IESBA`     | IESBA Code                             | 2          |
| `IND-AICPA`     | AICPA Code of Professional Conduct     | 1          |
| `IND-GAO`       | GAO — **affiliate rules not modelled** | —          |
| `IND-DOL`       | DOL — **affiliate rules not modelled** | —          |

```
resolve(entity):
    groups = { fw : entity ∈ group(IND-fw) }
    if len(groups) == 1:  return the one,     "override group IND-<fw>"
    if len(groups) >  1:  return strictest,   "override group IND-<fw> (strictest of N - CONFLICT)"
    return provider(entity)          # cache → live Orbis → synthesis
```

**Groups are global, not per-tree** — membership is a firm-wide statement, so the same eight calls serve any target. A group that does not resolve is empty, not an error.

> **Resolve a multi-membership conflict *and* report it.** Two memberships is a mistake someone should fix; picking silently hides it. The strictness ordering is the same convention used elsewhere in this skill, with the same caveat: it is not a sourced legal hierarchy and not a total order.

> **`GAO` and `DOL` sit outside the ordering** because their affiliate definitions are not modelled. A member resolves to that framework, renders `UNK`, and must be reported as unmodelled — never quietly ranked.

> **The failure mode is now silent omission.** A party dropped from its group does not error; it quietly starts deriving. Report **resolved-by-group vs derived counts every run** so a drop shows up as a number that moved.

**The target is resolved the same way.** Its result sets the governing framework for the whole run, so an override on the target re-governs every row — the code prefix, the prong tokens and the citations all follow it.

> **`GAO` and `DOL` affiliate definitions are not modelled**, and neither has a pill colour in the display — a GAO run renders every badge grey `UNK`, indistinguishable from "framework undetermined".

### Three record gaps — report each separately
| Gap                   | Meaning                                                                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `groupCoverage`       | How many entities resolved by override group vs derived. A drop in the first number is a party silently removed from a group |
| `groupConflict`       | An entity in two or more jurisdiction groups. Resolved strictest-wins, but it is a data error                                |
| `unmodelledFramework` | A member of `IND-GAO` or `IND-DOL`, whose affiliate rules this skill does not model                                          |
| `auditSignalFallback` | Matters whose audit status came from the name prefix because `_serviceLine` was empty                                        |

> The three gaps this replaced — `auditClientNoRule`, `ruleWithoutClient`, `unrecognisedRule` — cannot occur any more. Their conditions were artefacts of reading a free-text field and a boolean that disagreed with each other.

> `IN0001040867` Arm Embedded carries `_auditIndRule: "SEC-PCAOB"` — hyphen, not slash. Silently degrades to Moody's. Report as `unrecognisedRule`.

## Provider data — cache, then tool, then synthesis
Framework resolution needs six fields per entity. Resolve them **in this order** and record which step answered:

| Step | Source                                                             | `frameworkSource` must read        |
| ---- | ------------------------------------------------------------------ | ---------------------------------- |
| 1    | The cache below — 75 real records                                  | `cache (live Orbis 2026-08-21)`    |
| 2    | A live `get_orbis_company_data` call, where the tool exists        | `Moody's Orbis (live)`             |
| 3    | Deterministic synthesis from the BvD ID prefix and the entity name | `synthesised — no provider record` |

**Never drop the label.** A consumer that cannot tell step 1 from step 3 cannot tell a provider fact from a guess.

> **Every record in this cache is real** — returned by live Orbis calls, one per BvD ID, with the full select. A hit is evidence; a miss is honestly a miss. An earlier version shipped 474 synthetic records alongside these. They were removed because a synthetic value looks identical to a real one, will decide a framework, and nobody will notice.

### Step 1 — the cache
Keyed by BvD ID. Null fields are omitted. These six are the only fields framework resolution reads, and they are exactly what the display's Moody's tab shows.

```json
{
  "AE0118238106": {"countryIso": "AE", "listed": "Unlisted", "nace": "6492", "naics2017": "5222", "ussic": "616"},
  "AU673801673": {"countryIso": "AU", "listed": "Unlisted"},
  "CA*Z00448425": {"countryIso": "CA", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "CN9455813254": {"countryIso": "CN", "listed": "Unlisted", "nace": "4651", "naics2017": "4234", "ussic": "504"},
  "DE5010223637": {"countryIso": "DE", "listed": "Unlisted", "nace": "2611", "naics2017": "3344", "ussic": "367"},
  "DE8330384011": {"countryIso": "DE", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "DK38433709": {"countryIso": "DK", "listed": "Unlisted", "nace": "6202", "naics2017": "5415", "ussic": "737"},
  "FM*110229506477": {"countryIso": "FM", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "FR399339431": {"countryIso": "FR", "listed": "Unlisted", "nace": "7219", "naics2017": "5417", "ussic": "873"},
  "GB01594599": {"countryIso": "GB", "listed": "Unlisted", "nace": "4634", "naics2017": "4248", "ussic": "518"},
  "GB02548782": {"countryIso": "GB", "listed": "Delisted", "exchange": "XLON", "nace": "6209", "naics2017": "5415", "ussic": "737"},
  "GB02557590": {"countryIso": "GB", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "GB03504622": {"countryIso": "GB", "listed": "Unlisted"},
  "GB03701846": {"countryIso": "GB", "listed": "Unlisted", "nace": "7219", "naics2017": "5417", "ussic": "873"},
  "GB05375817": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB08413587": {"countryIso": "GB", "listed": "Unlisted", "nace": "6209", "naics2017": "5415", "ussic": "737"},
  "GB09195191": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB09569889": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB10194528": {"countryIso": "GB", "listed": "Unlisted", "nace": "6430", "naics2017": "5239", "ussic": "615"},
  "GB11299879": {"countryIso": "GB", "listed": "Listed", "exchange": "XNAS", "nace": "2611", "naics2017": "3344", "ussic": "367"},
  "GB12115783": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB13644392": {"countryIso": "GB", "listed": "Unlisted", "nace": "7022", "naics2017": "5416", "ussic": "874"},
  "GB13965450": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB13965760": {"countryIso": "GB", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "GB16803102": {"countryIso": "GB", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "HK0041163231": {"countryIso": "HK", "listed": "Unlisted"},
  "HU13014962": {"countryIso": "HU", "listed": "Unlisted", "nace": "7219", "naics2017": "5417", "ussic": "873"},
  "ID0000017005C": {"countryIso": "ID", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "IE316532": {"countryIso": "IE", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "IL53-254-5118": {"countryIso": "IL", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "IN0001040867": {"countryIso": "IN", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "IN0017720824": {"countryIso": "IN", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "JP1010401056795": {"countryIso": "JP", "listed": "Listed", "exchange": "XJPX", "nace": "6190", "naics2017": "5171", "ussic": "481"},
  "JP2011101084078": {"countryIso": "JP", "listed": "Unlisted", "nace": "6492", "naics2017": "5222", "ussic": "614"},
  "JP4010401039979": {"countryIso": "JP", "listed": "Listed", "exchange": "XJPX", "nace": "6209", "naics2017": "5192", "ussic": "737"},
  "JP4010401058731": {"countryIso": "JP", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "JP4020001034215": {"countryIso": "JP", "listed": "Unlisted"},
  "JP5010001192707": {"countryIso": "JP", "listed": "Unlisted", "nace": "8299", "naics2017": "5614", "ussic": "738"},
  "JP5290001009981": {"countryIso": "JP", "listed": "Unlisted", "nace": "9004", "naics2017": "7111", "ussic": "792"},
  "JP7010401052054": {"countryIso": "JP", "listed": "Unlisted"},
  "JP7010701019678": {"countryIso": "JP", "listed": "Delisted", "exchange": "XJPX", "nace": "6209", "naics2017": "5192", "ussic": "737"},
  "JP7290001067061": {"countryIso": "JP", "listed": "Unlisted", "nace": "6499", "naics2017": "5222", "ussic": "614"},
  "KH*4000000145384": {"countryIso": "KH", "listed": "Unlisted"},
  "KR1311110513365": {"countryIso": "KR", "listed": "Unlisted", "nace": "5829", "naics2017": "5132", "ussic": "737"},
  "KR1311110701085": {"countryIso": "KR", "listed": "Unlisted", "nace": "5829", "naics2017": "5132", "ussic": "737"},
  "KYLEI1809914": {"countryIso": "KY", "listed": "Unlisted"},
  "MY1600814-V": {"countryIso": "MY", "listed": "Unlisted"},
  "NL80860125": {"countryIso": "NL", "listed": "Unlisted", "nace": "4669", "naics2017": "4238", "ussic": "508"},
  "NO983229301": {"countryIso": "NO", "listed": "Unlisted", "nace": "6202", "naics2017": "5415", "ussic": "737"},
  "NZ9429041756485": {"countryIso": "NZ", "listed": "Unlisted", "nace": "8121", "naics2017": "5617", "ussic": "734"},
  "PL368009349": {"countryIso": "PL", "listed": "Unlisted", "nace": "6209", "naics2017": "5415", "ussic": "737"},
  "SA*4000000349643": {"countryIso": "SA", "listed": "Unlisted"},
  "SE5567154868": {"countryIso": "SE", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "SE5590187448": {"countryIso": "SE", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "SG201608953D": {"countryIso": "SG", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "TH0105557074618": {"countryIso": "TH", "listed": "Unlisted", "nace": "6209", "naics2017": "5415", "ussic": "737"},
  "TW24941093": {"countryIso": "TW", "listed": "Listed", "exchange": "XTAI", "nace": "5829", "naics2017": "5132", "ussic": "737"},
  "TW50869130": {"countryIso": "TW", "listed": "Unlisted", "nace": "7112", "naics2017": "5416", "ussic": "899"},
  "TW83477590": {"countryIso": "TW", "listed": "Unlisted", "nace": "6419", "naics2017": "5221", "ussic": "602"},
  "US149127205L": {"countryIso": "US", "listed": "Unlisted", "nace": "6190", "naics2017": "5178", "ussic": "489"},
  "US268711469L": {"countryIso": "US", "listed": "Unlisted", "nace": "6419", "naics2017": "5221", "ussic": "602"},
  "US277583366L": {"countryIso": "US", "listed": "Unlisted", "nace": "6190", "naics2017": "5178", "ussic": "489"},
  "US296630873L": {"countryIso": "US", "listed": "Unlisted", "nace": "2611", "naics2017": "3344", "ussic": "367"},
  "US412397634L": {"countryIso": "US", "listed": "Unlisted"},
  "US418886604L": {"countryIso": "US", "listed": "Unlisted"},
  "US423273299L": {"countryIso": "US", "listed": "Unlisted", "nace": "6420", "naics2017": "5511", "ussic": "671"},
  "US852994421": {"countryIso": "US", "listed": "Delisted", "exchange": "XNAS", "nace": "2829", "naics2017": "3339", "ussic": "356"},
  "USLEI2116419": {"countryIso": "US", "listed": "Unlisted"},
  "VN0313309036": {"countryIso": "VN", "listed": "Unlisted", "nace": "6201", "naics2017": "5415", "ussic": "737"},
  "YY*110323231976": {"listed": "Unlisted"},
  "YY*110330318434": {"listed": "Unlisted"},
  "YY*4000000134424": {"listed": "Unlisted"},
  "YY*4000000219101": {"listed": "Unlisted", "nace": "6619", "naics2017": "5239", "ussic": "679"},
  "YY*4000000219149": {"listed": "Unlisted"},
  "YY*4000000338038": {"listed": "Unlisted"}
}
```

### Step 3 — synthesising a missing record
Two inputs exist and no others: the **ID prefix** and the **entity name**. Generate on the fly. Do not write a cache file.

**`countryIso`** — the ID prefix. A `YY` prefix means Orbis holds no country: emit `null`, never a guess. Other asterisk-shaped IDs (`CA*`, `FM*`, `KH*`, `SA*`) do resolve to their prefix.

**`listed`** — `Unlisted`, unless the company is *genuinely* publicly traded in the real world. Never infer listing from size, sector or an impressive name. Where you do assert `Listed`, use the real exchange MIC and leave `isin` null rather than minting an identifier. Where several BvD IDs share a company name, assert the listing on **one** of them — three IDs each claiming the same Nasdaq listing manufactures three issuers out of one company.

> **This field has the blast radius.** A fabricated `Listed` invents a public issuer; a fabricated `Unlisted` silently closes out an issuer test. In the real sample 68 of 75 are `Unlisted`, 4 `Listed`, 3 `Delisted`. Default hard to `Unlisted`.

**`nace` / `naics2017` / `ussic`** — pick a **matched triple** from the table below on name semantics. These are the triples actually observed in the real records, so a synthesised record speaks the same vocabulary as a fetched one. Never mix codes across rows: each field is read independently downstream, and a mismatched triple is worse than no code at all.

| NACE   | Label                                                                           | NAICS 2017 | US SIC |
| ------ | ------------------------------------------------------------------------------- | ---------- | ------ |
| `6420` | Activities of holding companies                                                 | `5511`     | `671`  |
| `6201` | Computer programming activities                                                 | `5415`     | `737`  |
| `8299` | Other business support service activities nec                                   | `5614`     | `738`  |
| `6209` | Other information technology and computer service activities                    | `5415`     | `737`  |
| `2611` | Manufacture of electronic components                                            | `3344`     | `367`  |
| `7219` | Other research and experimental development on natural sciences and engineering | `5417`     | `873`  |
| `5829` | Other software publishing                                                       | `5132`     | `737`  |
| `6202` | Computer consultancy activities                                                 | `5415`     | `737`  |
| `6209` | Other information technology and computer service activities                    | `5192`     | `737`  |
| `6419` | Other monetary intermediation                                                   | `5221`     | `602`  |

[read truncated: returned 417 of 705 requested lines (29,977 chars, budget 30,000). Use a smaller `limit` or `grep` for targeted access.]
