---
name: wikidata-brief-enrich
description: Use when the user pastes a briefing, news article, intelligence brief, or any free-text document and wants entities extracted and the text enriched with Wikidata triples (SPARQL against the local QLever Wikidata Truthy endpoint at http://localhost:7001/sparql). Triggers: "extract entities", "retrieve triples", "wikidata", "enrich this brief/report/article", "entity resolution". Do the extraction in a subagent to keep the session context clean, then emit the enhanced document.
---

# Wikidata Brief Enrichment

Resolve entities from a pasted briefing against the **local QLever Wikidata
Truthy SPARQL endpoint** (`http://localhost:7001/sparql`, English-only,
~2.9B best-rank triples, no qualifiers/references) and enrich the source text
with retrieved triples.

## Workflow (two phases — keep context clean)

1. **Subagent extraction.** Launch a `general` subagent via the Task tool to
   do ALL of the endpoint work. Give it the briefing text, the endpoint, and
   the technical notes + return-payload spec below. The subagent must return a
   single compact structured payload (not the raw query dumps).
2. **Main-context enrichment.** Use the returned payload to compose the
   enhanced document inline: keep the original text verbatim, tag first entity
   mentions with `QID` references, and add a `▸ Wikidata grounding:` line
   under each claim/SITREP tying the retrieved triples to the claim.

Never paste raw multi-row TSV into the final answer; curate to the relevant
relations.

## Endpoint notes (gained from production runs on this repo)

- Read-only `SELECT`s only. Endpoint: `http://localhost:7001/sparql`, send the
  query via `--data-urlencode 'query=...'`, accept
  `text/tab-separated-values`. Verify it is up first:
  `SELECT (COUNT(*) AS ?c) WHERE { ?s ?p ?o }` → `2926237465`.
- Declare prefixes explicitly (no auto-prefixes in this build):
  `PREFIX wd: <http://www.wikidata.org/entity/>`,
  `PREFIX wdt: <http://www.wikidata.org/prop/direct/>`,
  `PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>`.
- **No property or entity labels** carry human-readable names beyond
  `rdfs:label`; the archive holds instance data only. Map predicate IDs to
  names yourself (see whitelist).
- **Disambiguation**: a plain label match returns hundreds of homonyms
  (towns, songs, scholarly articles in US towns, etc.). Resolve by joining
  `rdfs:label` + `wdt:P31` and filtering `P31` to class QIDs via `VALUES` (a
  `FILTER(?tlabel IN (...))` on labels silently returns nothing in QLever;
  filter on class *QIDs* instead). Use class QIDs appropriate to each mention's
  domain — e.g. country `Q6256` / sovereign state `Q3624078` /
  constitutional monarchy `Q41614` / unitary state `Q179164`; organization
  `Q43229`; human `Q5`; company/business `Q4830453`; central bank `Q163740`;
  strait `Q37901`; oil and gas field `Q4291415`.
- Every object label appears under both `@en` and `@mul` → dedupe the rows.
- **Labels ≠ aliases**: an entity's stored `rdfs:label` can differ from its
  commonplace name, and aliases are NOT indexed in the truthy dump. Search the
  label you see, then fall back to alternate forms (official names, `@mul`
  variants) before giving up.
  Example: China resolves via the label `"People's Republic of China"`, not
  the alias "China".
- **Events vs persistent entities**: the archive holds persistent facts, not
  headlines. News events (strikes, elections, disputes on a given day) almost
  never have an entity; look for the ongoing underlying entity (a war, an
  institution) and flag "no stable entity" for the event itself.
  Example: the full-scale Russo-Ukrainian war = `Q110999040` (participants via
  `P710`: Russia, Ukraine; typed invasion / total war / international
  conflict) — that, not a dated strike, is what resolves.

## Resolution recipe (data-driven — no QID lookup tables)

Never include brief-specific QID lookup tables in the skill or in a subagent
prompt. Every brief has different entities; resolution must be recomputed from
the archive each run. QIDs recalled from memory are frequently stale or
mispointed (films, towns, taxa) — always confirm identity via the data.

1. **Label match + type filter.** `?s rdfs:label <mention>@en` joined with
   `?s wdt:P31 ?t`, filter `?t` to the mention's domain classes via `VALUES`
   (see class QIDs above). This usually collapses homonyms to the canonical
   entity.
2. **Multi-candidate tiebreak.** If >1 candidate survives, discriminate with
   distinguishing properties (`P571` inception, `P112` founder, `P159` HQ,
   `P17` country, `P414` listed on) and pick the candidate whose facts fit the
   brief's domain. Reject candidates whose `P31` types contradict the mention
   (a "bank" that is actually a taxon, a "company" that is a film, etc.).
3. **Zero candidates.** Retry alternate label forms (`@mul` variants, official
   names, abbreviated vs expanded), then record the mention as "not mapped".
4. **Round-trip check.** Before emitting, confirm the chosen QID's `rdfs:label`
   matches the mention's meaning; if not, go back to step 2.
5. **Cross-link.** Once entities resolve, link them via shared predicates
   (`P463` member of, `P710` participant, `P17` country, `P206` located in) to
   surface the relations the briefing actually claims (e.g., membership in
   alliances, shared conflicts, basin/coastal countries).

## Predicate whitelist (label it yourself in output)

`P31` instance of · `P30` continent · `P36` capital · `P37` official language ·
`P38` currency · `P35` head of state · `P6` head of government · `P122`
form of government · `P463` member of · `P47` shares border · `P159`
headquarters · `P571` inception · `P1448` official name · `P17` country ·
`P749` parent organization · `P27` country of citizenship · `P735`/`P734`
given/family name · `P569` date of birth · `P106` occupation · `P102` party ·
`P39` position held · `P112` founded by · `P414` listed on · `P206` located
in/on physical feature · `P205` basin country · `P488` chairperson · `P138`
named after · `P710` participant (inbound conflicts) · `P1889` different from.

## Return-payload spec for the subagent

Return ONLY:
1. `entities`: list of `{mention, qid, label, instance_of}` — every resolvable
   entity, plus explicit "not mapped" flags for generics (blocs, policies,
   hypothetical events).
2. `outbound`: selected relevant triples per entity as `subject QID | predicate
   (named) | object QID+label`, deduped, curated (NOT full dumps — pick the
   relations that matter to the briefing's claims).
3. `inbound`: key inbound sets as compact lists (member-of rosters `P463`,
   conflict participants `P710`, basin/country lists `P17`/`P205`) — whatever
   the briefing's claims implicate; do not pre-bake QIDs for specific
   alliances/unions.
4. `bridging_notes`: at most a few bullets connecting the briefing's claims to
   what is/isn't in the archive (e.g., "the accused state is NOT a member of
   the alliance").

## Output format for the final enhanced document

- Preserve the briefing's structure and wording verbatim.
- Tag the first mention of each entity inline, e.g. `Russia \`Q159\``.
- After each claim/SITREP add one `▸ Wikidata grounding:` line with the
  relevant 2–5 triples.
- End with a short "Resolution notes" section listing entities with no stable
  Wikidata match and any QID caveats.