# Open Brain: A Twenty Minute Tour of the Domain Model

A spoken-word tour of the concepts that Open Brain models in its schema. Written for text to speech apps like Speechify on iPhone. The five diagrams referenced throughout this script live in the diagrams folder, in a subfolder named domain dash model. Each diagram is a Mermaid file. You can render them in GitHub, in any Markdown viewer that supports Mermaid, or by pasting them into the Mermaid live editor.

If you are following along visually, keep that subfolder open in a second window. If you are listening hands free, do not worry. Every diagram is described in words as we go.

---

## Section One. What We Mean by Domain Model

Welcome. This is a tour of Open Brain that focuses on the domain model. What that means is, the concepts the schema models, the shapes of the tables, and the rules the schema enforces. Not the architecture. Not the deployment. Just the data model.

If you have ever read a thoughtful database schema and felt that the table names alone told you what the product believes about the world, that is the experience we are going for here. Open Brain has opinions. The schema encodes those opinions. By the end of this tour you should be able to look at a brand new contribution and ask sensible questions like, is this a sidecar or does it modify the core, is this evidence or is it instruction grade, what is the provenance, and what is the scope.

The whole model rests on one rule that the contributing guide states bluntly. Never modify the core thoughts table structure. You can add columns. You cannot alter or drop existing ones. That rule is enforced by the automated review on every pull request. Every concept we are about to walk through obeys it.

Let us start with the atom, then layer the sidecars on top.

---

## Section Two. The Thoughts Table, the Atom of the System

Open the first diagram, named oh one thoughts and sidecars, in the domain dash model folder. The central box is the thoughts table. Around it are the sidecars that we will visit one at a time.

The thoughts table is the durable heart of Open Brain. One row per captured snippet. The columns are intentionally few. There is an i d, which is a U U I D. There is content, which is plain text. There is an embedding column, which is a fifteen hundred thirty six dimensional vector. There is metadata, which is a J S O N B column, completely flexible. There is a content fingerprint, which is a deterministic hash used to detect duplicate captures. And there are two timestamps, created at and updated at.

That is the entire core schema. Seven columns. The simplicity is the point. Every capture source ends up in the same shape. One row. One embedding. One bag of metadata.

The flexibility lives in the metadata column. Because it is J S O N B, it can hold anything the capture source decides to attach. Topics. People mentioned. Source U R I. Sensitivity tier. The schema does not constrain its shape. The downside is that querying J S O N B is slower than querying plain columns. We will see in a moment how Open Brain solves that without breaking the core rule.

The embedding column powers semantic search. Open Brain uses pgvector and an H N S W index for fast similarity lookups. The cosine distance between two embeddings approximates how related two thoughts are by meaning, not by keyword. That is what makes recall feel intelligent.

The content fingerprint supports dedup. When a recipe imports the same content twice, the fingerprint matches and the second insert can be suppressed.

That is the atom. Now let us look at how the model grows without touching it.

---

## Section Three. The Sidecar Pattern and Enhanced Thoughts

Look back at diagram one. There are two ways the model grows. The cheap way is to add a column to thoughts. The expressive way is to add a sidecar table that references thoughts dot i d through a foreign key. Both are allowed. Modifying or removing existing columns is not.

The first schema that uses the cheap way is called enhanced thoughts. It does not add any new tables. It only adds six columns. Type, which classifies the thought as an idea, a task, a person note, a reference, a decision, a lesson, a meeting, a journal, or an observation. Sensitivity tier, which can be standard or restricted. Importance, a number from zero to one hundred. Quality score, also zero to one hundred. Source type, which records where the thought came from. And enriched, a boolean for whether post processing has run.

These six fields used to live inside the metadata J S O N B column. Promoting them to top level columns lets dashboards filter and rank without parsing J S O N. The canonical upsert function keeps them in sync automatically. You write to metadata. The function mirrors the relevant fields into columns. The metadata stays the source of truth. The columns are a performance cache.

That is denormalization for a reason. You will see this pattern again in extensions, where a last contacted column on a contact gets updated by a trigger every time you log an interaction. The repo treats denormalization as a tool, not a sin, and triggers keep the copies fresh.

The enhanced thoughts schema also installs three R P C functions, where R P C stands for remote procedure call. One does full text search with a tsvector index and a substring fallback. One returns counts and top types and top topics for a time window. One finds thoughts that share topics or people with a given thought, ranked by overlap. The work happens in the database. The dashboards stay simple.

