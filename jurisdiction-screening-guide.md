# Which relationships to screen, by audit jurisdiction

For a chosen audit jurisdiction, this lists which relationships to the company being audited (the **target**) are:

- **Restricted**: the relationship alone puts the entity in scope. No further test.
- **Materiality Check**: the relationship only restricts the entity if a materiality test passes. The screening application runs the test. Flagging a row as Materiality Check does not mean the test passed.

Everything is relative to the target. The target can be anywhere in the tree and does not have to be the ultimate parent. An entity can hold several relationships at once. Screen it at the **strictest** one (Restricted beats Materiality Check).

## How sure is each row?

| Mark | Meaning |
|---|---|
| ✅ | Taken directly from the `acct-independence-rules` / `acct-audit-rule-conflicts` skills |
| ⚠️ | My own addition. **Unverified.** Confirm with your independence / policy team before relying on it |

The skills use **one shared category table** for all five jurisdictions (SEC, AICPA, IESBA-PIE, IESBA, EU-537). Only the rule citation changes, plus one IESBA exception. So the tables below look similar. The ⚠️ rows are where I think the real rules may differ and the skills don't say.

## Relationship vocabulary

Tree relationships (see `entity-relationship-classification.md`): Ultimate Parent, Immediate Parent, Indirect Parent, Immediate Subsidiary, Indirect Subsidiary, Sibling, Cousin.

Non-tree relationships: Board / management, Beneficial owner (UBO), Shareholder, Close affiliate.

Sibling and Cousin are one bucket in the skills ("common control"). The skills treat both the same way.

---

## SEC (SEC / PCAOB)

Source: 17 CFR 210.2-01.

| Relationship to the target | Category | Basis |
|---|---|---|
| The target itself | **Restricted** | ✅ 2-01(f)(6) |
| Ultimate / Immediate / Indirect Parent | **Restricted** | ✅ (f)(4)(i) control |
| Immediate / Indirect Subsidiary | **Restricted** | ✅ (f)(4)(i) control |
| Board / management | **Restricted** | ✅ |
| Beneficial owner | **Restricted** | ✅ |
| Shareholder over 50% (or "CTP") | **Restricted** | ✅ control |
| Shareholder 20–50% | **Materiality Check** | ✅ (f)(4)(iv) significant influence, upward |
| Sibling / Cousin | **Materiality Check**, two-leg test | ✅ (f)(4)(ii) common control |
| Close-affiliate-only entity | **Materiality Check** | ✅ (f)(4)(iii) down / (iv) up |
| Shareholder under 20% | Neither (judgment only) | ✅ general standard, 2-01(b) |

⚠️ Not asserted: whether the SEC rule attaches a materiality condition to parents or subsidiaries. The skills treat both as unconditional.

---

## AICPA

Source: AICPA Code of Professional Conduct, ET 0.400.02.

| Relationship to the target | Category | Basis |
|---|---|---|
| The target itself | **Restricted** | ✅ attest client |
| Ultimate / Immediate / Indirect Parent | **Restricted** | ✅ 0.400.02(a),(c) |
| Immediate / Indirect Subsidiary | **Restricted** | ✅ 0.400.02(a),(c) |
| Board / management | **Restricted** | ✅ |
| Beneficial owner | **Restricted** | ✅ |
| Shareholder over 50% (or "CTP") | **Restricted** | ✅ control |
| Shareholder 20–50% | **Materiality Check** | ✅ 0.400.02(d) |
| Sibling / Cousin | **Materiality Check**, two-leg test | ✅ 0.400.02(e) |
| Close-affiliate-only entity | **Materiality Check** | ✅ (b) down / (d) up |
| Shareholder under 20% | Neither (judgment only) | ✅ Conceptual Framework |

⚠️ Not asserted: whether AICPA's affiliate definition adds materiality conditions the skills don't model.

---

## IESBA-PIE (IESBA, public interest entity)

Source: IESBA Code of Ethics, related-entity provisions. The skills flag the subsection numbering as unchecked.

| Relationship to the target | Category | Basis |
|---|---|---|
| The target itself | **Restricted** | ✅ audit client |
| Ultimate / Immediate / Indirect Parent | **Restricted** | ✅ related entity (a),(c) |
| Immediate / Indirect Subsidiary | **Restricted** | ✅ related entity (a),(c) |
| Board / management | **Restricted** | ✅ |
| Beneficial owner | **Restricted** | ✅ |
| Shareholder over 50% (or "CTP") | **Restricted** | ✅ control |
| Shareholder 20–50% | **Materiality Check** | ✅ related entity (d) |
| Sibling / Cousin | **Materiality Check**, two-leg test | ✅ related entity (e) |
| Close-affiliate-only entity | **Materiality Check** | ✅ (b) down / (d) up |
| Shareholder under 20% | Neither (judgment only) | ✅ R120 |

⚠️ The skills don't say whether the non-listed exception (see IESBA below) applies to PIE engagements. A PIE is not necessarily listed. I would confirm this rather than assume.

---

## IESBA (non-PIE)

