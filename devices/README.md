# devices

One folder per device. Each device's source documents (GUDID record, 510(k) decision summary, package
insert) are ingested into the topic library in `../sources/`; the device folder holds a **reconstruction
assembly of each source document** and the device's **authored DeX record**, side by side. Adapted from
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
                ├── linked-devices.adoc                      related devices (optional)
                └── src/docs/asciidoc/
                    ├── DeXRecord_{primary-DI}.adoc          authored DeX record, never a topic
                    ├── GUDIDRecord_{primary-DI}.adoc        reconstruction of the GUDID topic
                    ├── 510kDecisionSum_{K-number}.adoc      reconstruction of the 510(k) topic
                    └── {slug}/{slug}-ifu.adoc               reconstruction of the IFU topic
```

The worked example is
`IVDs/infectious-disease/sars-cov-2-nucleic-acid-ivd-kit-nucleic-acid-technique-nat/07332940008581/`
(Cepheid Xpert Xpress CoV-2 plus).

| Level | Folder | Rule |
| --- | --- | --- |
| 1 | `IVDs/` | The only name with capitals. |
| 2 | `{category}/` | Lowercase, hyphens, no spaces. `other` is the catch-all. |
| 3 | `{gmdn-term}/` | The device's GMDN term name, slugged: lowercase, every run of non-alphanumerics becomes one hyphen, no leading or trailing hyphen. `SARS-CoV-2 nucleic acid IVD, kit, nucleic acid technique (NAT)` becomes `sars-cov-2-nucleic-acid-ivd-kit-nucleic-acid-technique-nat`. |
| 4 | `{primary-DI}/` | The Primary DI exactly as issued (digits, keep leading zeros). Unique per device. |

Category and GMDN folders are created on demand (git does not track empty folders) and need no pom: the
aggregator lists devices by their full path. Find a device by DI:
`find devices/IVDs -mindepth 3 -maxdepth 3 -type d -name <DI>`.

## Source-document assemblies

Each ingested source document gets a thin assembly file in the device's `src/docs/asciidoc/`: a title, the
device metadata, and one `include::{topics}/topics/{domain}/{slug}.adoc[leveloffset=+1]`. Its **name must be the
name the DeX record uses** in its `:src-gudid:`, `:src-510k:` and `:src-ifu:` attributes (a subfolder is
fine), so the record's `xref:` links resolve in the HTML output unchanged. Metadata attributes
(`:device-primary-di:`, `:device-name:`, `:device-labeler:`, `:device-gmdn-term:`, `:document-type:`,
`:source-topic:`) identify the device and the topic it rebuilds. The topic's own numbering is kept, so
`:sectnums:` is not set.

`ext/` topics may be included here (the standard forbids it in other assemblies), because this is a
reconstruction. The pom depends on `topics:adoc:zip` and has `devices/pom.xml` as its parent through
`<relativePath>../../../../pom.xml</relativePath>` (exactly four levels down). `pdf.source.document` names
the DeX record, since a device module holds several documents.

`devices/pom.xml` sets the asciidoctor plugin's `preserveDirectories`, so the IFU stays in its subfolder in
the HTML. The single-file HTML (`html-single/{primary-DI}.html`) is the DeX record with its images embedded;
its links to the other documents need the multi-file `html` output.

## DeX records

`DeXRecord_{primary-DI}.adoc` is **authored by hand and is not a topic**: it has no `:topic-*:` header, is
never placed in `sources/topics/`, and the registry does not see it. It lives in `src/docs/asciidoc/` so the
build renders it beside the source-document assemblies. One record per device. Its Source column links to the
source assemblies (`xref:{src-gudid}#gudid-{primary-DI}[GUDID]`); anchors are the topic ids, so a source
topic's id must not change once a record links to it.

A verbatim 510(k) decision summary is a different thing: it is ingested as a topic (`dex-k######`, in
`topics/dex/`, per `IKE-DEX-INGEST`) and rebuilt here by an assembly. The format of the authored record is
not defined by any IKE standard; copy a finished one. A device with no 510(k) number simply has no
`510kDecisionSum` file.

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

1. **Ingest** each source document into `../sources/` (repository README, Ingest steps 1 to 6), including
   `slb:reformat`, the registry run and the `index.adoc` entry. Put a document's images in
   `sources/topics/src/docs/asciidoc/images/{domain}/{doc}/` and reference them relative to `imagesdir`.
2. Find the device's category, GMDN term name and Primary DI in its GUDID record.
3. `cp -R devices/_template devices/IVDs/{category}/{gmdn-term}/{primary-DI}`, then in the copy rename
   `pom.xml.template` to `pom.xml` and replace every `{placeholder}`. Make one source-document assembly per
   ingested document from `{SourceDocName}.adoc`, named as the DeX record expects.
4. **Add the device to `devices/pom.xml`** (`<subproject>IVDs/{category}/{gmdn-term}/{primary-DI}</subproject>`).
   One left out is silently not built.
5. Put the DeX record in `src/docs/asciidoc/`, unchanged, and build (below).
6. Fill `linked-devices.adoc` here **and** in every device it names.

## DeX record changeset

The Tinkar (protobuf) form of the record is not kept here. It lives in the repository's gitignored
`src/main/resources/`, in a folder that mirrors this device's path
(`IVDs/{category}/{gmdn-term}/{primary-DI}/`), as `{name}-ike-cs.zip` with its hand-written
`{name}-manifest.mf`. The repository README's "Ship a record" section covers installing and deploying the pair.

## Build

| From | Command |
| --- | --- |
| Repository root | `mvn clean verify -pl devices/IVDs/{category}/{gmdn-term}/{primary-DI} -am` |
| `devices/` or the device folder | `mvn clean verify`, only after `mvn install -pl topics` was run in `sources/` |

Without the install, `topics:zip:adoc` is unresolved. A plain `mvn clean verify` at the root builds
everything. Outputs are in the device's `target/generated-docs/html/` (one HTML per document) and
`html-single/`. Check that every `xref:` in the record resolves: the file exists and the anchor id is in it.

## Differences from the reference repository

- Source documents are ingested into the topic library and rebuilt by assemblies per device, instead of being
  kept as converted `gudid.adoc`, `510k-decision-summary.adoc` and `package-insert.adoc` files.
- The authored DeX record is stored beside those assemblies; `package-insert-NOTE.adoc` is replaced by a
  comment in the assembly saying what is absent and why.
- Standards come from `ike-tooling` (`IKE-Network`) through `mvn validate`, not a personal fork's branch.
- The `xref:` link convention is documented as the files use it, including the cross-category path, and the
  GMDN slug rule is written down.
