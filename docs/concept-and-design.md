# PDS4 Data Dictionary Search — Operational Concept and Design Specification

**Repository**: https://github.com/NASA-PDS/dd-search  
**Status**: Draft  
**Date**: 2026-04-27  
**Related Issues**: [NASA-PDS/pds4-information-model#926](https://github.com/NASA-PDS/pds4-information-model/issues/926), [NASA-PDS/pds4-information-model#244](https://github.com/NASA-PDS/pds4-information-model/issues/244)

---

## 1. Background and Motivation

The PDS4 Information Model is documented via a Data Dictionary that describes every class, attribute, data type, and namespace in the standard. This dictionary is an essential reference for PDS data providers building compliant data products and for PDS operations staff validating them.

### 1.1 Current State

The current documentation pipeline is:

1. **LDDTool** generates a large monolithic DocBook XML file (e.g., `PDS4_PDS_DD_1Q00.xml`) containing all Common, Discipline, and Mission Local Data Dictionaries (LDDs).
2. An **Oxygen XML Editor** ANT build transforms the DocBook XML into a Classic WebHelp static HTML site.
3. The generated site is hosted at: https://pds.nasa.gov/datastandards/documents/dd/all/current/

### 1.2 Problems with the Current Approach

- **Oxygen dependency**: Oxygen XML Editor is a commercial tool that is no longer supported for this use case in modern versions. The build requires a special workaround ANT script.
- **Fragility**: The pipeline is not easily reproducible by new team members and is not integrated into the standard CI/CD build.
- **Aesthetics**: The WebHelp output uses an outdated visual style inconsistent with NASA Horizon Design System (HDS) standards.
- **Performance**: The monolithic DocBook file is extremely large, causing slow transforms and slow initial page loads.
- **Maintainability**: DocBook is a complex toolchain that few team members understand.

### 1.3 Goals

1. **Eliminate the Oxygen dependency** by implementing a standalone Java tool that transforms DocBook XML directly to a modern static HTML site.
2. **Preserve search capability** — full-text, client-side search across all classes, attributes, data types, and namespaces.
3. **Improve visual consistency** with the NASA Horizon Design System.
4. **Improve performance** via lazy loading and split output files rather than a single large document.
5. **Enable future extensibility** to support richer input formats (schema, RDF/OWL, JSON-LD) as the PDS4 toolchain evolves.

---

## 2. Operational Concept

### 2.1 System Overview

`dd-search` is a standalone Java command-line tool that:

1. Accepts a PDS4 Data Dictionary DocBook XML file as input.
2. Parses and indexes its content (classes, attributes, data types, permissible values, namespaces).
3. Generates a self-contained static HTML/CSS/JavaScript site with:
   - Hierarchical navigation by namespace and dictionary type (Common, Discipline LDD, Mission LDD).
   - Individual detail pages for each class and attribute.
   - Client-side full-text search across all content.
   - Visual styling consistent with NASA Horizon Design System.

The output is a directory of static files that can be deployed to any web server or static hosting service without a backend.

### 2.2 Actors

| Actor | Role |
|-------|------|
| **PDS Data Provider** | Browses and searches the dictionary to find classes and attributes to use when building PDS4-compliant data products |
| **PDS Operations / Engineering** | Uses the dictionary as a reference when validating data products and maintaining the Information Model |
| **PDS IM Release Engineer** | Runs `dd-search` as part of the build/release pipeline to regenerate the published documentation |

### 2.3 Use Cases

| ID | Name | Actor | Description |
|----|------|-------|-------------|
| UC-01 | Generate static site | Release Engineer | Run `dd-search` on a DocBook XML file to produce the static HTML output |
| UC-02 | Search dictionary | Data Provider | Search for a class or attribute by name or keyword |
| UC-03 | Browse by namespace | Data Provider | Navigate all classes/attributes within a specific namespace (e.g., `img`, `cart`, `pds`) |
| UC-04 | View class details | Data Provider | View a class page showing description, associations, attributes, and constraints |
| UC-05 | View attribute details | Data Provider | View an attribute page showing description, data type, permissible values, and usage context |
| UC-06 | Browse data types | Operations | View available PDS4 data types and their definitions |
| UC-07 | Cross-reference | Data Provider | Follow links from a class to its attributes and vice versa |
| UC-08 | Generate for IM version | Release Engineer | Generate documentation for a specific IM version (e.g., 1Q00, 1P00) |

### 2.4 Workflow

```
LDDTool
  └─► PDS4_PDS_DD_<version>.xml  (DocBook XML)
            │
            ▼
        dd-search (Java CLI)
            │
            ├── parse DocBook XML
            ├── build in-memory model (classes, attributes, namespaces, etc.)
            ├── generate search index
            └── write static HTML/CSS/JS output
                    │
                    ▼
            output/                    (deployable static site)
            ├── index.html
            ├── search/
            │   └── index.json         (pre-built search index)
            ├── classes/
            │   └── <class-name>.html
            ├── attributes/
            │   └── <attr-name>.html
            ├── namespaces/
            │   └── <ns>.html
            └── assets/
                ├── style.css          (NASA Horizon Design System)
                └── search.js          (client-side search)
```

---

## 3. Design Specification

### 3.1 Architecture

#### 3.1.1 Input

**Primary (current)**: PDS4 Data Dictionary DocBook XML  
- File: `PDS4_PDS_DD_<version>.xml` generated by LDDTool  
- Format: DocBook 4.x/5.x XML  

**Future extensions** (not in scope for initial release):
- LDDTool JSON-LD output (`-J` / `-W` flags)
- PDS4 XSD schema files
- RDF/OWL ontology files

The tool should define a clean parser interface (`DDInputParser`) so future input formats can be added without rewriting the generation layer.

#### 3.1.2 Processing

The tool builds an in-memory domain model from the parsed input:

- `Namespace` — identifier, title, type (Common / Discipline LDD / Mission LDD), steward
- `DDClass` — name, namespace, description, associations, abstract flag, deprecated flag
- `DDAttribute` — name, namespace, description, data type, permissible values, nillable flag, deprecated flag
- `DDDataType` — name, description, XML type
- `DDUnit` — name, description

This domain model is independent of input format and drives all output generation.

#### 3.1.3 Output

The generator writes a directory of static files:

| File | Purpose |
|------|---------|
| `index.html` | Landing page with namespace overview and statistics |
| `classes/index.html` | Sortable/filterable list of all classes |
| `classes/<name>.html` | Detail page for a single class |
| `attributes/index.html` | Sortable/filterable list of all attributes |
| `attributes/<name>.html` | Detail page for a single attribute |
| `namespaces/<id>.html` | Namespace overview page |
| `datatypes/index.html` | Data types reference |
| `assets/style.css` | NASA Horizon Design System styles |
| `assets/search.js` | Client-side search (Fuse.js or similar) |
| `search/index.json` | Pre-built search index |

#### 3.1.4 Search

Client-side full-text search is implemented using a pre-built JSON index loaded at page startup. No server is required.

Search must cover:
- Class names and descriptions
- Attribute names and descriptions
- Permissible value labels and descriptions
- Namespace identifiers and titles

Search results must display: item name, type (class/attribute), namespace, and a brief description excerpt.

### 3.2 Technology Stack

| Component | Technology |
|-----------|-----------|
| Core tool | Java 11+, Maven |
| DocBook parsing | Apache Xerces / standard Java XML (JAXP) |
| HTML generation | Apache Velocity templates or similar (no Thymeleaf server dependency) |
| Client-side search | Fuse.js (CDN or bundled) |
| CSS framework | NASA Horizon Design System |
| CI/CD | GitHub Actions (existing NASA-PDS workflows) |

### 3.3 Command-Line Interface

```
dd-search [options] <input-file>

Options:
  -o, --output <dir>       Output directory (default: ./output)
  -V, --im-version <ver>   IM version string (e.g. 1Q00) — embedded in output
  -t, --title <title>      Site title override
  -h, --help               Show help
  -v, --version            Show version
```

Example:
```bash
java -jar dd-search.jar -o ./site -V 1Q00 PDS4_PDS_DD_1Q00.xml
```

### 3.4 Non-Functional Requirements

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-01 | Performance | Initial page load < 3 seconds on a standard connection |
| NFR-02 | Performance | Search results appear within 500ms of keystroke |
| NFR-03 | Accessibility | Output meets WCAG 2.1 Level AA |
| NFR-04 | Portability | Tool runs on any platform with Java 11+ (no OS-specific dependencies) |
| NFR-05 | Deployability | Output is self-contained static files (no server required) |
| NFR-06 | Maintainability | No dependency on commercial tools (Oxygen, DocBook toolchain) |
| NFR-07 | Compatibility | Generates output for all supported IM versions (1B00–current) |
| NFR-08 | Style | Visual output is consistent with NASA Horizon Design System |
| NFR-09 | Responsiveness | Output is usable on desktop, tablet, and mobile viewports |

---

## 4. Requirements

The following requirements are tracked as individual GitHub issues in this repository.

### 4.1 Functional Requirements

| ID | Title |
|----|-------|
| REQ-F-01 | Parse PDS4 Data Dictionary DocBook XML input |
| REQ-F-02 | Generate static HTML site from parsed content |
| REQ-F-03 | Generate pre-built client-side search index |
| REQ-F-04 | Support navigation by namespace (Common, Discipline LDD, Mission LDD) |
| REQ-F-05 | Generate individual detail pages for each class |
| REQ-F-06 | Generate individual detail pages for each attribute |
| REQ-F-07 | Generate data types reference page |
| REQ-F-08 | Generate namespace overview pages |
| REQ-F-09 | Provide cross-reference links between classes and attributes |
| REQ-F-10 | Support multiple IM versions via CLI flag |
| REQ-F-11 | Display permissible values on attribute detail pages |
| REQ-F-12 | Indicate deprecated, abstract, and nillable items visually |
| REQ-F-13 | Define extensible parser interface for future input formats |

### 4.2 Non-Functional Requirements

| ID | Title |
|----|-------|
| REQ-NF-01 | No commercial tool dependencies |
| REQ-NF-02 | Standalone Java CLI (Java 11+) |
| REQ-NF-03 | Output is self-contained static files |
| REQ-NF-04 | Initial page load under 3 seconds |
| REQ-NF-05 | Search response under 500ms |
| REQ-NF-06 | WCAG 2.1 AA accessibility compliance |
| REQ-NF-07 | NASA Horizon Design System visual style |
| REQ-NF-08 | Mobile-responsive output |

### 4.3 Integration Requirements

| ID | Title |
|----|-------|
| REQ-I-01 | Accept DocBook XML generated by LDDTool as primary input |
| REQ-I-02 | Provide GitHub Actions workflow for automated generation |
| REQ-I-03 | Future: Accept LDDTool JSON-LD / schema / RDF/OWL as alternate input |

---

## 5. Relationship to Existing Tools

### 5.1 LDDTool (pds4-information-model)

`dd-search` is a downstream consumer of LDDTool output. LDDTool continues to generate the DocBook XML; `dd-search` replaces the Oxygen/ANT transform step. The two tools are decoupled — `dd-search` can be run independently on any conformant DocBook XML file.

LDDTool already contains an experimental web app (`-W` flag) and MkDocs output in `model-dmdocument`. Those outputs remain available but target a different workflow. `dd-search` is the officially supported documentation generation path for the published PDS4 Data Dictionary.

### 5.2 docbook2webhelp

An earlier Claude Code-assisted prototype (`docbook2webhelp`) demonstrated that the Oxygen dependency can be eliminated. `dd-search` formalizes that approach as a properly maintained NASA-PDS repository with CI/CD, versioned releases, and requirements traceability.

---

## 6. Open Questions

| # | Question | Resolution |
|---|----------|------------|
| OQ-01 | What is the target deployment host? (pds.nasa.gov, GitHub Pages, or other) | TBD |
| OQ-02 | Should dd-search also ingest Mission and Discipline LDD DocBook files separately, or only the All-LDD combined file? | TBD |
| OQ-03 | Should the tool support incremental generation (only regenerate changed namespaces)? | TBD |
| OQ-04 | What is the required browser support matrix? | TBD |