Now let us look at the expressive option. Sidecar tables.

---

## Section Four. The Knowledge Graph of Entities and Edges

Open the second diagram, named oh two knowledge graph. This one is drawn as an entity relationship diagram. Five tables. Lines between them. Cardinalities marked.

The entity extraction schema models a knowledge graph that gets built automatically from your thoughts. The main table is called entities. Each row is a canonical node. It has an entity type, which can be person, project, topic, tool, organization, or place. It has a canonical name. It has a normalized name, which is lower cased and trimmed, used for deduplication. It has an aliases column for alternate spellings. There is a unique constraint on entity type and normalized name, so the same person cannot exist twice within the same type.

The next table is called thought entities. This is the junction between thoughts and entities. Each row says, this thought mentions this entity, in this role, with this confidence, supported by this evidence payload. The mention role can be mentioned, subject, or author.

The third table is called edges. These are relationships between entities. The relation types are co occurs with, works on, uses, related to, member of, and located in. Each edge tracks a support count, a confidence, valid from and valid until timestamps, and a decay weight. Repeated extractions bump the support count instead of creating duplicates.

The fourth table is the entity extraction queue. Every thought that is inserted or whose content or metadata is updated gets enqueued automatically by a trigger. The queue has one row per thought, with a status of pending, processing, complete, failed, or skipped. A separate edge function called entity extraction worker picks up pending rows, runs natural language processing, and populates the graph. The trigger skips system generated artifacts and skips no op updates by comparing fingerprints, so the queue does not flood.

The fifth table is the consolidation log. It is an append only ledger of every dedup merge and metadata fix. Graph maintenance is messy. The log makes it auditable.

This is the engine that turns text into structure. You write a Slack message that says, we are kicking off Project Atlas with Alice and Bob. The worker creates three entity nodes. It creates two edges, Alice works on Atlas and Bob works on Atlas. It records thought entities tying the source thought to each entity. The next time the same trio appears, support counts go up and the graph gets stronger.

---

## Section Five. Typed Reasoning Edges Between Thoughts

The entity graph tells you that Alice works on Atlas. The next schema tells you that one thought supports or contradicts another. That is a different kind of relationship and it lives in a different table.

The typed reasoning edges schema adds a single table called thought edges. The vocabulary is deliberately small. Supports means thought A strengthens or provides evidence for thought B. Contradicts means A disagrees with B. Evolved into means A was replaced by a refined version B. Supersedes means A is the newer replacement for B, typically used for decisions and document versions. Depends on means A is conditional on B. And related to is the generic fallback.

Each edge has a confidence, a decay weight, valid from and valid until timestamps, a classifier version, a support count, and metadata. The classifier version is important. It records which version of the reasoning model produced the edge, so you can replay or revisit old classifications when the model improves.

The same schema adds three temporal validity columns to the existing entity edges table. Valid from, valid until, and decay weight. That lets you say, Alice worked on Atlas from January through June, and after June the relationship lapsed. Decisions evolve. Hypotheses get overturned. Validity windows and decay weights let the graph age gracefully instead of accumulating stale facts.

The upsert function for thought edges is worth a mention. When the same edge is classified again, it bumps the support count, takes the max of the confidence values, takes the earliest valid from, and extends the valid until. A null valid until means still current, so the upsert prefers null. Repeated signal strengthens the edge.

Row level security on reasoning edges is strict. Service role only. Exposing reasoning edges to authenticated users would leak derived relationships between private thoughts.

---

## Section Six. Provenance, Derivation, and Supersession

Open the third diagram, named oh three provenance and workflow. The top half shows a provenance example. The bottom half shows the workflow status state machine.

The provenance chains schema adds four columns directly to the thoughts table. Derived from is a J S O N array of parent thought identifiers. Derivation method is a text field that today holds the value synthesis or null. Derivation layer is either primary or derived. Primary means an atomic capture. Derived means a regenerable artifact, like a weekly digest. Supersedes is an optional U U I D pointing to the prior thought this one replaces.

Why does this matter. When you let an A I produce a weekly digest from twenty atomic thoughts, you lose the evidence. The digest looks like a free standing paragraph. Provenance fixes that. The digest row carries an array of parent identifiers. You can walk upward and see exactly which atomic thoughts contributed.

