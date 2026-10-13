# Relationship types by audit rule

For each audit rule, which relationship types (relative to the company being audited, the **target**) are:

- **Restricted**: in scope by the relationship alone. No further test.
- **Materiality Check**: in scope only if a materiality test is met.
- **Out of scope**: the rule does not reach this relationship.

The target can be anywhere in the tree. An entity can hold several relationships. Apply the strictest one (Restricted over Materiality Check over Out of scope).

## Where this comes from, and how far to trust it

- The repo skills (`acct-independence-rules`, `acct-audit-rule-conflicts`) use **one shared table for every rule**. They do not model the differences below.
- The differences come from web sources: rule summaries, commentary and a few primary-text excerpts. **I could not fetch the primary rule texts**, so each rule below ends with what is confirmed and what is not.
- Verify against the official texts before relying on this: eCFR 17 CFR 210.2-01, AICPA ET 0.400.02, the IESBA Handbook glossary, and EUR-Lex 32014R0537.

Relationship words: parent (ultimate, immediate, indirect), subsidiary (immediate, indirect), sibling and cousin (together: "common control"), significant influence.

---

## SEC

17 CFR 210.2-01(f)(4)

**Restricted**
- The target itself
- Parents
- Subsidiaries

**Materiality Check**
- Sibling, cousin: in scope only if the sibling **and** the target are each material to the common parent. The SEC added this two-sided test in 2020 (Release 33-10876). Before that, all common-control entities were in scope.
- Entity over which the target has significant influence: in scope unless the entity is not material to the target.

**Confirmed:** the 2020 sibling test; the downward significant-influence materiality condition.
**Not confirmed:** the exact wording for parents and subsidiaries (no materiality condition appeared in the sources); the condition on the upward significant-influence prong.

---

## AICPA

Code of Professional Conduct, ET 0.400.02. The affiliate definition applies only to financial statement attest clients (ET 0.400.16).

**Restricted**
- The target itself
- Subsidiaries: entities the target controls (item a)

**Materiality Check**
- Sibling, cousin: in scope if the sibling and the target are each material to the entity that controls both (item e).
- Entity over which the target has significant influence: in scope if material to the target (item b).

**Confirmed:** items a, b and e as described.
**Not confirmed:** the conditions on parents and on the upward significant-influence item. The skills treat parents as Restricted.

---

## IESBA-PIE

IESBA Code of Ethics, related entities. Public interest entity audit client.

**Restricted**
- The target itself
- Subsidiaries: related entities the target controls

**Materiality Check**
- Parent: one commentary says the definition excludes a parent where the target is immaterial to it, which makes this a Materiality Check.
- Sibling, cousin: in scope if both are material to the controlling entity. The only source for this is a student quiz.

**Confirmed:** for a publicly traded entity, the group audit client always includes its related entities.
**Not confirmed:** the glossary wording itself; the parent condition (the commentary may predate the 2020 SEC change); the sibling test. The scope rule in the source turns on "publicly traded entity", not on PIE as such.

---

## IESBA (non-PIE)

IESBA Code of Ethics, related entities. Audit client that is not publicly traded.

**Restricted**
- The target itself
- Subsidiaries: related entities over which the target has direct or indirect control

**Out of scope (likely)**
- Parents
- Sibling, cousin

**Confirmed:** for a non-publicly traded entity, the group audit client includes only related entities the target controls.
**Not confirmed:** whether parents and siblings are fully excluded, and any significant-influence rule.

---

## EU

Regulation (EU) 537/2014, Art. 5. Covers prohibited non-audit services. Network members of the audit firm are also covered.

**Restricted**
- The target itself (audited entity)
- Parent undertaking
- Controlled undertakings within the Union

**Out of scope**
- Sibling, cousin (not named in Art. 5(1))
- Controlled undertakings outside the Union

**Materiality Check:** none found.

**Confirmed:** Art. 5(1) names the audited entity, its parent undertaking and its controlled undertakings within the Union. Member States may apply stricter rules.
**Not confirmed:** whether "controlled undertakings" includes indirect subsidiaries.

### UK (Financial Reporting Council)

The UK version of the Regulation replaces "controlled undertakings" with "subsidiary undertakings" and drops "within the Union". The UK scope is therefore wider on subsidiaries than the EU's.

---

## Relationships the rule texts above do not cover

These come only from the repo skills. I found no rule text for them. They are not split by rule.

| Relationship to the target | Category in the skills |
|---|---|
| Board / management | Restricted |
| Beneficial owner | Restricted |
| Shareholder over 50% (or "CTP") | Restricted |
| Shareholder 20–50% | Materiality Check |
| Close-affiliate-only entity | Materiality Check |
| Shareholder under 20%, or undisclosed percentages | Neither (judgment under the general standard) |

## Summary by relationship

| Relationship | SEC | AICPA | IESBA-PIE | IESBA non-PIE | EU |
|---|---|---|---|---|---|
| Parent | Restricted | Not confirmed | Materiality Check | Out of scope (likely) | Restricted |
| Subsidiary | Restricted | Restricted | Restricted | Restricted | Restricted (within the Union) |
| Sibling, cousin | Materiality Check | Materiality Check | Materiality Check | Out of scope (likely) | Out of scope |
| Significant influence | Materiality Check (downward) | Materiality Check (downward) | Not confirmed | Not confirmed | Not covered |

## Limits

- A Materiality Check means a test applies. It does not mean the test was met.
- The skills' investment company complex rule (name patterns such as SVF, HoldCo, TopCo, L.P.) makes Materiality Check rows undetermined. Parents stay Restricted. This is not from the rule texts.
- The 20% and 50% bands are conventional proxies. The rules do not define "control" or "significant influence".
- GAO and DOL are not covered.

## Sources

- [SEC comment letter comparing the SEC and AICPA affiliate definitions](https://www.sec.gov/comments/s7-26-19/s72619-7068515-215530.pdf)
- [SEC Release 33-10876 (2020 final rule)](https://www.sec.gov/files/rules/final/2020/33-10876.pdf)
- [Wilson Sonsini summary of the 2020 amendments](https://www.wsgr.com/en/insights/sec-adopts-amendments-to-auditor-independence-requirements.html)
- [Journal of Accountancy, "Untangling client affiliates"](https://www.journalofaccountancy.com/issues/2022/aug/untangling-client-affiliates/)
- [IRBA pronouncement on listed entity and PIE definitions](https://www.irba.co.za/upload/Final%20Pronouncement%20-%20Revisions%20to%20the%20Definitions%20of%20Listed%20Entity%20and%20Public%20Interest%20Entity_Nov%2023.pdf)
- [Comparison of international standards with SEC/PCAOB rules](https://www.auditconduct.com/newsletters/comparing-international-independence-standards-for-nonaudit-services-to-sec-pcaob-independence-rules)
- [Regulation 537/2014 Art. 5, Irish standards index](https://www.charteredaccountants.ie/chariotindex/text/ec_2014_537/5.html)
- [Accountancy Europe briefing on non-audit services](https://www.accountancyeurope.eu/wp-content/uploads/BP_140825_-_Provision_of_NAS.pdf)
- [UK version of Regulation 537/2014](https://www.legislation.gov.uk/uksi/2019/177/part/4/made)
