---
name: ingest-document
description: Ingest source documents for one device (GUDID record, 510(k) decision summary, package insert or IFU, other PDF or Word files) into the topic library and create the device's folder under devices/. Use when asked to ingest, import, convert or decompose a source document.
argument-hint: <paths to the source documents>
---

Ingest the source documents named in the arguments (`$ARGUMENTS`; ask for their paths if none were given) for one device into this repo: topics in `sources/topics`, then the device's folder under `devices/`. The worked example is device 07332940008581 (Cepheid Xpert Xpress CoV-2 plus): its topics and its module under `devices/`. DeX records, manifests and changesets come later; do not create them.

**Read first.** `CLAUDE.md`, `README.md` and `devices/README.md` (the source of truth). Get the IKE standards as the README's Setup describes, then read IKE-INGEST, IKE-TOPIC-DECOMPOSITION, IKE-ASCIIDOC-FRAGMENT, IKE-TOPIC-REGISTRY, IKE-ASSEMBLY and IKE-INDEX. IKE-DEX-INGEST is not unpacked: read it from IKE-Network/ike-tooling.

**Classify and confirm before converting.** Run the 510(k) gate (README, Ingest step 1). Classify each source. For third-party sources, stop and ask the user the IKE-INGEST confirmation: type, handling (decomposed, whole document, summary), quotation, `:topic-license:`. Never reproduce copyrighted text without the user's explicit decision.

**Converter prerequisite.** Check that `../ike-dex-docs` exists (a sibling of this repo) and has `utilities/.venv/bin/python`. If not, stop and warn the user: from this repo's parent directory run `git clone https://github.com/ikmdev/ike-dex-docs.git`, install Pandoc, then run `./utilities/setup.sh` inside it. Do not clone or install it yourself.

**Convert and verify.** `../ike-dex-docs/utilities/.venv/bin/python ../ike-dex-docs/utilities/document_to_asciidoc.py <src> -o sources/work/raw/<name>.adoc` (`work/` is gitignored). For a PDF, run `check_fidelity.py` from the same folder with the same Python: every gate must pass; also compare page images, since it trusts the PDF's tags. Report converter defects to the user; don't hand-patch.

**Normalize early and often.** From `sources/`: `mvn -B -N slb:reformat "-Dfile=work/raw/<name>.adoc"` (`-DdryRun=true` first). Run it right after conversion, after the header and any hand edit, and on the placed file. Compare words, ignoring whitespace, before and after each run. A last dry run must report 0 changes.

**Topics** (`sources/topics/src/docs/asciidoc/`). Place under `topics/ext/{type}/`, `topics/gudid/` or `topics/dex/` (whole-document `gudid-{DI}` and `dex-k######` records). Header per IKE-ASCIIDOC-FRAGMENT, `ext-` ids for external sources, status at most `review`, no document-level attributes, images in `images/{domain}/{doc-slug}/` relative to imagesdir. Add 3-10 `indexterm:[…]` (5-15 for DeX records). Add each topic to `index.adoc`, non-`ext/` topics to `assemblies/compendium`, and `:topic-related:` ids both ways. `mvn -B idoc:topic-registry -pl topics` (from `sources/`) must report `0 findings`.

**Device folder.** From the GUDID record take the Primary DI, GMDN term and device type. Copy `devices/_template/` (omit its DeX record skeleton) to `devices/{device-group}/{category}/{gmdn-term-slug}/{primary-DI}/`: `IVDs` for in-vitro diagnostics; for other device types follow `devices/README.md` and ask the user which group folder to use. Rename `pom.xml.template` and fill the placeholders. Add one assembly file per ingested document in `src/docs/asciidoc/`, named like the topic file it includes (e.g. `GUDIDRecord_{DI}.adoc`), with device metadata and `include::…[leveloffset=+1]` (`ext/` allowed). Leave `pdf.source.document` as the template sets it, and do not add the device to `devices/pom.xml` yet: both wait until the DeX record exists.

**Done means** root `mvn clean verify` succeeds with no unresolved include, missing image or invalid-reference warning; registry `0 findings`; every `include::` in the device's assemblies points at an existing topic; no source content lost.

**Constraints.** Don't commit or push unless the user asks (then end messages with `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`). Don't deploy. Don't `sed`/`awk` POMs. Finish with files created, check results, deviations, questions.
