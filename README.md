# db-pedia

![logo](logo.jpeg)

A local, read-only, **English-only** Wikidata **Truthy** SPARQL endpoint for
personal OSINT work: it provides offline, verifiable knowledge grounding for
[MacronX](https://github.com/macron1-automations/macronx) and its companion
[wikidata-brief-enrich](https://github.com/macron1-automations/wikidata-brief-enrich)
agent skill.

Built with [QLever](https://qlever.cs.uni-freiburg.de/) running in Docker, it
is designed for **modest, consumer-level hardware**: the English-only index,
on-disk-compressed vocabulary, and external-memory build budget keep peak RAM
low while indexing and serving queries — a small desktop or even a laptop can
host it. Everything runs on localhost, so queries never leave the machine,
matching the privacy-first, cheap-to-run ethos of the macronx OSINT stack.

The Python `qlever` command-line tool is used to download the dump, build the
index, and run the server via the `docker.io/adfreiburg/qlever:latest`
container image.

> The concrete numbers in this README (RAM, cores, timings, sizes) come from
> the development machine that built this index. They are reference points,
> not requirements — see [Tuning notes](#tuning-notes) for scaling down.

## What this powers

This endpoint is the knowledge substrate for the macronx OSINT pipeline:

- **[macronx](https://github.com/macron1-automations/macronx)** — a personal
  OSINT intake and analysis pipeline. Its [News Analysis](https://github.com/macron1-automations/macronx#news-analysis)
  workflow (prompt: [prompts/news_analysis.md](https://github.com/macron1-automations/macronx/blob/main/prompts/news_analysis.md))
  synthesizes daily feeds into an **Executive Intelligence Brief (EIB)** —
  BLUF, cross-sector synthesis, sector SITREPs, and a watchlist.
- **[wikidata-brief-enrich](https://github.com/macron1-automations/wikidata-brief-enrich)**
  — an agent skill that turns that EIB into a referenceable artifact: it
  resolves every named entity to a Wikidata QID and appends
  `▸ Wikidata grounding:` triples beneath each claim.

```
news feeds ──▶ macronx ──▶ news_analysis (local LLM) ──▶ EIB
                                                          │
                                                          ▼
                                            wikidata-brief-enrich ──▶ QID-tagged, triple-grounded brief
                                                          ▲
                                       SPARQL ── localhost:7001 ── this repo
```

The audience is individual OSINT analysts running macronx: anyone who wants
their intelligence products checked against a private, always-available
Wikidata archive instead of a public endpoint.

## Repository layout

This repo ships **only documentation**. The two working/data directories are
gitignored and rebuilt from scratch:

| Directory          | Contents                                                                |
| ------------------ | ----------------------------------------------------------------------- |
| `olympics-test/`   | Small smoke-test dataset (~1.7M triples) used to validate the toolchain  |
| `wikidata-truthy/` | The Wikidata Truthy index (~2.9B triples, English-only), ~130 GB on disk |

Everything needed to recreate them is embedded below.

## Prerequisites

- Docker (any platform with a working daemon)
- The `qlever` CLI (Python, installed via pipx)
- ≈180 GB free disk, and patience: download ~2-4 h, index ~1.5-2 h
  (≈43 GB compressed dump + ≈132 GB index). RAM-light by design — see
  [Tuning notes](#tuning-notes) for budgets that scale to small hosts.

### Install the `qlever` CLI

The steps are identical on any OS with Docker and Python. Install `pipx`
directly, or via your package manager — e.g.:

```bash
# Debian / Ubuntu
sudo apt install pipx
# Arch-based Linux
sudo pacman -S python-pipx
# macOS
brew install pipx
```

Then:

```bash
pipx install qlever
export PATH="$HOME/.local/bin:$PATH"        # add to your shell rc
qlever --version                            # 0.6.0
```

## Phase 0: smoke test (olympics)

Validate the whole toolchain on a 13 MB dataset before committing to a multi-hour
Wikidata build.

```bash
mkdir -p olympics-test && cd olympics-test
qlever setup-config olympics   # writes Qleverfile (see below if offline)
qlever get-data                # downloads olympics-nt-nodup.zip, 323 MB unpacked
qlever index                   # ~10 s, builds olympics.index.*
qlever index-stats             # expect ~1.7M triples
```

Reference `Qleverfile` for the smoke test (`qlever setup-config olympics`
generates an equivalent one):

```ini
[data]
NAME              = olympics
BASE_URL          = https://github.com/wallscope/olympics-rdf
GET_DATA_CMD      = curl -sLo olympics.zip -C - ${BASE_URL}/raw/master/data/olympics-nt-nodup.zip && unzip -q -o olympics.zip && rm olympics.zip
DESCRIPTION       = 120 Years of Olympics, data from ${BASE_URL}

[index]
INPUT_FILES     = olympics.nt
CAT_INPUT_FILES = cat ${INPUT_FILES}
SETTINGS_JSON   = { "ascii-prefixes-only": false, "num-triples-per-batch": 100000 }

[server]
PORT               = 7019
ACCESS_TOKEN       = olympics_7643543846L8tht6YadqPt
MEMORY_FOR_QUERIES = 5G
CACHE_MAX_SIZE     = 2G
TIMEOUT            = 30s

[runtime]
SYSTEM = docker
IMAGE  = docker.io/adfreiburg/qlever:latest
```

## Phase 1: Wikidata Truthy — config and download

```bash
mkdir -p wikidata-truthy && cd wikidata-truthy
```

Write this `Qleverfile`:

```ini
# Qleverfile for Wikidata Truthy, tuned for the reference machine (23 GB RAM, 16 cores)
#
# qlever get-data  # ~2-4 h, ~41 GB compressed
# bash index.sh    # ~1.5-2 h, RAM-lean (STXXL 6G, on-disk vocab, external IRIs)
# bash server.sh   # starts the server (see note below)
#
# Truthy dump: best-rank statements only, no qualifiers, no references.
# Based on the reference Qleverfile.wikidata-truthy (ad-freiburg/sparqloscope).

[DEFAULT]
NAME = wikidata-truthy

[data]
GET_DATA_URL    = https://dumps.wikimedia.org/wikidatawiki/entities
GET_DATA_CMD    = curl -LRC - -O ${GET_DATA_URL}/latest-truthy.nt.bz2 -O ${GET_DATA_URL}/latest-lexemes.ttl.bz2 2>&1 | tee wikidata-truthy.download-log.txt
DATE            = $$(date -r latest-truthy.nt.bz2 +%d.%m.%Y || echo "NO_DATE")
DESCRIPTION     = Wikidata "Truthy" dump (best-rank, no qualifiers/references) from ${GET_DATA_URL}, version ${DATE}

[index]
# English-only build: the truthy NT stream is filtered in `index.sh` (only
# literal language tags "en" and "mul" survive). Lexemes are EXCLUDED: they are
# multilingual word data and not safely line-filterable (run `qlever index` or
# `bash index.sh` to rebuild).
INPUT_FILES      = latest-truthy.nt.bz2
MULTI_INPUT_JSON = [{ "cmd": "lbzcat -n 4 latest-truthy.nt.bz2 | awk \"!(/\\\"@/) || /\\\"@(en|mul)[[:space:].]/\"", "format": "ttl", "parallel": "true" }]
SETTINGS_JSON    = { "languages-internal": [], "prefixes-external": [""], "locale": { "language": "en", "country": "US", "ignore-punctuation": true }, "ascii-prefixes-only": true, "num-triples-per-batch": 5000000 }
STXXL_MEMORY     = 6G
VOCABULARY_TYPE  = on-disk-compressed

[server]
PORT                        = 7001
ACCESS_TOKEN                = ${data:NAME}
MEMORY_FOR_QUERIES          = 8G
CACHE_MAX_SIZE              = 3G
CACHE_MAX_SIZE_SINGLE_ENTRY = 1G
TIMEOUT                     = 300s

[runtime]
SYSTEM = docker
IMAGE  = docker.io/adfreiburg/qlever:latest

[ui]
UI_CONFIG = wikidata
```

Download the dump (backup/XY-resumable):

```bash
qlever get-data    # or: curl -LRC - -O https://dumps.wikimedia.org/wikidatawiki/entities/latest-truthy.nt.bz2
# ~2-4 h, expect latest-truthy.nt.bz2 ~43 GB
```

## Phase 2: filtered index

> NOTE: only `latest-truthy.nt.bz2` is indexed. `latest-lexemes.ttl.bz2` is
> deliberately excluded - lexemes are multilingual word data whose Turtle file
> is full of multi-line object lists, so it cannot be safely line-filtered to
> English. The English filter keeps every literal whose language tag is `en` or
> `mul` and drops all other language-tagged literals; IRIs and untagged values
> are kept unchanged.

Write `index.sh` in `wikidata-truthy/`:

```bash
#!/usr/bin/env bash
# Rebuild the index with an English filter: only literal language tags "en" and
# "mul" survive; other language-tagged literals are dropped (streaming awk).
set -euo pipefail
DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$DIR"

echo '{ "languages-internal": [], "prefixes-external": [""], "locale": { "language": "en", "country": "US", "ignore-punctuation": true }, "ascii-prefixes-only": true, "num-triples-per-batch": 5000000 }' > wikidata-truthy.settings.json

docker run --rm -u "$(id -u):$(id -g)" -v /etc/localtime:/etc/localtime:ro \
  --mount "type=bind,src=$DIR,target=/index" -w /index \
  --name qlever.index.wikidata-truthy --init --entrypoint bash \
  docker.io/adfreiburg/qlever:latest \
  -c 'ulimit -Sn 500000 && qlever-index -i wikidata-truthy -s wikidata-truthy.settings.json --vocabulary-type on-disk-compressed -f <(lbzcat -n 4 latest-truthy.nt.bz2 | awk "!(/\"@/) || /\"@(en|mul)[[:space:].]/") -g - -F ttl -p true --stxxl-memory 6G 2>&1 | tee wikidata-truthy.index-log.txt'
```

Build it (stop the server first if it is running):

```bash
bash index.sh    # ~1.5-2 h on the reference machine
# finishes with "Index build completed"; ~2.93B triples in the final index
```

## Phase 3: start the server

> **Do not rely on `qlever start`.** On this setup it hangs without output: the
> container image's entrypoint requires `-w /data`, `qlever start` runs
> docker-in-docker (no docker executable inside the container), and the entry
> point's `usermod`/`groupmod` fail for UID 1000. Use `server.sh` below instead
>
> - it is exactly the command captured from `qlever start --show` and works
> reliably.

Write `server.sh` in `wikidata-truthy/`:

```bash
#!/usr/bin/env bash
# Start/stop the Wikidata Truthy QLever server container.
# NOTE: `qlever start` (host CLI) is unreliable here; this is the command it
# would run, captured from `qlever start --show`.
set -euo pipefail

NAME=qlever.server.wikidata-truthy
DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PORT=7001

case "${1:-start}" in
  start)
    docker run --restart=unless-stopped -d \
      -u "$(id -u):$(id -g)" \
      -v /etc/localtime:/etc/localtime:ro \
      --mount "type=bind,src=$DIR,target=/index" \
      -p "$PORT:7001" -w /index \
      --name "$NAME" --init --entrypoint bash \
      docker.io/adfreiburg/qlever:latest \
      -c "qlever-server -i wikidata-truthy -j 8 -p 7001 -m 8G -c 3G -e 1G -k 200 -s 300s -a wikidata-truthy > wikidata-truthy.server-log.txt 2>&1"
    echo "started (or attempted); wait ~10-20s then query http://localhost:$PORT/sparql"
    ;;
  stop)
    docker stop "$NAME" >/dev/null 2>&1
    docker rm "$NAME" >/dev/null 2>&1
    echo "stopped"
    ;;
  status)
    docker ps --filter "name=$NAME" --format '{{.Names}} {{.Status}}'
    ;;
  *)
    echo "usage: $0 {start|stop|status}" >&2
    exit 1
    ;;
esac
```

```bash
bash server.sh start    # HTTP endpoint on http://localhost:7001/sparql
bash server.sh status
bash server.sh stop
```

## Querying

Declare prefixes explicitly (auto-prefixes are not active in this build):

```bash
curl -s http://localhost:7001/sparql \
     --data-urlencode 'query=PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT ?city ?cityLabel ?pop WHERE {
  ?city wdt:P31 wd:Q515 ; wdt:P1082 ?pop ; rdfs:label ?cityLabel .
} ORDER BY DESC(?pop) LIMIT 5' \
     -H "Accept: text/tab-separated-values"
```

English labels come back directly (no `FILTER(LANG(...))` needed) because
non-English literals were dropped at index time. Wikidata labels whose value is
language-independent are stored under `@mul`; those are kept too.

## Using the brief-enrichment skill

This repo ships an opencode skill (`.opencode/skills/wikidata-brief-enrich/SKILL.md`)
that turns the endpoint above into an entity-extraction + triple-retrieval
workflow for free-text documents — the natural next step after the macronx
News Analysis workflow has produced an EIB (see [What this powers](#what-this-powers)):

- **Trigger**: paste any briefing, news article, or report and ask to "extract
  entities", "retrieve triples", or "enrich this brief". The skill also fires on
  keywords like `wikidata`, `entity resolution`, `enrich this report`.
- **What it does**: resolves each named entity to a Wikidata QID via
  `rdfs:label` + `wdt:P31` disambiguation, retrieves relevant outbound triples
  (capitals, membership, heads of state, founders, inception dates, listings,
  …) plus key inbound sets (member rosters, conflict participants, basin
  countries), and emits the original text verbatim with `QID`-tagged mentions
  and `▸ Wikidata grounding:` lines under each claim.
- **Requirement**: the server must be running (`bash server.sh status` — expect
  the `qlever.server.wikidata-truthy` container to be `Up`), otherwise the
  subagent's queries will fail. Verify with the count query from the
  [Data semantics](#data-semantics) section.
- **Architecture**: extraction is delegated to a `general` subagent so the main
  session context stays lean; the subagent returns a compact payload
  (entities / outbound / inbound / bridging notes) that the main agent formats.
- **Data-driven by design**: no hardcoded per-brief QID lookup tables — the
  skill encodes *resolution principles* (label ≠ alias, events vs persistent
  entities, multi-candidate tiebreak via `P571`/`P112`/`P159`/`P17`/`P414`,
  `@en`/`@mul` dedupe) that recompute against the archive on every run.
- **Caveats to expect**: aliases are not indexed (e.g. "China" resolves as
  "People's Republic of China"), current/news events rarely resolve (look for
  the persistent underlying entity instead), and oil benchmarks like "Brent
  crude" map to their namesake field rather than a dedicated price entity.

After editing the skill (or any config), restart opencode so it re-scans
`.opencode/skills/`.

## Tuning notes

Adjust for the machine in `server.sh` / `index.sh` / `Qleverfile`:

| Parameter                      | Reference | Meaning                                         |
| ------------------------------ | --------- | ----------------------------------------------- |
| `lbzcat -n 4` / `-j 8`         | 4 / 8     | parallel decompress / server threads            |
| `-m 8G` (MEMORY_FOR_QUERIES)   | 8G        | per-query memory (4G fails on 100M-row DISTINCT; 8G succeeds) |
| `-c 3G` (CACHE_MAX_SIZE)       | 3G        | server cache                                    |
| `STXXL_MEMORY`                 | 6G        | external-memory budget during indexing          |
| `VOCABULARY_TYPE`              | on-disk-compressed | keeps indexing RAM-lean (~6 GB peak on the reference machine) |

### Scaling down (low-end / laptop hosts)

The defaults above come from the development machine. On a smaller host the
biggest levers are per-query memory (`-m`), cache (`-c` / `-e`), and
parallelism (`-j`, `lbzcat -n`). The archive itself (~132 GB) is the same
either way — RAM and index speed are what scale:

| Host budget                | `-m` query | `-c` / `-e` cache | `-j` / `lbzcat` | STXXL |
| -------------------------- | ---------- | ----------------- | --------------- | ----- |
| Reference (this README)    | 8G         | 3G / 1G           | 8 / 4           | 6G    |
| 16 GB laptop / mini-PC     | 4G         | 2G / 512M         | 4 / 2           | 4G    |
| 8 GB host (works, slowest) | 2G         | 1G / 256M         | 2 / 1           | 2G    |

Keep `VOCABULARY_TYPE = on-disk-compressed` — it is what keeps indexing
RAM-lean on small machines.

Expected sizes (reference machine):

| Metric                 | Full multilingual | English-only (this repo) |
| ---------------------- | ----------------- | ------------------------ |
| Triples                | 8,474,704,545     | 2,926,237,465            |
| `rdfs:label` @en/@mul  | -                 | 93.0M / 20.4M            |
| non-en/mul labels      | billions          | 0                        |
| Data dir size          | ~227 GB           | ~132 GB                  |
| Build time             | ~3 h              | ~1.5-2 h                 |

## Data semantics

- The **Truthy** dump contains best-rank statements only: no qualifiers, no
  references. Regular languages like `en` co-exist with the `mul` tag used for
  canonical, language-neutral labels.
- The English filter keeps `en` and `mul` language-tagged literals; everything
  else with a language tag is dropped. This is ~62% smaller than the full
  multilingual truthy build and returns English results by default.
- Total triples: `SELECT (COUNT(*) AS ?c) WHERE { ?s ?p ?o }` -> `2926237465`.

## Reinstall / recreate

An agent (or a fresh machine) can rebuild everything by running this README
top-to-bottom: install the CLI, run the olympics smoke test, then create
`wikidata-truthy/` with the Qleverfile, `index.sh`, and `server.sh` above, and
run `bash index.sh` followed by `bash server.sh start`. The original dumps are
kept on disk, so a multilingual rebuild is possible by removing the `awk`
filter and re-adding the lexemes input.

