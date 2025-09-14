# Memory Ontology Manager — Requirements Spec

**Memory Ontology Manager** — *A governance subsystem that defines, versions, and semantically annotates memory schemas. It combines a Schema Registry, which captures the structural contracts of memory types and fields, with an Annotation Engine, which enriches those schemas with searchable and identity metadata for downstream indexing and update resolution.*

“**Memory Ontology Manager**” is a strong umbrella term — it carries the right weight. It signals:

* **Ontology** → you’re not just storing schemas, you’re defining the *conceptual universe* of what a memory can be.
* **Manager** → this is an active subsystem: versioning, annotating, governing.

## Subcomponents

1. **Schema Registry**

   * The authoritative catalog of memory types, fields, and versions.
   * Think: classes, columns, constraints, pii flags.
   * Status: draft → active → deprecated.

2. **Annotation Engine** (powered by the LLM)

   * The semantic layer that enriches schemas.
   * Decides **searchable fields** (for indexing) and **identity fields** (for updates).
   * Produces confidence scores, notes, and requires approval if confidence is low.

**Together:** *Memory Ontology Manager = Schema Registry + Annotation Engine.*

**Taxonomy guidance:**

* Use *ontologies* at the highest level.
* Use *schemas* for structure.
* Use *annotations* for the LLM’s enrichments.

---

## 1) Scope

* **In scope:** schema definition and versioning; LLM-based annotations for searchable + identity fields; confidence + notes; approval requirement when confidence is low; audit trail; analyzer versioning; annotation provenance; rollback to last active version; turn pinning contract.
* **Out of scope:** extraction, retrieval, packing, memory CRUD, ranking logic, or any runtime beyond the turn-pinning contract.

---

## 2) Core Entities & Data (conceptual)

* **Schema Version**

  * `id`, `name`, `version`, `status ∈ {draft, active, deprecated}`, `created_at`, `activated_at`.
* **Class**

  * `schema_version_id`, `class_name`, `class_description`.
* **Field**

  * `class_id`, `field_name`, `dtype ∈ {string,int,number,bool,date,enum,list_string}`, `enum_values?`, `pii: bool`, `description`, `example?`.
* **Annotation** (per class)

  * `searchable_fields: [{field, boost ∈ {1,2,3}}]`
  * `identity_fields: [field, ...]`
  * `confidence_searchable`, `confidence_identity`
  * `notes` (short rationale)
  * `approval_status ∈ {pending, approved, rejected}`
  * **Provenance:** `{model_name, model_version, prompt_version, temperature, policy_flags}`
  * **Analyzer reference:** `analyzer_version`
* **Analyzer Version**

  * `{tokenizer, stopwords, stemming, normalization}` as a single immutable `analyzer_version` identifier.
* **Audit Entry**

  * `actor`, `timestamp`, `entity_ref`, `action`, `diff`, `rationale`.

---

## 3) Functional Requirements

### 3.1 Schema Registry

* **Create schema version (draft).**
  Must allow adding classes and fields (including `pii` flags, enums, and descriptions).
* **Version status transitions.**
  Supported states: `draft → active → deprecated`. Only one **active** version per schema name.
* **Activate schema version.**
  Activation **must** require that each class in the version has an **annotation** with `approval_status = approved`.

### 3.2 Annotation Engine

* **Run annotator (per schema version).**
  Produces, for each class: `searchable_fields` (with boosts), `identity_fields`, `confidence_*`, and `notes`, plus **provenance** and linked **analyzer\_version**.
* **Low-confidence gate.**
  If confidence is low (definition is implementation-defined), the annotation **must** be set to `pending` and require human approval before the schema version can be activated.
* **Edit/approve/reject.**
  Must allow updating annotations and changing `approval_status` to `approved` or `rejected`.

---

## 4) Governance Additions (required)

* **Audit trail.**
  An immutable log **must** record who changed what and when, including schema/annotation diffs and any rationale/notes.
* **Analyzer versioning.**
  `{tokenizer, stopwords, stemming, normalization}` **must** be persisted as a single `analyzer_version`. Analyzer settings **must not** change silently; changing them requires a new `analyzer_version`.
* **Annotation provenance.**
  Each annotation **must** store `{model_name, model_version, prompt_version, temperature, policy_flags}`.
* **Rollback path.**
  System **must** support reverting to the **last active** schema version in a single action; any dependent indices/maps are rebuilt automatically as part of rollback.
* **Turn pinning.**
  Every downstream conversation turn **must** pin `{schema_id, schema_version}` at turn start; there is **no mid-turn drift**.

---

## 5) Workflows (minimal)

1. **Define Schema (Draft)**

   * Create schema version (status: `draft`), add classes and fields (including `pii` flags).

2. **Annotate**

   * Run Annotation Engine → create per-class annotations with `searchable_fields`, `identity_fields`, confidences, notes, **provenance**, and **analyzer\_version**.
   * If low confidence → `pending` (requires approval). Else may be set to `approved` or still reviewed manually.

3. **Activate**

   * When all classes have `approved` annotations, set schema version to `active`. Record audit entry.

4. **Rollback**

   * If needed, revert to last `active` version in one action. Record audit entry and rebuild indices/maps automatically.

5. **Consume (Turn Pinning)**

   * Downstream systems pin `{schema_id, schema_version}` at turn start and use that version only for the turn.

---

## 6) Acceptance Criteria

* A schema version can be **activated only** when:

  * All classes have annotations with `approval_status = approved`.
  * Each annotation persists **provenance** and references an **analyzer\_version**.
* **Audit trail** records schema changes, annotation edits/approvals, activations, and rollbacks with diffs and timestamps.
* **Rollback** restores the previous `active` schema version with automatic index/map rebuild.
* **Turn pinning** contract is documented and respected by consumers (no mid-turn version changes).

---

## 7) Glossary

* **Ontology:** The conceptual universe of memory types the system recognizes.
* **Schema:** The concrete structural definition (classes, fields) within a version.
* **Annotation:** LLM-enriched metadata over a class’s fields specifying searchability (with boosts) and identity fields.
* **Analyzer Version:** Immutable identifier for the lexical analyzer configuration used by downstream indexing.
* **Turn Pinning:** Binding a conversation turn to a specific `{schema_id, schema_version}` for its entire duration.