The schema installs four helper functions. Trace provenance walks upward through derived from, with depth and node caps and cycle detection. Find derivatives does the reverse lookup, asking what artifacts were derived from this atomic thought. Restricted rows are hard filtered out of find derivatives, with no client side opt out, on purpose. The other two functions do server side atomic merges of metadata, which solves a real race when two processes update the same metadata blob at the same time.

The supersedes pointer is the smaller cousin of derivation. When an updated digest replaces an older one, the new digest carries a supersedes pointer to the old one. Combined with the supersedes relation on thought edges we saw a moment ago, you get both a fast pointer and a confidence weighted edge. The pointer is for the application. The edge is for the audit trail.

Now the bottom half of the diagram. The workflow status schema is much smaller. It adds two columns to thoughts. Status, which can be new, planning, active, review, done, or archived. And status updated at, a timestamp. Only task and idea types get a non null status. Everything else stays null. There is a partial index on status that only indexes non null rows, so the index stays small. The schema is intentionally a simple state machine without explicit transitions. The application enforces meaningful moves.

---

## Section Seven. Agent Memory, Trust and Audit

Open the fourth diagram, named oh four agent memory. It is an entity relationship diagram showing seven sidecar tables and a recall flow that crosses several of them. This is the most elaborate schema in the repo, and it encodes a specific point of view about what agents are allowed to remember.

The main table is called agent memories. Each row links to a thought and adds a long list of governance fields. Workspace identifier. Project identifier. Visibility, which scopes the memory across personal, channel, project, workspace, or organization. Memory type, which can be decision, output, lesson, constraint, open question, failure, artifact reference, or work log. Provenance status, which can be observed, inferred, user confirmed, imported, generated, superseded, or disputed. Confidence between zero and one. Created by, which can be user, agent, system, or import. Review status, which can be pending, confirmed, evidence only, restricted, rejected, stale, or merged.

Two boolean fields encode the trust model. Can use as evidence defaults to true. Can use as instruction defaults to false, and a check constraint enforces that it can only become true when the provenance status is user confirmed or imported. That single check is the heart of the agent memory product guardrails that live in the agents file at the root of the repo. Treat inferred or generated memory as evidence. Instruction grade memory requires a human or a trusted import.

The other six tables hang off agent memories or off the recall flow.

Source refs links each memory to the external sources that fed it. Artifacts catalogs downstream artifacts associated with the memory.

Relations is the peer relationship graph. From memory, to memory, and a relation type like supersedes, conflicts with, or merged into. This is how deduplication and conflict detection are modeled.

Review actions is the human audit trail. Every action a reviewer takes on a memory gets a row. Confirm. Edit. Mark stale. Reject. Dispute. Each row carries a before and after snapshot in J S O N B, so you can replay history.

Recall traces records every recall request. Workspace. Runtime. Task. The query that was asked. The response policy.

Recall items is the per result table. For each trace, there is a row for each memory that was considered. Rank. Similarity. Ranking score. Whether the memory was returned. Whether it was used. Why it was ignored if it was ignored.

Audit events is the system wide append only ledger. Recall requested. Memory returned. Memory used. Memory ignored. Memory written. Memory confirmed. Memory rejected.

Take a step back. This model lets you ask very specific questions. Which memories did the agent use on this task. Which did it ignore and why. Which were instruction grade and how did they get there. The schema is structured so those questions have S Q L answers, not guesses.

---

## Section Eight. Extension Schemas, Domain Knowledge as Tables

Open the fifth diagram, named oh five extension schemas. It is a grid of six subgraphs, one per extension in the learning path.

Each extension introduces its own user facing tables, and they follow the same shared design language. Row level security scoped by auth dot u i d equals user identifier. An updated at trigger on every mutable table. Denormalized lookups kept fresh by triggers. J S O N B for flexible domain fields. Indexes tuned for the dashboard access patterns each extension cares about.

The household knowledge extension adds two tables. Household items, with name, category, location, and flexible details. Household vendors, with contact info and a rating. Two independent tables.

The home maintenance extension adds two tables that do relate. Maintenance tasks, with frequency in days, last completed, and next due. Maintenance logs, which records each completion. A trigger on logs updates the parent task's last completed and next due. That is the denormalization pattern in action.

