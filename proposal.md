# Build a Modular Website Mirroring and Text-Corpus Crawler

## Objective

Rebuild a lost Python project from scratch.

The project is a **generic, modular website mirroring and text extraction pipeline** whose first source adapter targets:

`http://channeling.safo.cz/`

The immediate purpose is to collect Czech-language articles from that site into a durable local corpus that can later be used for LLM experimentation.

The architecture must **not be coupled to channeling.safo.cz**.

Future sources may include:

- GOV.UK pages
- technical documentation such as GTK4 documentation
- static blogs
- documentation sites
- knowledge bases
- other mostly-static websites

The system should therefore separate:

1. source discovery
2. URL inventory
3. HTTP/browser acquisition
4. raw artifact storage
5. content extraction
6. normalization
7. source reconciliation
8. corpus generation

Do not implement everything as one crawler class or script.

---

## Primary Design Principle

The crawler is a **source mirror**, not merely a scraper.

A source has an externally observable state.

The local system maintains its own state representing the latest known state of that source.

Each synchronization run should answer:

- What pages exist at the source?
- Which pages are already known locally?
- Which pages are missing locally?
- Which pages changed?
- Which pages disappeared from the source?
- Which pages are newly discovered?
- Which pages should be downloaded during this run?
- Which pages were skipped because of the configured limit?
- Which local pages should be moved to `bin/` because they disappeared upstream?
- What failed?
- What remains incomplete?

A dry-run must be able to answer these questions without downloading page bodies.

---

# Target Website

Inspect the real site before implementing the adapter:

`http://channeling.safo.cz/`

The current site is a simple static-style Czech blog/archive.

The front page exposes a large numbered article archive. Articles contain metadata such as:

- numeric article identifier
- title
- author
- category
- publication date
- article URL
- article content/excerpt
- possible external source/reference links

The archive currently contains thousands of articles.

Important:

Do not assume that an external "Source" link is the article's canonical content.

The crawler must distinguish:

- the page hosted by channeling.safo.cz
- external references linked from that page

The initial source adapter should mirror the target site's own articles.

External source pages must not automatically become corpus documents unless explicitly supported by a future adapter/configuration.

---

# Technology

Use:

- Python 3.12+
- Playwright for browser acquisition where required
- standard-library-first design
- typed Python
- pathlib
- dataclasses or another explicit typed domain model
- pytest

Avoid introducing a database during the initial implementation.

The filesystem is the initial persistence layer.

Do not introduce a heavyweight framework unless a concrete requirement justifies it.

---

# Required Repository Structure

Use a structure approximately equivalent to:

    src/
      crawler/
        domain/
        discovery/
        acquisition/
        extraction/
        normalization/
        storage/
        reconciliation/
        adapters/
        cli/

    tests/

    data/
      sources/
      raw/
      extracted/
      corpus/
      inventory/
      bin/
      reports/

    config/

    pyproject.toml

The exact structure may be adjusted if there is a strong architectural reason.

Keep domain concepts independent from the concrete source adapter.

---

# Domain Model

Define explicit typed models for at least:

## SourceDocument

Represents the identity and metadata of a remotely discoverable document.

Fields should include concepts equivalent to:

- stable document ID
- canonical URL
- source identifier
- title
- author
- category
- published date
- discovery timestamp
- source metadata
- optional content fingerprint
- optional remote modification information

Do not use the URL as the only identity if the source provides a more stable identifier.

For channeling.safo.cz, investigate whether the numeric article IDs can serve as the stable document identity.

---

## DiscoveredDocument

Represents the result of discovery.

It must contain enough information to decide whether acquisition is necessary without downloading the full page.

---

## RawDocument

Represents an acquired source response.

Preserve:

- requested URL
- final URL after redirects
- HTTP metadata where available
- retrieval timestamp
- raw HTML
- content hash
- acquisition method
- status/error information

Raw source data should be immutable.

If a page is downloaded again, do not silently overwrite historical raw data.

---

## ExtractedDocument

Represents semantic extraction from HTML.

It should contain:

- document identity
- metadata
- title
- author
- date
- category
- main textual content
- source URL
- extraction metadata
- extraction warnings

---

## CorpusDocument

Represents the normalized training-oriented representation.

This is deliberately separate from the extracted source.

The corpus format must not become the canonical storage format.

---

# Filesystem Persistence

Every persisted document should have a deterministic path.

Do not create filenames directly from arbitrary article titles.

Titles can contain:

- Unicode
- punctuation
- emojis
- slashes
- duplicate names
- changing text

Prefer stable IDs plus safe extensions.

Example concept:

    raw/channeling-safo/13605/<retrieval-id>.html
    extracted/channeling-safo/13605.json
    corpus/channeling-safo/13605.jsonl

The exact layout is up to the implementation.

Document identity must remain stable even if its title changes.

---

# Inventory

Maintain a machine-readable inventory.

JSON or JSONL is acceptable.

