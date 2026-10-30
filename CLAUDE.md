# ike-dex-source: Claude instructions

[README.md](README.md) is the source of truth for layout, ingest steps, commands and pass criteria.

## First

Run `mvn validate` at the repository root; it unpacks the IKE standards into `.claude/standards/` (never
edit or commit). Read the standards listed in the README before ingesting. Run the 510(k) check before
decomposing anything.

## Easy to get wrong

- Source documents are ingested into `sources/`; per-device assemblies and authored DeX records live in
  `devices/`. `sources/pom.xml` and `devices/pom.xml` must never name the main pom as their parent and must
  not set `root="true"`.
- Device folders are `devices/IVDs/{category}/{gmdn-term}/{primary-DI}/`: a Maven module (copy
  `devices/_template/`, rename `pom.xml.template`) plus `src/docs/asciidoc/` holding `DeXRecord_{primary-DI}.adoc` (hand-authored, copied in unchanged, never
  ingested into `sources/topics/`) and one assembly file per ingested source document, named as the record's
  `:src-*:` attributes expect. Run the 510(k) detection gate first; ask before replacing an existing
  `dex-k######`. Never name the template pom `pom.xml`.
- Every new assembly goes in `sources/assemblies/pom.xml` `<subprojects>`; every new device in
  `devices/pom.xml`; an unlisted one is silently not built.
- Never write or commit registry YAML. Run `idoc:topic-registry` until `0 findings`.
- Never ignore `.asciidoctorconfig`; commit the generated ones.
- `src/main/resources/` is gitignored. Never `git add -f` a changeset unless asked.
- The main pom ships nothing unless a person passes `-Ddex.record=<name> -Ddex.record.dir=<dir>`: it then
  installs and deploys `{name}-ike-cs.zip` and `{name}-manifest.mf` from
  `src/main/resources/<dir>/` (`<dir>` mirrors the device's path under `devices/`) as one artifact with classifiers
  `ike-cs` and `manifest`. The manifest is hand-written; never generate one, never sweep up other files in
  that folder, never remove the `maven.install.skip`/`maven.deploy.skip` defaults, and add no Java to the root.
- The main pom stays `pom` packaging. Don't `sed`/regex-edit POMs. Don't commit or push unless asked.
- Use `-pl topics` inside `sources/`, `-pl sources/topics` at the root.