The family calendar extension adds three tables. Family members. Activities, which can be recurring by day of week or one time within a date range. Important dates, with a recurring yearly flag.

The meal planning extension adds three tables. Recipes, with ingredients and instructions stored as J S O N B arrays so you do not need separate join tables. Meal plans, tying a recipe to a week, day, and meal type. Shopping lists, with items stored as a J S O N B array including a purchased boolean for the checkbox state. This is also the first extension that introduces row level security beyond simple ownership, with a household member role for shared access.

The professional C R M extension adds three tables. Professional contacts, with a generated full text search column and a follow up date. Contact interactions, which records every meeting, call, email, or event. Opportunities, with a stage that walks from identified through proposal, negotiation, won, or lost. A trigger keeps last contacted on the contact row updated whenever an interaction is logged.

The job hunt pipeline extension is the deepest relational model in the curated path. Five tables forming a chain. Companies. Job postings, with salary ranges and requirements. Applications, which walks through draft, applied, screening, interviewing, offer, accepted, or rejected. Interviews, with type, scheduled at, and a rating. Job contacts, with a loose application managed link back to the C R M contacts table from extension five. The two extensions integrate without enforcing a hard foreign key. The link is conceptual rather than required.

---

## Section Nine. The Design Language That Repeats

Step back from any one schema and you start to see a design language that repeats across all of them.

Pattern one is the sidecar rule. Every schema that adds new concepts either adds columns to thoughts or adds sidecar tables that reference thoughts dot i d. Nothing modifies existing columns. That keeps the system pluggable and migration safe.

Pattern two is denormalization with triggers. When a query needs to be fast, the schema copies a value into a column and keeps it fresh with a trigger. Last contacted on professional contacts. Last completed on maintenance tasks. The mirrored type and importance columns on enhanced thoughts. The denormalized copy is never the source of truth. The trigger reconciles it.

Pattern three is row level security everywhere. Core thoughts and agent memory and reasoning edges are service role only. Personal extensions use auth dot u i d equals user identifier. Shared extensions like meal planning add a household member role. Restricted thoughts are hard filtered from sensitive functions with no client side opt out.

Pattern four is confidence and evidence. Almost every relationship table carries a confidence number, a support count, and an audit trail. The schema does not pretend to be certain. It records the strength of belief.

Pattern five is unique constraints on the natural keys of relationships. The triple of thought, entity, and role. The triple of from entity, to entity, and relation. The triple of from thought, to thought, and relation. These constraints prevent double counting and make idempotent ingest possible.

Pattern six is the queue and worker split. Entity extraction enqueues thoughts and a separate worker processes them. The agent memory recall flow records a trace and the items returned in two tables. The pattern separates ingest from processing so ingest stays fast and processing stays observable.

---

## Section Ten. Closing Thoughts

That is the domain model. In one sentence. The thoughts table is the atom, every other concept is a sidecar that attaches without modifying it, and the sidecars layer trust, derivation, reasoning, classification, and lifecycle on top of plain text and vector embeddings.

If you are reading or writing schema contributions, the questions to keep in your head are these. Does this modify existing columns on thoughts. If yes, find a different design. Does this introduce a relationship. If yes, define the unique constraint and the support count and the confidence. Does this hold derived data. If yes, set row level security to service role only. Does this serve a user facing extension. If yes, scope it by auth dot u i d. Does this need an audit trail. If yes, model it as an append only log with before and after snapshots.

You now have the mental model. Open any schema file in the repo and you should be able to predict the structure before you read it. That is what a coherent domain model feels like. Welcome to Open Brain at the table level.

---

## Listening Notes

This script runs about three thousand three hundred words. At Speechify's default reading speed on iPhone, around one hundred sixty five words per minute, it is roughly twenty minutes of audio. At one point two five times speed, around sixteen minutes. At one point five times, around thirteen minutes.

The five diagrams live in the diagrams folder, in a subfolder named domain dash model. They are written in Mermaid syntax.

Diagram one. Thoughts and sidecars. The core table and everything that attaches to it.

Diagram two. Knowledge graph. Entities, edges, thought entities, thought edges, and the extraction queue.

Diagram three. Provenance and workflow. The derivation chain and the kanban state machine.

Diagram four. Agent memory. The seven sidecar tables that govern trust, scope, recall, and audit.

Diagram five. Extension schemas. The user facing tables added by the six learning path extensions.
