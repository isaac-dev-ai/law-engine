# Law Engine: How an Explanatory Metaphor Is Kept from Silently Becoming “Law”

A sandwich-shop analogy is a good way to teach implied warranty of merchantability. It is also a good way to ship a wrong rule, if the analogy is stored in the same field as the statute.

That is the architectural problem this case study is about. Law Engine is a legal-learning system over a bounded Virginia UCC corpus. Its hard design question is not “can we cite sources.” It is: **how do you keep source text, retrieval metadata, a teaching illustration, an authored inference, and a conclusion from collapsing into one paragraph that looks like authority?**

The falsified assumption: **a cited, authentic source is an authoritative legal conclusion.**

Initial user, and so far the only one: the founder, using the product to build commercial-law competence during a career transition. Adjacent jobs (small-business owner, procurement, untrained contract handler) are design hypotheses, not served users. This document does not claim those jobs have been studied.

---

## The failure a conventional content model would allow

Take Va. Code Ann. § 8.2-314(1):

> Unless excluded or modified (§ 8.2-316), a warranty that the goods shall be merchantable is implied in a contract for their sale if the seller is a merchant with respect to goods of that kind.

A tempting teaching sentence already exists in the codebase:

> Think of it like buying a sandwich from a sandwich shop: even if the cashier never says a word about it, you reasonably expect the sandwich is actually fit to eat, because that’s what a shop selling sandwiches implicitly stands behind. A friend selling you their half-eaten lunch off a park bench isn’t making that same implicit promise.

If that illustration lives in the same string as the statute — a `body` field, a markdown lesson, an LLM answer with a citation at the bottom — three silent substitutions become cheap:

1. “Fit to eat” stands in for the six merchantability standards in § 8.2-314(2).
2. “Sandwich shop” stands in for “merchant with respect to goods of that kind.”
3. The park-bench friend stands in for the whole of disclaimer-and-exclusion law under § 8.2-316, which is named by the statute and is **not ingested** as its own section.

A learner can leave the lesson able to repeat the analogy and still unable to see that the analogy is not the rule, not currentness-checked, and not a holding about food service.

Law Engine’s response is not a disclaimer at the footer. It is separate types.

---

## One worked representation (current architecture)

The implied-warranty contract in `services/pedagogical_contract.py` (`build_implied_warranty_of_merchantability_contract()`), grounded in the ingested Article 2 extract, is the smallest end-to-end example. Layers below are the actual records, not a restatement of the analogy.

### 1. Official authority (enacted text)

`Va. Code Ann. § 8.2-314(1)`, from `library/normalized/ucc/article-2-sections.json`, `source_document_id` `va-code-title-8.2-article-2`. Jurisdiction: Commonwealth of Virginia. `AuthorityType.STATUTE`. `SourceLayer.ENACTMENT`. This is one state’s enactment, not “the UCC,” and not the copyrighted ALI/ULC model text (which is not ingested).

### 2. Source / retrieval metadata

From `library/manifests/va-code-title-8.2-article-2.json`:

- `official_source_url`: `https://law.lis.virginia.gov/vacode/title8.2/`
- `publisher`: Commonwealth of Virginia, Division of Legislative Automated Systems
- `retrieval_timestamp`: `2026-08-16T09:25:29.000213`
- `licensing_status`: public-domain enacted state statute (edict of government)

Retrieval proves where the bytes came from. It does not prove the provision is the current controlling text in a live dispute.

### 3. Hash / provenance

`sha256_hash`: `a478370eede77b96f8e49f2210f51dfadac6c043f306769c6bdbebd417936dd6`. The hash identifies the ingested extract. A later amendment at the same URL would be a different document. Hashing does not version amendments.

### 4. Normalized definition

`StatuteSection` for `8.2-314`: structured `paragraphs`, `citation`, `cross_references` (`8.2-104`, `8.2-316`), `defined_terms` (empty on this section; “merchant” is defined on `8.2-104`). The trailing history string in the ingested paragraph list is `"1964, c. 219."` — captured as source text, not as an `effective_date` the engine reasons over.

`publication_or_effective_date` on this manifest is `null`. `superseded` is `false` as a stored flag, not as the output of a citator.

### 5. Pedagogical metaphor (not the rule)

`PedagogicalMetaphor` on that contract:

```
illustration:  <sandwich-shop paragraph above>
disclaimer:    "This is a simplified teaching illustration, not a legal rule --
                it never overrides governing_text_excerpt or authority_citation."
is_pedagogical_only: true
```

