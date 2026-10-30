# ike-dex-source

Source documentation for DeX records. A converted source document is decomposed into a **topic
library**, indexed, and reassembled by **assemblies**, following the standards in
[`IKE-Network/ike-tooling`](https://github.com/IKE-Network/ike-tooling)
(`ike-build-standards/src/main/standards/`). Nearest worked example:
[`ikmdev/ike-dex-docs`](https://github.com/ikmdev/ike-dex-docs) (its whole tree is this repo's `sources/`).

Source documents are ingested into the topic library in `sources/`. Each device then gets an assembly in
`devices/` that rebuilds its source documents, beside the DeX records authored for it. The Tinkar changesets
for the DeX records live at the repository root.

## Layout

```
ike-dex-source/
├── pom.xml                      main pom: root="true", lists sources/ and devices/; ships one changeset + manifest per command (manual)
├── .mvn/, mvnw, mvnw.cmd        Maven 4 wrapper and config (found from every subdirectory)
├── .gitignore
├── src/
│   └── main/resources/         IVDs/{category}/{gmdn-term}/{primary-DI}/{name}-ike-cs.zip + {name}-manifest.mf (GITIGNORED; force-add on purpose)
├── sources/                     ingest tree; standalone pom, NOT a child of the main pom
│   ├── pom.xml                  lists topics and assemblies
│   ├── <source document, utilities/, work/>   ingest material (conversion scripts, scratch)
│   ├── topics/                  topic library; artifactId "topics" (fixed by standard)
│   │   └── src/docs/asciidoc/{index.adoc, topics/{domain}/{slug}.adoc, images/{domain}/}
│   └── assemblies/
│       ├── pom.xml              intermediate aggregator: the list of assemblies
│       ├── compendium/          all-topics validation assembly (required)
│       └── {assembly-name}/     pom.xml, src/docs/asciidoc/{assembly-name}.adoc
└── devices/                     per-device assemblies + authored DeX records; standalone pom; see devices/README.md
    ├── pom.xml                  the list of devices (path from this file)
    ├── _template/               copy to start a device (not built)
    └── IVDs/{category}/{gmdn-term}/{primary-DI}/   pom.xml, linked-devices.adoc, and in src/docs/asciidoc/:
                                                    DeXRecord_{primary-DI}.adoc + one assembly per source document
```

| Req | How it is met |
| --- | --- |
| R1 | `topics/` and the compendium are inside `sources/`; assemblies are siblings in `sources/assemblies/`. **Changed by decision:** per-device assemblies live in `devices/`, not in `sources/`. |
| R2 | `sources/assemblies/pom.xml` lists assemblies and `devices/pom.xml` lists devices. **Add each new assembly or device to its `<subprojects>` on every ingest**; one left out is silently not built. |
| R3 | `.asciidoctorconfig` is not ignored. The build writes one into each module with `src/docs/asciidoc/`; commit them. |
| R4 | `src/main/resources/` is gitignored. Add a changeset on purpose with `git add -f src/main/resources/<file>`. This protects untracked files only. |
| R5 | Shipping is manual, one record per command; see [Ship a record](#ship-a-record). `{name}-ike-cs.zip` and `{name}-manifest.mf`, in the folder under `src/main/resources/` that mirrors the device's folder in `devices/`, ship as one artifact `{groupId}:{name}:{version}` with classifiers `ike-cs` and `manifest`. Nothing else in the folder is touched. |
| R6 | Ingest and topics sit under `sources/`; per-device assemblies sit under `devices/`; there are no other top-level modules. |
| R7 | `sources/pom.xml` and `devices/pom.xml` have `ike-parent` as parent, not the main pom, so they inherit none of the shipping setup. Both set `maven.deploy.skip=true`; install stays on (see Build). |
| R8 | Every Maven command works from the root, `sources/`, `sources/topics`, `sources/assemblies/<x>`, `devices/` and a device folder (command forms below; the last two need `topics` installed first). |

## Setup

Maven 4 (`./mvnw` pins 4.0.0-rc-7) and a JDK; Prince 17 is optional (PDF only). Parent is
`network.ike.platform:ike-parent:204`; older parents pin an `ike-doc-maven-plugin` without
`idoc:topic-registry`.

Run `mvn validate` **at the repository root** first. It unpacks the standards into
`.claude/standards/` of every module (git-ignored). Read `IKE-INGEST`, `IKE-TOPIC-DECOMPOSITION`,
`IKE-ASCIIDOC-FRAGMENT`, `IKE-TOPIC-REGISTRY`, `IKE-ASSEMBLY`, `IKE-INDEX`, `IKE-DOC` before ingesting;
`IKE-DEX-INGEST` is missing from older unpacked sets, so read it from `ike-tooling`.

## Ingest

Source documents for a device (GUDID record, 510(k) decision summary, package insert) are ingested here
into the topic library. The device's own folder under `devices/` gets an assembly that rebuilds them and
holds the DeX records you author; its [README](devices/README.md) has the folder rules and workflow.

1. **Gate: 510(k) check** (`IKE-INGEST` Step 1). A first page with `510(k) SUBSTANTIAL EQUIVALENCE
   DETERMINATION`, `DECISION SUMMARY` and `A. 510(k) Number: K######` is a DeX record: do **not**
   decompose it; ingest it whole per `IKE-DEX-INGEST` (`topics/dex/`, id `dex-k######`; the supplied
   converted file is `510kDecisionSum_K######.adoc`). Whole-document GUDID records (`gudid-{DI}`, in
   `topics/gudid/`) are handled the same way. Anything else continues below.
2. **Classify** the source (authored, internal, regulatory, standards, literature) and confirm the
   handling with the user. Third-party sources need provenance attributes and `ext-` ids.
3. **Convert** to AsciiDoc into `sources/work/`, then **normalize** line breaks (below).
4. **Decompose** into fragments: header comments, `:topic-*:` attributes, literal `[[id]]` anchor,
   `= Title`, 500-5000 characters, 3-10 index terms, summary of 150-400 characters.
5. **Place** under `sources/topics/src/docs/asciidoc/topics/{domain}/` and add each topic to `index.adoc`.
6. **Register**: run the registry goal and fix every finding until it reports `0 findings`.
7. **Assemble**: add the new non-`ext/` topics to the compendium. For a device, create its folder from
   `devices/_template/`, make one assembly file per ingested source document (named as the DeX record's
   `:src-*:` attributes expect), and **add the device to `devices/pom.xml`**. Other assemblies: clone
   `compendium/` into `sources/assemblies/{name}/` and **add it to `sources/assemblies/pom.xml`**.
8. **DeX record**: place the hand-authored `DeXRecord_{primary-DI}.adoc` unchanged in the device's
   `src/docs/asciidoc/` (see `devices/README.md`). It is never ingested into `sources/topics/`.
9. **Manifest**: write `{name}-manifest.mf` by hand beside `{name}-ike-cs.zip` in
   `src/main/resources/IVDs/{category}/{gmdn-term}/{primary-DI}/` (the device's path under `devices/`)
   (DeX record ids, target devices, provenance, copyright). Nothing generates or validates it.
10. **Validate** and re-normalize any hand-edited file.

The registry is generated from topic headers: never write or commit its YAML. Topic ids are immutable
once committed. Assemblies own `:doctype:`, `:toc:` and `:sectnums:`; include topics with
`include::{topics}/topics/{domain}/{slug}.adoc[leveloffset=N]` where `:topics: {generated}/topics-asciidoc`.
`ext/` topics may be included in device (reconstruction) assemblies; the standard forbids it elsewhere.

## Commands

Run from the directory shown; `sources/` forms work inside `sources/`, the root forms from the root.

| Task | Inside `sources/` | From the root |
| --- | --- | --- |
| Registry | `mvn -B idoc:topic-registry -pl topics` | `mvn -B idoc:topic-registry -pl sources/topics` |
| Add file to registry | `... -pl topics -Dike.topic-registry.add=src/docs/asciidoc/topics/{domain}/{topic}.adoc` | same with `-pl sources/topics` |
| Line breaks | `mvn -N slb:reformat "-Dfile=topics/src/docs/asciidoc/topics/{domain}/"` | `mvn -N slb:reformat "-Dfile=sources/topics/src/docs/asciidoc/topics/{domain}/"` |
| Preview line breaks | add `-DdryRun=true` | add `-DdryRun=true` |
| Full build | `mvn clean verify` | `mvn clean verify` |
| One assembly | `mvn clean verify -pl assemblies/compendium -am` | `mvn clean verify -pl sources/assemblies/compendium -am` |
| One device | not in this reactor | `mvn clean verify -pl devices/IVDs/{category}/{gmdn-term}/{primary-DI} -am` |

Inside `sources/topics` use `mvn -B idoc:topic-registry` (no `-pl`). The `-am` forms above need no install.
Building from `devices/` (`mvn verify -pl IVDs/.../{primary-DI}`) or inside a device folder works only after
`mvn install -pl topics` has been run in `sources/`: without it, `topics:zip:adoc` is unresolved (tested).
A leftover `sources/topics/target/` can hide this, so do not rely on it.

`slb:reformat` takes a file or a directory (recursive), plugin `network.ike.docs:semantic-linebreak`.
Clause breaks are on (`-DclauseBreak=false` or `-DsentencesOnly=true` to reduce), lines soft-wrap at 64,
only paragraphs are touched, and **stdin is not read**. Commit a reflow on its own. A second dry run
reporting `0 of N file(s) had changes` means the files are normalized.

## Ship a record

A record is two hand-made files with the same prefix, in a folder under `src/main/resources/` that mirrors the
device's folder in `devices/` (the worked example is device 07332940008581):

```
src/main/resources/IVDs/infectious-disease/sars-cov-2-nucleic-acid-ivd-kit-nucleic-acid-technique-nat/07332940008581/
├── XpertXpressCov2Plus-07332940008581-DeXRecord-ike-cs.zip     the protobuf changeset
└── XpertXpressCov2Plus-07332940008581-DeXRecord-manifest.mf    its manifest, written by hand
```

```bash
mvn deploy -pl . -Ddex.record=XpertXpressCov2Plus-07332940008581-DeXRecord -Ddex.record.dir=IVDs/infectious-disease/sars-cov-2-nucleic-acid-ivd-kit-nucleic-acid-technique-nat/07332940008581
```

| Rule | Detail |
| --- | --- |
| Value | `-Ddex.record` is the file prefix only: letters, digits, `.`, `_`, `-`; no path, no `..`. `-Ddex.record.dir` is the folder under `src/main/resources/` (default `.`): relative, no `..`, no leading slash. |
| Activation | `-Ddex.record=<name>` activates profile `dex-record`. Without it the root installs and deploys nothing (`maven.install.skip` and `maven.deploy.skip` stay `true`). |
| Artifact | `dev.ikm.dex.source:<name>:<project version>` (group id is a placeholder; the prefix is the artifactId, so keep it unique across records): `-ike-cs.zip` (classifier `ike-cs`, type `zip`), `-manifest.mf` (classifier `manifest`, type `mf`) and a generated pom. |
| Fails when | Either file is missing or the name is malformed. `maven-enforcer-plugin` checks this in `validate`, before anything is installed or deployed. |
| Repository | The version is the root pom's (`1-SNAPSHOT`), so the default is `ike-snapshots` from `ike-parent`. For a release version add `-Ddex.repo.id=ike-staging -Ddex.repo.url=https://nexus.tinkar.org/repository/ike-staging/`. Credentials come from `settings.xml` under that id. |
| Local only | `mvn install -pl . -Ddex.record=<name> -Ddex.record.dir=<dir>` installs without deploying. `deploy` installs first, as usual. |
| Scope | Use `-pl .` so only the root ships. `sources/` and `devices/` never deploy (`maven.deploy.skip=true`). |

The root has no Java and no scripting: `maven-enforcer-plugin` validates, `maven-install-plugin:install-file`
and `maven-deploy-plugin:deploy-file` do the rest (`deploy-file` pushes both files in one deployment). One
record per command: repeat the command for more. A release version cannot be redeployed to a release
repository, so a record changes only when the project version does.

## Build and pass criteria

- `mvn validate` alone fails on a fresh clone: assemblies need the topic library's `adoc` ZIP, which exists
  only after `package`. Start with `mvn clean verify`. Add `-Dike.skip.prince=false` for the PDF.
- The main pom needs no `src/main/resources/` to build (fresh clones have none); with no `-Ddex.record`
  it ships nothing.
- **Passing:** `BUILD SUCCESS`; no unresolved `include::` and no "possible invalid reference"; registry
  `0 findings` with every new topic at the path where it was placed; every topic in `index.adoc` and the
  compendium; every assembly listed in `sources/assemblies/pom.xml` and every device in `devices/pom.xml`, each present in the reactor output.
- The registry does not check summaries, `related` bidirectionality, size bounds, index density, or that
  published topics appear in an assembly. Check those by hand.
- Every pom that inherits `ike-parent` must declare `<distributionManagement><site>`; the main pom,
  `sources/pom.xml` and `devices/pom.xml` do (placeholders; nothing is published to a site).

## Manifest

`{name}-manifest.mf` is written by hand, lives beside its changeset in the device's folder in the gitignored `src/main/resources/`
(add it with `git add -f`, like the changeset) and is installed and deployed as the `manifest` classifier. No
IKE standard defines its content (DeX record ids, target devices, provenance, copyright) and the build does
not read or check it. It is unrelated to the Tinkar changeset's internal `META-INF/MANIFEST.MF`.

## Open decisions

- `ext/` topics in device assemblies: the standard forbids them in assemblies; the examples allow them for
  per-source reconstruction, and device assemblies are reconstructions, so they are allowed here. Confirm.
- One assembly per device (all of its source documents as chapters) versus one per source document.
  Per document would need unique ids such as `{primary-DI}-package-insert`, and the directory must match.
- The local `IKE-DEX-INGEST` (unpacked from `ike-tooling`) names the verbatim 510(k) file
  `DeXRecord_K######.adoc`; the ingested file is `510kDecisionSum_K230440.adoc` and cites an `IKE-GUDID-INGEST`
  that is not in the unpacked standards. The files were placed as supplied; check them against the newer standard.
- Authored DeX records live in `devices/` as `DeXRecord_{primary-DI}.adoc`, are not topics and are not seen by
  the registry. Their format is not defined by any IKE standard.
- The Cepheid IFU is proprietary and is reproduced in full by decision (`:topic-license:` says so). Confirm
  before this repository is shared beyond the team.
- Whether `sources/` and `devices/` artifacts are ever released: `ike:release-publish` assumes one reactor
  version, and the main pom, `sources/pom.xml` and `devices/pom.xml` each carry `1-SNAPSHOT`. Keep them equal.
- The content and format of `{name}-manifest.mf`, and whether the build should ever validate it.
- Deploy to Nexus is tested against a `file://` repository only. A real push needs `ike-snapshots` or
  `ike-staging` credentials in `settings.xml`. A GitHub `workflow_dispatch` wrapper is not built.
- Group id `dev.ikm.dex.source` and the site URLs are placeholders.
- `leveloffset` guidance differs between the standards (`+1` versus `+2` under a `==` chapter); use the
  topic's depth in the source, as `ike-dex-docs` does.
