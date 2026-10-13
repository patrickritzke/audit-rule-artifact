# Relationship types by audit rule

For each audit rule, which relationship types (relative to the company being audited, the **target**) are:

- **Restricted**: in scope by the relationship alone. No further test.
- **Materiality Check**: in scope only if a materiality test is met.

The target can be anywhere in the tree. An entity can hold several relationships. Apply the strictest one (Restricted over Materiality Check).

Source: the `acct-independence-rules` and `acct-audit-rule-conflicts` skills. They use the same category assignment for every rule below. Where the rule text may differ from that, it is listed under **Note** and is unverified.

---

## SEC

17 CFR 210.2-01

**Restricted**
- The target itself
- Ultimate parent, immediate parent, indirect parent
- Immediate subsidiary, indirect subsidiary
- Board / management
- Beneficial owner
- Shareholder over 50% (or "CTP")

**Materiality Check**
- Shareholder 20–50%
- Sibling, cousin (common control; the test has two legs, each material to the common parent)
- Close-affiliate-only entity

**Note:** whether the SEC rule adds a materiality condition for parents or subsidiaries is not confirmed.

---

## AICPA

Code of Professional Conduct, ET 0.400.02

**Restricted**
- The target itself
- Ultimate parent, immediate parent, indirect parent
- Immediate subsidiary, indirect subsidiary
- Board / management
- Beneficial owner
- Shareholder over 50% (or "CTP")

**Materiality Check**
- Shareholder 20–50%
- Sibling, cousin (two-leg test)
- Close-affiliate-only entity

**Note:** whether AICPA's affiliate definition adds conditions beyond this is not confirmed.

---

## IESBA-PIE

IESBA Code of Ethics, related entities (public interest entity audit client)

**Restricted**
- The target itself
- Ultimate parent, immediate parent, indirect parent
- Immediate subsidiary, indirect subsidiary
- Board / management
- Beneficial owner
- Shareholder over 50% (or "CTP")

**Materiality Check**
- Shareholder 20–50%
- Sibling, cousin (two-leg test)
- Close-affiliate-only entity

**Note:** the skills do not say whether the IESBA non-listed exception (below) applies to a PIE. Confirm before assuming.

---

## IESBA

IESBA Code of Ethics, related entities (non-PIE audit client)

**Restricted**
- The target itself
- Ultimate parent, immediate parent, indirect parent
- Immediate subsidiary, indirect subsidiary
- Board / management
- Beneficial owner
- Shareholder over 50% (or "CTP")

**Materiality Check** (only once the engagement is confirmed as listed)
- Shareholder 20–50%
- Sibling, cousin (two-leg test)
- Close-affiliate-only entity

If listing status is not confirmed, the Materiality Check types are left undetermined. Do not assume "non-listed".

**Note:** the real non-PIE scope may be narrower than this (for example, parents and siblings may not be automatically in scope). Unverified.

---

## EU

Regulation (EU) 537/2014, Art. 5

**Restricted**
- The target itself
- Ultimate parent, immediate parent, indirect parent
- Immediate subsidiary, indirect subsidiary
- Board / management
- Beneficial owner
- Shareholder over 50% (or "CTP")

**Materiality Check**
- Shareholder 20–50%
- Sibling, cousin (two-leg test)
- Close-affiliate-only entity

**Note:** Art. 5 appears to cover the audited entity, its parent undertaking and its controlled undertakings within the Union. If so, siblings and cousins may be out of scope rather than a Materiality Check. Unverified.

UK (Financial Reporting Council Ethical Standard) uses the same assignment as EU.

---

## Not in either list

- Shareholder under 20%, or shareholder with only undisclosed percentages: judgment under the general standard. Neither Restricted nor Materiality Check.

## Rules that apply to all of the above

- A Materiality Check means a test applies. It does not mean the test was met.
- Sibling and cousin are treated as one bucket (common control).
- Investment company complexes (detected by name pattern, for example SVF, HoldCo, TopCo, L.P.) block Materiality Check types, which become undetermined. Parents stay Restricted.
- The 20% and 50% bands are conventional proxies. The rules do not define "control" or "significant influence".