Same table as IESBA-PIE, with one difference the skills do encode:

| Relationship to the target | Category | Basis |
|---|---|---|
| The target itself | **Restricted** | ✅ |
| Ultimate / Immediate / Indirect Parent | **Restricted** | ✅ related entity (a),(c) |
| Immediate / Indirect Subsidiary | **Restricted** | ✅ related entity (a),(c) |
| Board / management | **Restricted** | ✅ |
| Beneficial owner | **Restricted** | ✅ |
| Shareholder over 50% (or "CTP") | **Restricted** | ✅ control |
| Shareholder 20–50% | **Undetermined** until listing is confirmed | ✅ non-listed exception |
| Sibling / Cousin | **Undetermined** until listing is confirmed | ✅ non-listed exception |
| Close-affiliate-only entity | **Undetermined** until listing is confirmed | ✅ non-listed exception |
| Shareholder under 20% | Neither (judgment only) | ✅ R120 |

**Undetermined** here means no category is assigned yet. The skills use it when sibling scope needs a confirmed listing status. If the engagement turns out to be listed, those rows become Materiality Check, as in IESBA-PIE. Never assume "non-listed". The listing field has to be requested explicitly from the provider data.

⚠️ I believe the non-PIE IESBA scope may be narrower than the skills model. For example, parents and siblings may not be automatically in scope for a non-PIE audit client. This is my general understanding, not from the skills, and I haven't verified it.

---

## EU (EU Regulation 537/2014)

Source: Regulation (EU) 537/2014, Art. 5. The skills cite only "Art. 5" for every row. They give no per-prong reference.

| Relationship to the target | Category | Basis |
|---|---|---|
| The target itself | **Restricted** | ✅ Art. 5 |
| Ultimate / Immediate / Indirect Parent | **Restricted** | ✅ Art. 5 |
| Immediate / Indirect Subsidiary | **Restricted** | ✅ Art. 5 |
| Board / management | **Restricted** | ✅ |
| Beneficial owner | **Restricted** | ✅ |
| Shareholder over 50% (or "CTP") | **Restricted** | ✅ control |
| Shareholder 20–50% | **Materiality Check** | ✅ significant influence |
| Sibling / Cousin | **Materiality Check**, two-leg test | ✅ common control |
| Close-affiliate-only entity | **Materiality Check** | ✅ down / up |
| Shareholder under 20% | Neither (judgment only) | ✅ general standard |

⚠️ My understanding, not from the skills and unverified: Art. 5 prohibits certain non-audit services to the audited entity, its parent undertaking and its controlled undertakings within the Union. If that is right, siblings and cousins may be out of scope rather than a Materiality Check. The skills' common-control row may be broader than the regulation.

Related notes from the skills:

- EU-537 is the **strictest** framework in the skills' ordering (SEC 6, EU-537 5, UK-FRC 4, IESBA-PIE 3, IESBA 2, AICPA 1). That ordering is a stated convention, not a legal hierarchy.
- UK-FRC (the UK Financial Reporting Council Ethical Standard) is modelled with the same categories as EU-537. It shares the EU token, so UK runs also cite "Reg 537/2014 Art. 5" instead of an FRC reference. That is a known defect in the skills.
- A company listed on an EU, EEA or UK exchange resolves to EU-537, even a UK one. An unlisted UK company resolves to UK-FRC.

---

## Side by side

| Relationship to the target | SEC | AICPA | IESBA-PIE | IESBA | EU-537 |
|---|---|---|---|---|---|
| Target, parents, subsidiaries | Restricted | Restricted | Restricted | Restricted | Restricted |
| Board, beneficial owner | Restricted | Restricted | Restricted | Restricted | Restricted |
| Shareholder over 50% / CTP | Restricted | Restricted | Restricted | Restricted | Restricted |
| Shareholder 20–50% | Materiality Check | Materiality Check | Materiality Check | Undetermined (needs listing) | Materiality Check |
| Sibling / Cousin | Materiality Check | Materiality Check | Materiality Check | Undetermined (needs listing) | Materiality Check |
| Close-affiliate-only | Materiality Check | Materiality Check | Materiality Check | Undetermined (needs listing) | Materiality Check |
| Shareholder under 20% | Neither | Neither | Neither | Neither | Neither |

## Two rules that apply to every jurisdiction

1. **Sibling / Cousin is a two-leg test.** Record both legs separately: the affiliate is material to the common parent, and the target is material to the common parent. One yes/no cannot carry two answers.
2. **Investment company complexes block Materiality Check rows, not Restricted rows.** The skills detect these by name pattern (SVF, HoldCo, TopCo, trailing L.P. and similar), which is a heuristic and not a classification. Matching rows become Undetermined. A matching entity that is a parent stays Restricted.

## Limits

- The 20% and 50% bands are conventional proxies. Rule 2-01 does not define "control" or "significant influence".
- GAO and DOL are not covered here. The skills do not model their affiliate rules.
- UK-FRC has no table of its own. It uses the EU-537 categories (see the EU section).
- Materiality cannot be computed from the provider data. A Materiality Check row says a test applies.