It is its own dataclass, not a string reused for governing text. `PedagogicalContract.governing_text_excerpt` holds the statute; `metaphor` holds the sandwich. Tests assert `is_pedagogical_only` is always true on serialization (`test_models.py`). A caller that wants the rule has to ask for a different field.

### 6. Verified fact

In the Riverside Bistro simulation (the learner-facing exercise of this same doctrine, step 3): the failed oven is one of the three the learner *accepted*; it failed in ordinary use within a week; the supplier is a merchant regularly dealing in commercial ovens; the supplier is invoking invoice language the learner already evaluated. Those are stipulated facts in the task JSON (`FactStatus` in the proof-graph vocabulary would call a stipulated pattern `ASSUMED` / `TRUSTED_FOR_ANALYSIS`). They are not findings from a docket.

### 7. Authored proposition

From the pedagogical contract, not from the statute’s mouth: the warranty “exists even though no one wrote it down or promised it out loud,” and it “attaches at the time of sale/contract formation, not at delivery.” Those are author-written teaching propositions. They are stored as `what_it_is` / `timing_notes`, not as `governing_text_excerpt`.

### 8. Deterministic inference (what the engine will actually refuse)

`Task.validate()` (and the same citation check in the learning-item builder): a task that cites a section `get_section()` cannot resolve **fails to build**. Wrong-answer distractors do not carry their own citation. This is mechanical. It is not legal inference. It will not decide whether a particular disclaimer “materially alters” a deal. It will not let a task cite § 8.2-316 as governing authority, because § 8.2-316 is **not ingested** — even though § 8.2-314(1) names it.

The syntax engine (`syntax_engine.py`) is a deterministic grammatical/structural analysis of statutory sentences. It does not apply the statute to facts.

### 9. Contestable legal interpretation

Riverside step 2’s keyed option:

> It depends — since you’re both merchants, this term could become part of the deal unless you object, but a total warranty disclaimer likely counts as a “material alteration” you’re not stuck with by default.

That sentence is an authored reading of Va. Code Ann. § 8.2-207(2) plus a claim about what “courts typically treat” as material. The engine stores it as the correct `TaskOption`, with `governing_sections` `8.2-207` and `8.2-104`. It does not have a case on point in this jurisdiction for that “typically.” A different lawyer could contest both the material-alteration characterization and the leap from “not automatically part of the deal” to “you likely still have an implied-warranty claim” in step 3.

### 10. Conclusion

Learner-facing conclusion, after the correct path: because the disclaimer likely never became part of the deal, and a merchant seller of ovens impliedly promises they are fit for ordinary use, the learner likely still has a claim. That is a conclusion node in pedagogical form. In the proof-graph vocabulary it would be a `CONCLUSION` deriving from intermediate propositions, not a `GOVERNING_AUTHORITY`.

### 11. Uncertainty / confidence

`VerificationStatus` and `ConfidenceLabel` are different enums on purpose (`services/models.py`).

The Article 2 manifest is `SOURCE_VERIFIED`. The shared mapping `verification_status_to_confidence_label()` renders `SOURCE_VERIFIED` as `LIKELY`, not `VERIFIED`. `VERIFIED` is reserved for `CROSS_VERIFIED` / `TRUSTED_FOR_ANALYSIS`. `CURRENTNESS_CHECKED` exists on the enum and is **never assigned** to an ingested source.

A disclosed leak: this pedagogical contract currently sets `confidence_label=ConfidenceLabel.VERIFIED` while its `verification_status` is `SOURCE_VERIFIED`. The two fields are not kept in lockstep by construction. The type boundary exists; the coupling is not yet mechanical. A caller that reads only `confidence_label` will overread what source verification earned.

### 12. User-facing explanation

After the learner chooses, `plain_language_feedback` plus `professional_terminology` (`implied warranty of merchantability`) plus `reasoning_chain` citing § 8.2-314 and § 8.2-207. Terminology arrives because the learner needed it to understand what just happened, which is the product bet — not because the sandwich is the law.

---

## Which layers are not equivalent

| This | is not | this |
|---|---|---|
| Source authenticity (`SOURCE_VERIFIED`) | = | controlling authority in a forum |
| Source authenticity | = | current authority (no subsequent-history pass has run) |
| Source authenticity | = | correct interpretation of the text |
| Source authenticity | = | applicability to these facts |
| SHA-256 of an extract | = | temporal validity |
| `PedagogicalMetaphor` | = | `governing_text_excerpt` |
| Authored `TaskOption.is_correct` | = | a holding |
| `ConfidenceLabel` on a teaching contract | = | the verification status of the underlying manifest |
| Virginia enactment | = | the model UCC, or another state’s enactment |

