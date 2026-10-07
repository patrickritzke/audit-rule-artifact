---
name: "acct-audit-rule-engagement-tree"
description: "Given an ultimate owner BvD ID, show a small hierarchy of ONLY the parties with engagements (audit or non-audit), each with an audit/non-audit indicator and a jurisdiction pill, plus a short assessment of which audit rule applies. Pulls parties, clients and matters from the ultimate owner's component group, resolves jurisdiction for engaged parties only, and renders one fast self-contained HTML artifact. No screening codes, no relationship classification, no search creation. Trigger on \"engagement tree\", \"audit rule for this family\", \"which audit rule applies to this engagement\", \"audit jurisdiction for the engaged parties\", or \"run acct-audit-rule-engagement-tree\". For every entity with screening codes use acct-audit-rule-conflicts; for one company use acct-audit-rule-jurisdiction-single."
---

# acct-audit-rule-engagement-tree

**Ultimate owner BvD ID in. Filtered tree plus one assessment out.**

Small on purpose. Fetch only what an engagement touches, resolve jurisdiction for those parties only, render one page.

| Want | Use |
|---|---|
| Engaged parties, tree, jurisdiction, which rule applies | **this skill** |
| Every entity in the family with screening codes | `acct-audit-rule-conflicts` |
| One company's framework, explained | `acct-audit-rule-jurisdiction-single` |

## Input

`ultimateOwnerBvdId` — the BvD ID of the ultimate owner (GUO), e.g. `JP1010401056795`. Trim and uppercase; keep `*` and `-` exactly. Never guess an ID from a name.

## Steps

```
1. get_component_group(ultimateOwnerBvdId)  → parties, clients, matters in this family
2. get_client(...)                          → clientId → partyId   (ONE call, match locally)
3. get_matter(matterId)                     → _serviceLine, name    (per matter)
4. get_party(guoPartyId, propertiesToLoad:["CorporateFamily"])
                                            → the hierarchy only
5. Jurisdiction                             → engaged parties only: cache, then get_orbis_company_data
```

The component group is the same group the family skills already use for the tree (`get_component_group(treeGuoId)`). If it does not resolve, say so and stop. Do not fall back to a broader search.

**`get_party` is large (450–740KB).** Save the result and process it with a script. Never read it into context. Read only each node's `externalId`, `name`, `subCompanies`.

**Never fetch per party.** Step 2 is one call matched locally; step 5 touches only engaged parties.

## 1. Engagements

A matter carries `clientId`, not `partyId`. Walk matter → client → party.

```
isAudit(matter):
  _serviceLine non-empty array → any(e.key == "audit")      # lowercase key, never the display value
  name starts "AUDIT - "       → true    (fallback)
  name starts "NON-AUDIT - "   → false   (fallback)
  otherwise                    → false
```

Per party: `audit` = holds any audit matter; `nonAudit` = holds any non-audit matter. A party can be both. A party with no matter is **not engaged**, whatever flags it carries. Never read `_auditClient` or `_auditIndRule`, and never read status from the client record.

## 2. Filtered tree — keep the hierarchy intact

Flatten the tree with a **recursion-stack guard**, not a global visited set (duplicated subtrees must re-expand):

```
flatten(node, depth, stack):
    if node.externalId in stack: return
    emit { id, name, depth }
    for child in node.subCompanies: flatten(child, depth+1, stack ∪ {node.externalId})
```

Keep a node if it is engaged **or** has an engaged descendant. Everything else is dropped. Kept nodes retain their original depth and order, so every engaged party shows its full chain up to the ultimate owner. Ancestors that are not engaged are kept for structure only: no indicator, no pill.

If an engaged party is not found in the tree, list it under `footer` as "engaged but not in the tree". Never silently drop it.

## 3. Jurisdiction — engaged parties only

Look up each engaged party's own BvD ID in the cached Moody's data in `acct-audit-rule-conflicts` (Step 1 cache). On a miss, call `get_orbis_company_data` for that one ID with `LISTED`, `SD_MAIN_EXCHANGE_CODE`, `NACE2_CORE_CODE`, `NAICS2017_CORE_CODE`, `USSIC_CORE_CODE` and the country in the select. A party with no usable record is `UNKNOWN`. **Never synthesise a value, never infer a country from an ID prefix or a name.**