The inventory represents the last known remote discovery state.

It should allow the system to determine:

- known documents
- newly discovered documents
- missing documents
- previously failed documents
- downloaded documents
- extraction status
- content fingerprints
- last successful synchronization
- last observed timestamp

Do not use directory existence alone as the state machine.

---

# Discovery

Discovery is independent from downloading.

Implement a discovery interface similar to:

    discover(source_config) -> Iterable[DiscoveredDocument]

For channeling.safo.cz, create a dedicated adapter.

The adapter should inspect the actual site and determine how the archive is exposed.

Do not hard-code assumptions such as:

    /article/<id>

unless verified.

The front page currently exposes a large numbered archive and a "100 articles" navigation option. Investigate the actual HTML/link structure with Playwright and/or HTTP requests.

Discovery should be capable of:

- traversing archive pages
- extracting article links
- extracting article metadata
- detecting pagination
- detecting the end of the archive
- avoiding duplicate URLs
- canonicalizing URLs

Do not download every article body during discovery.

---

# Acquisition

Create an acquisition abstraction.

For example:

    class Acquirer(Protocol):
        async def acquire(url: str) -> RawResponse: ...

Provide at least:

1. lightweight HTTP acquisition
2. Playwright acquisition

Do not automatically use Playwright for everything.

Prefer ordinary HTTP when the source is static and the content can be reliably retrieved without browser execution.

Use Playwright when:

- JavaScript is required
- navigation requires browser behavior
- content is dynamically generated
- the adapter explicitly requires it

The target site should be tested to determine whether HTTP acquisition is sufficient.

---

# Extraction

Extraction must be source-specific but exposed through a generic interface.

Conceptually:

    extract(raw_document) -> ExtractedDocument

For channeling.safo.cz:

- identify the article title
- author
- category
- date
- article identifier
- main article text
- source links
- continuation/source indicators
- irrelevant navigation/UI elements

Do not use an LLM for extraction.

Use deterministic HTML parsing.

If CSS selectors are required, isolate them inside the source adapter.

Extraction should produce warnings when expected fields are missing.

For example:

    missing_author
    missing_date
    empty_content
    unexpected_layout
    external_continuation
    extraction_selector_failed

Do not silently produce apparently valid documents when extraction failed.

---

# Normalization

Normalization should be deterministic.

It may perform:

- whitespace normalization
- paragraph normalization
- Unicode normalization
- removal of boilerplate
- removal of navigation
- preservation of meaningful headings
- preservation of paragraph boundaries

Do not aggressively rewrite the source.

Do not summarize.

Do not translate.

Do not correct the author's language.

Do not use an LLM in the normalization pipeline.

The goal is preservation of source semantics.

---

# Corpus Format

The initial corpus should support LLM fine-tuning later.

Prefer JSONL with explicit metadata.

Conceptually:

    {
      "id": "...",
      "source": "...",
      "url": "...",
      "title": "...",
      "author": "...",
      "category": "...",
      "published": "...",
      "language": "cs",
      "text": "..."
    }

Do not yet optimize the corpus for a specific training framework.

A later transformation layer can produce:

- conversational datasets
- instruction datasets
- completion datasets
- chat templates
- tokenized datasets

The crawler should preserve source information independently of those formats.

---

# Synchronization Semantics

Implement a synchronization planner before implementing full downloading.

Given:

    remote_inventory
    local_inventory

produce a plan containing categories such as:

- new
- missing
- changed
- unchanged
- failed
- pending

The plan must be inspectable.

Example CLI:

    crawler sync channeling-safo --dry-run

Possible output:

    Remote documents: 13605
    Local documents: 120
    New: 13485
    Changed: 3
    Unchanged: 117
    Missing remotely: 2
    Failed previously: 1

Do not actually download anything in dry-run mode.

---

# Download Limits

The user must be able to limit work per run.

Examples:

    crawler sync channeling-safo --limit 10

    crawler sync channeling-safo --limit 10 --oldest-first

    crawler sync channeling-safo --limit 10 --newest-first

The limit applies to actual acquisition work, not discovery.

Discovery should determine the complete known remote inventory first.

The user must therefore be able to ask:

    crawler status channeling-safo

and obtain information such as:

    Remote:       13605
    Local:        1250
    Missing:      12355
    Pending:      12355
    Failed:       4
    Changed:      7

The exact numbers are examples only.

---

# Incremental Operation

The system must support repeated partial runs.

Example:

Day 1:

    sync --limit 10

Day 2:

    sync --limit 10

Day 3:

    sync --limit 10

The crawler must continue from the existing state rather than starting from scratch.

It must not redownload unchanged documents.

It must not assume that "first 10" means the same ten documents forever.

Use the synchronization state and deterministic selection policy.

---

# Gap Filling

A critical requirement:

The user may manually delete local documents.

The next synchronization must detect those gaps.

Example:

Remote:

    1
    2
    3
    4
    5

Local:

    1
    2
    4
    5