`SourceLayer` exists so an enactment is never silently labeled “the UCC.” `AuthorityType` distinguishes `STATUTE`, `JUDICIAL_HOLDING`, `DICTA`, `PERSUASIVE_OPINION`, `CLAIM_ALTERNATIVE_THEORY`. A fully `SOURCE_VERIFIED` alternative theory is still not controlling law — the models.py docstring states that orthogonality as the point of the enum.

There is no `EQUIVALENT_TO` edge in the proof graph. Shared classification is not identity (`CLASSIFIED_AS` only). Two propositions that reach different results for a named reason are linked by `DISTINGUISHED_BY`, not `NEGATED_BY`.

---

## Effective dates, amendments, subsequent history — what exists

**Present as data-shaped fields, not as an engine:**

- `SourceManifest.publication_or_effective_date` (populated on the two case-law manifests; `null` on the statute manifests)
- `SourceManifest.superseded` (boolean, currently `false` on every ingested source)
- `SourceManifest.version` (string, currently `"1"`)
- history fragments left inside statutory paragraph text (`"1964, c. 219; 2024, c. 652."` on § 8.2-102; `"1964, c. 219."` on § 8.2-314)
- `PedagogicalContract.version_or_effective_date` (the merchantability contract stores `"1964, c. 219."` as a copied history string)
- `VerificationStatus.CURRENTNESS_CHECKED` as an unused enum member
- `override_mechanism` / `binding_scope` / `hierarchy_level` on the two judicial manifests (free-text / integer, author-filled)

**Not built:**

- amendment ingestion or invalidation of a previously hashed extract when the official page changes
- a subsequent-history / citator pass (overruled, limited, distinguished by later courts)
- jurisdiction variants of the same UCC provision (only Virginia enactments are ingested; `ProvisionComparison.model_section_id` is `None` until a licensed model text exists)
- automatic mapping from `SOURCE_VERIFIED` to `CURRENTNESS_CHECKED`

Provenance without temporal validity is incomplete legal-authority management. The architecture can *represent* “this might be superseded.” It does not currently *compute* it.

---

## Current state (not a roadmap of already-finished work)

- Provenance-tracked corpus: Virginia UCC Articles 1, 2, 3, and 9 (bounded section slices, not complete titles). Orientation pages cover all 11 pre-2022 Articles conceptually; seven remain orientation-only and are labeled that way.
- Surfaces: statute browser/search, orientation, deterministic syntax analysis, Task Ladder, four multi-step simulations (consumer-to-operator Article 2 ladder, Riverside Bistro, secured-financing, negotiable-instrument/promissory-note).
- Legal Proof Graph: definition → premise → governing authority → verified fact → intermediate proposition → conclusion, with weakest-link verification over supporting authorities.
- Case-law layer: exactly two opinions, both `SOURCE_VERIFIED` as of the independent primary-source reads recorded in `library/manifests/fl-4dca-rodriguez-v-wells-fargo-2015.json` and `library/manifests/nc-app-greene-v-trustee-services-2016.json`. See the companion precedent case study. Still a bounded prototype, not a case-law database. No dedicated learner UI for the mapper.
- Public source: Apache-2.0, `github.com/Qlander17/law-engine`. This case study describes the local tree it was written against.

Rejected alternative that was cheaper: store lessons as markdown with inline citations. That model cannot type-check “this string is a metaphor” against “this string is governing text,” and it cannot fail a build when a citation points at a section that was never ingested.

---

## What was deliberately not built

- **Personalized legal advice.** Educational and technical demonstration only. The simulation content is written inside that constraint, not merely disclaimed after the fact.
- **A Mastery Engine.** A competency-state concept (`UNSEEN` → `MASTERED`) has been named and outlined at the cross-referencing GhostOS-platform level, but no complete Law-Engine-side design specification exists yet. There is no attempt history to model. Building tracking infrastructure for data that does not exist was judged lower-value than shipping the thing that would generate the data. That decision stands.
- **Breadth over depth.** Seven of eleven UCC Articles remain orientation-only.
- **A hosted production app.** The public artifact is source plus a local demo.

---

## Risks that still apply

Single-maintainer. Virginia Articles 1/2/3/9 only. No learner data, so the situation-first bet is reasoned, not measured. `ConfidenceLabel` and `VerificationStatus` can still disagree on a single `PedagogicalContract`. Metaphor/authority separation is enforced at the type level in Python; it is only as strong as every writer who might later put rule-text into `illustration` or analogy into `governing_text_excerpt`.

Success metrics, once users other than the founder exist: simulation-step completion; time-to-first-correct-action on a novel variant of a practiced competency; ratio of independent to assisted performances. None of those are claimed as measured.