Ladder, first match wins:

1. **Issuer** — `listed == "Listed"` and an exchange: US MIC (`XNAS XNYS ARCX XASE BATS IEXG`) → `SEC/PCAOB`; EU/EEA/UK MIC → `EU-537`; any other MIC → `IESBA-PIE`. `Delisted` and `not requested` do not fire this.
2. **PIE by sector** — credit institutions and insurers only, matched as written: NACE `6419`, `6492`, `65xx`; NAICS `5221`, `5241`; US SIC `60xx`, `63xx` → `IESBA-PIE`. Never a `64xx` or `52xx` range.
3. **Country** — `US` → `AICPA`; `GB` → `UK-FRC`; EU/EEA (`AT BE BG HR CY CZ DK EE FI FR DE GR HU IE IT LV LT LU MT NL PL PT RO SK SI ES SE IS LI NO`) → `EU-537`; any other country → `IESBA`.
4. **No country** → `UNKNOWN`.

The pill shows the jurisdiction label only (`SEC/PCAOB`, `EU-537`, `UK-FRC`, `IESBA-PIE`, `IESBA`, `AICPA`, `UNKNOWN`). No rule body, no code, no prong.

Strictness rank, a stated convention and not a legal hierarchy: `SEC/PCAOB` 6 · `EU-537` 5 · `UK-FRC` 4 · `IESBA-PIE` 3 · `IESBA` 2 · `AICPA` 1 · `UNKNOWN` 0.

## 4. Assessment — above the table

One headline, then at most three short points. Computed only from the engaged parties:

| Situation | Headline |
|---|---|
| One audit client | **`<jurisdiction>` governs this engagement**, set by `<audit client name>`. |
| Several audit clients, one jurisdiction | **`<jurisdiction>` governs**, all N audit clients agree. |
| Several audit clients, different jurisdictions | **`<strictest>` applies, as the strictest of N audit clients** (`<A>`, `<B>`). Rule is applied per engagement. |
| Audit client is `UNKNOWN` | **Rule not determined** — jurisdiction for `<name>` could not be resolved. Set `warn`. |
| No audit client, non-audit only | **No audit rule is triggered by this family's engagements.** N non-audit clients only. |

Points, only where true:

- Non-audit clients to check against the governing rule, by name.
- A party whose own jurisdiction ranks **strictly higher** than the governing one (compare ranks, never enumerate frameworks).
- Any `UNKNOWN` jurisdiction, and why (no country recorded, or not in the cache and no live record).

State the governing rule's jurisdiction, not the screening code. Whether a particular service is permitted, and materiality, are the screening application's job. Say nothing about either.

## 5. Render

Read `assets/template.html`, replace `__DATA__` with the JSON below (escape `</` as `<\/`), write it to a file and publish it with the Artifact tool. The page does no fetching and no processing beyond one pass over `nodes`, so it renders immediately. Do not add filters, tabs or popups.

```json
{
  "guo": { "bvdId": "…", "name": "…" },
  "counts": { "engaged": 4, "total": 549 },
  "assessment": { "headline": "…", "warn": false, "points": ["…"] },
  "nodes": [
    { "name": "…", "depth": 0, "audit": false, "nonAudit": false, "jur": null },
    { "name": "…", "depth": 2, "audit": true,  "nonAudit": false, "jur": "SEC/PCAOB" }
  ],
  "footer": "Jurisdiction from Moody's Orbis (cache 2026-08-21)…"
}
```

`nodes` is the pre-order filtered list. `counts.total` is the number of distinct parties in the full tree. `footer` names the data source and any engaged party missing from the tree. If the Artifact tool is unavailable, print the same assessment and an indented text tree using `●` audit, `○` non-audit and `[jurisdiction]`.

## Report back

After publishing, give the link and the assessment headline. Nothing else unless something was unresolved.

## Do not

- Do not read `_auditClient`, `_auditIndRule`, or client-record flags.
- Do not resolve jurisdiction for parties without engagements.
- Do not show screening codes, restriction basis, or relationship types.
- Do not create or modify a conflicts search.