The crawler must report:

    Missing locally: 3

and allow:

    sync --limit 1

to restore article 3.

Do not model synchronization purely as "download anything newer than the newest local ID."

The system must support arbitrary holes.

---

# Remote Deletions

If a document exists locally but is no longer discovered remotely:

    remote: 1,2,4,5
    local:  1,2,3,4,5

article 3 must be classified as:

    missing_remote

Do not permanently delete it.

Move its current material into:

    data/bin/

or an equivalent quarantine structure.

Preserve metadata explaining:

- original ID
- source
- URL
- detection timestamp
- reason
- previous storage path

A report must explicitly state remote deletions.

Do not move a document to `bin/` merely because acquisition failed.

"Not found remotely" and "download failed" are different states.

---

# Reappearing Documents

If a previously missing remote document reappears:

    missing_remote -> discovered

the system must be able to restore/reacquire it.

Do not permanently mark an ID as deleted.

---

# Content Changes

A page may retain the same article ID but change its content.

Use a content fingerprint/hash after acquisition.

If metadata is unchanged but content differs:

    same identity
    changed content

The system must create a new raw artifact rather than destroying the old raw representation.

The extracted/corpus representation can then be updated according to a deterministic policy.

---

# Reports

Every synchronization should produce a machine-readable report.

Also provide human-readable CLI output.

Report at minimum:

- source
- started_at
- completed_at
- discovery count
- local count
- new count
- changed count
- unchanged count
- locally missing count
- remotely missing count
- downloaded count
- extraction success
- extraction failures
- acquisition failures
- skipped due to limit
- quarantined documents
- warnings

Failed documents must remain visible.

Never silently swallow errors.

---

# CLI

Design a small explicit CLI.

Initial commands:

    crawler sources
    crawler discover <source>
    crawler status <source>
    crawler sync <source>
    crawler report <source>
    crawler inspect <document-id>

Useful options:

    --dry-run
    --limit N
    --newest-first
    --oldest-first
    --retry-failed
    --force
    --verbose

Do not create dozens of flags before the underlying model is correct.

---

# Configuration

Source-specific behavior must live in configuration or adapters.

Conceptually:

    config/
      sources/
        channeling-safo.toml

The configuration can contain:

- source ID
- base URL
- language
- crawl policy
- acquisition method
- rate limit
- pagination settings
- extraction adapter
- canonicalization policy

Do not put source-specific selectors into global crawler logic.

---

# Politeness and Safety

The crawler must:

- respect robots.txt where applicable
- rate-limit requests
- avoid uncontrolled concurrency
- identify itself with a configurable User-Agent
- use timeouts
- retry transient failures with bounded backoff
- distinguish HTTP errors from extraction errors
- avoid crawling external domains accidentally

The target source may contain links to many external websites.

Those links must not cause unrestricted crawling.

The source adapter must explicitly define the allowed domain(s).

---

# Testing

Tests are mandatory.

At minimum provide fixtures for:

1. normal article
2. article without author
3. article with external source link
4. article with continuation
5. malformed/partial article
6. duplicate URL
7. changed article
8. locally missing article
9. remotely deleted article
10. reappearing article
11. pagination
12. download failure
13. extraction failure
14. download limit

Use saved HTML fixtures.

Do not make normal unit tests depend on the live website.

Provide a small integration test that can optionally target the real site.

---

# First Development Phase

Do not implement the entire project immediately.

First:

1. inspect the target website
2. inspect its HTML and navigation structure
3. determine article URL patterns
4. determine archive/pagination behavior
5. determine whether ordinary HTTP is sufficient
6. identify stable article IDs
7. identify article content boundaries
8. document findings
9. design domain models
10. implement inventory + discovery
11. implement dry-run status/reconciliation

Only after this phase should acquisition and extraction be implemented.

---

# Important Architectural Constraint

Do not optimize for the current website.

The target site is merely the first adapter.

The intended architecture is:

    Generic crawler engine
            +
    source adapter
            +
    acquisition strategy
            +
    extraction strategy
            =
    reusable website mirroring system

A future GOV.UK adapter should not require changes to the synchronization engine.

A future GTK documentation adapter should not require changes to filesystem reconciliation.

---

# Definition of Done for the First Milestone

The first milestone is complete when:

- the project runs locally
- the target site can be discovered
- all currently discoverable article IDs can be inventoried
- no article bodies need to be downloaded for discovery
- `status` reports remote/local/pending counts
- `sync --dry-run` produces a deterministic plan
- `--limit 10` limits acquisition work
- local gaps are detectable
- remote deletions are detectable
- source-specific logic is isolated in an adapter
- tests cover reconciliation behavior
- no LLM is required by the crawler itself
- no database is required
- the raw source can later be reconstructed independently of the normalized corpus

Before writing significant code, create a short `DESIGN.md` documenting the observed target-site structure and the proposed domain model.

Do not invent undocumented site behavior. Inspect it first.