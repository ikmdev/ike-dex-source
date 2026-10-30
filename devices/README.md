# devices

One folder per device. Each device's source documents (GUDID record, 510(k) decision summary, package
insert) are ingested into the topic library in `../sources/`; the device folder holds an **assembly that
rebuilds them** and the **DeX records authored for the device**. Adapted from
[`MahaKaleem24/test`](https://github.com/MahaKaleem24/test); differences are at the end. The repository
README covers the ingest workflow, builds and shipping.

## Layout

```
devices/
├── pom.xml                            standalone aggregator: lists every device (path from this file)
├── _template/                         copy to start a device; not built
└── IVDs/                              device group; non-IVD groups sit beside it later
    └── {category}/                    e.g. chemistries, infectious-disease, at-home-tests, other
        └── {gmdn-term}/               GMDN term name, slugged (below)
            └── {primary-DI}/          one Maven module per device
                ├── pom.xml
                ├── src/docs/asciidoc/{primary-DI}.adoc    assembly of the device's ingested source documents
                ├── DeXRecord_K######.adoc                 authored DeX record(s)
                └── linked-devices.adoc                    related devices
```

| Level | Folder | Rule |
| --- | --- | --- |
| 1 | `IVDs/` | The only name with capitals. |
| 2 | `{category}/` | Lowercase, hyphens, no spaces. `other` is the catch-all. |
| 3 | `{gmdn-term}/` | The device's GMDN term name, slugged: lowercase, every run of non-alphanumerics becomes one hyphen, no leading or trailing hyphen. `Troponin I IVD, reagent` becomes `troponin-i-ivd-reagent`. |
| 4 | `{primary-DI}/` | The Primary DI exactly as issued (digits, keep leading zeros). Unique per device. |

Category and GMDN folders are created on demand (git does not track empty folders) and need no pom: the
aggregator lists devices by their full path. Find a device by DI:
`find devices/IVDs -mindepth 3 -maxdepth 3 -type d -name <DI>`.

## The device assembly

A reconstruction assembly: `src/docs/asciidoc/{primary-DI}.adoc` rebuilds the device's source documents from
the topic library, one chapter per source document, each topic included in source order with
`include::{topics}/topics/ext/{domain}/{slug}.adoc[leveloffset=+2]`. It is built exactly like any assembly
(`IKE-ASSEMBLY`): `{primary-DI}` is identical in the directory name, `<artifactId>`, the `.adoc` file name and
`pdf.source.document`; the pom depends on `topics:adoc:zip`; it has `devices/pom.xml` as parent through
`<relativePath>../../../../pom.xml</relativePath>` (exactly four levels down).

Reconstruction assemblies may include `ext/` topics (the standard forbids this for other assemblies).

## Linking devices

Devices that work together (an assay and its instruments, calibrators, controls, accessories) each have a
folder, possibly under different GMDN folders or categories. Relationships live in each device's
`linked-devices.adoc`: a heading per relationship type, then one bullet per device with the DI, the name and
an `xref:` to the other device's `linked-devices.adoc`.

```asciidoc
= Linked Devices: 20816101029031

== Controls

* 20816101029048 — LIAISON NES® FLU A/B, RSV & COVID-19 Control Swab Kit
  xref:../../multiple-genus-respiratory-virus-nucleic-acid-ivd-control/20816101029048/linked-devices.adoc[Linked Devices]
```

The path is written from the device's own folder: `../{DI}/` for the same GMDN folder,
`../../{gmdn-term}/{DI}/` for another GMDN folder, `../../../{category}/{gmdn-term}/{DI}/` for another
category. Link **both ways**: when A lists B, B lists A (for example "Approved IVDs / Assays").

## Adding a device

1. Ingest each source document into `../sources/` (repository README, Ingest steps 1 to 6). Convert into
   `sources/work/` first; instrument and system user manuals are linked in the assembly, not reproduced.
2. Find the device's category, GMDN term name and Primary DI in its GUDID record.
3. `cp -R devices/_template devices/IVDs/{category}/{gmdn-term}/{primary-DI}`, then in the copy rename
   `pom.xml.template` to `pom.xml` and `src/docs/asciidoc/{primary-DI}.adoc` to the DI, and replace every
   `{placeholder}`. Delete files that do not apply.
4. **Add the device to `devices/pom.xml`** (`<subproject>IVDs/{category}/{gmdn-term}/{primary-DI}</subproject>`).
   One left out is silently not built.
5. Write the assembly's includes, then build it (below) until there is no unresolved `include::`.
6. Fill `linked-devices.adoc` here **and** in every device it names.
7. Author the DeX record (next section).

## DeX records

Authored records are stored beside the assembly as `DeXRecord_K######.adoc`, named for the 510(k) number,
with the `IKE-DEX-INGEST` header block (`:topic-id: dex-k######`, summary, related, notes, provenance,
license). `_template/DeXRecord_K######.adoc` has the header. The 510(k) number is in the GUDID record's
**FDA Premarket Submission** table and in section **A** of the decision summary.

- **One record per 510(k) number.** Before creating one, run `find devices -name 'DeXRecord_K######.adoc'`
  and ask before replacing an existing record. A 510(k) shared by several devices is one record.
- **Detection gate first** (`IKE-DEX-INGEST`): a verbatim, whole-document 510(k) ingest needs all three of
  `510(k) SUBSTANTIAL EQUIVALENCE DETERMINATION`, `DECISION SUMMARY` and `A. 510(k) Number: K######` on the
  first page. A multi-filing history does not pass; handle it under `IKE-INGEST` after the user confirms.
- **Two kinds of record.** A verbatim ingest of the FDA decision summary still lands in the topic library as
  `topics/dex/DeXRecord_K######.adoc` (id `dex-k######`), and a device assembly can include it. The record
  authored here is a separate document. If both exist for one K number they would share a file name and
  topic id: decide the authored record's name before creating the second.
- **Not in the topic library.** The standard puts verbatim records in `topics/dex/` and registers them; these
  live here, so the registry does not see them (`idoc:topic-registry` covers `sources/topics` only).
  The format of a record authored from several sources, and whether to register it, are open decisions in
  the repository README. A device with no 510(k) number has no K-based file name yet.
- The device's assembly may cite `:topic-related:` ids so the record and its source topics link.

## Build

| From | Command |
| --- | --- |
| Repository root | `mvn clean verify -pl devices/IVDs/{category}/{gmdn-term}/{primary-DI} -am` |
| `devices/` or the device folder | `mvn clean verify`, only after `mvn install -pl topics` was run in `sources/` |

Without the install, `topics:zip:adoc` is unresolved. Assemblies are built with the rest of the reactor by
a plain `mvn clean verify` at the root, which also builds `sources/`.

## Differences from the reference repository

- Source documents are ingested into the topic library and rebuilt as an assembly per device, instead of
  being kept as converted `gudid.adoc`, `510k-decision-summary.adoc` and `package-insert.adoc` files.
  Re-ingest the reference repo's files rather than copying them in: they predate the `IKE-DEX-INGEST` shape
  (no `DeXRecord_K######` title, no `:topic-*:` header, section titles without the source's trailing
  punctuation).
- Authored DeX records are stored in the device folder; `package-insert-NOTE.adoc` is replaced by a comment
  in the assembly saying what is absent and why.
- Standards come from `ike-tooling` (`IKE-Network`) through `mvn validate`, not a personal fork's branch.
- The `xref:` link convention is documented as the files use it, including the cross-category path, and the
  GMDN slug rule is written down.
