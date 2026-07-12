# Semantic Open Knowledge Format (S-OKF)
## Version 0.1-JSONLD — Specification Draft
The **Semantic Open Knowledge Format (S-OKF)** is a graph-native evolution of the Open Knowledge Format. Instead of using a loose collection of Markdown files with frontmatter, S-OKF structures knowledge as JSON-LD documents organized into a discoverable bundle.
Every bundle in S-OKF forms an explicit, queryable RDF Graph using the JSON-LD serialization format.
## 1. Bundle Directory Structure
An S-OKF Knowledge Bundle is a directory tree of extensionless files containing JSON-LD data. The directory structures match the logical concepts, but the files omit extensions to align perfectly with POSIX-style paths.
```text
path/to/bundle/
├── index          # REQUIRED. Root catalog linking all concepts.
├── log            # OPTIONAL. Graph-native change history using PROV-O.
├── <concept>      # A concept file (JSON-LD syntax, no extension).
└── <subdirectory>/
    ├── index      # Directory-level sub-catalog.
    └── <concept>  # Organically nested concept file.

```
## 2. Core Vocabularies & Context
S-OKF repositories MUST include a central context file at the repository root named `context.jsonld`. This central context is the canonical JSON-LD context that SHOULD be referenced (by relative path) from all concept JSON-LD objects, index files, and the log file. Using a single shared context ensures consistent prefix expansion and stable interpretation across the bundle.

The canonical context maps the foundational namespaces used throughout S-OKF (for example `rdf`, `rdfs`, `dc`, `prov`, and `okf`). The repository root file `context.jsonld` contains these mappings; concept files should reference it using a relative context reference, e.g.:
```json
{
  "@context": "context.jsonld",
  "@id": "/tables/orders",
  "@type": "okf:BigQueryTable",
  "rdfs:label": "Customer Orders"
}
```

(See `context.jsonld` in the repository root for the canonical namespace bindings.)

## 3. Concept Files
Every concept document is an individual JSON-LD node. File naming maps directly to the node's subject URI (@id).
### Structural Mapping Rules
 * **No File Extensions:** Concept files are saved with a completely blank file extension (e.g., tables/orders instead of tables/orders.jsonld or tables/orders.md).
 * **Frontmatter Translation:** The original OKF YAML metadata attributes map cleanly to JSON-LD primitives:
   * type \rightarrow @type (defining the semantic class of the concept)
   * name or title \rightarrow rdfs:label
   * description \rightarrow dc:description
   * resource \rightarrow okf:boundResource (pointing to the target URI)
   * timestamp \rightarrow dc:modified
 * **Markdown Body Integration:** The structural or free-form Markdown body of the concept is embedded as a language-tagged string inside the rdfs:comment property.
### Concept Example (/tables/orders)
Here is how a standard asset description is modeled as an S-OKF Concept (note the shared context reference):
```json
{
  "@context": "context.jsonld",
  "@id": "/tables/orders",
  "@type": "okf:BigQueryTable",
  "rdfs:label": "Customer Orders",
  "dc:description": "One row per completed customer order across all channels.",
  "okf:boundResource": {
    "@id": "https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders"
  },
  "dc:modified": "2026-05-28T14:30:00Z",
  "rdfs:comment": {
    "@value": "# Schema\n| Column | Type | Description |\n|---|---|---|\n| `order_id` | STRING | Unique order identifier. |\n| `customer_id` | STRING | Foreign key linking to [/tables/customers]. [...]",
    "@language": "en"
  }
}

```
## 4. Index Files
The index file serves as the manifest and explicit map for progressive disclosure. It functions as a collection node linking to child or sibling entity URIs. Index files SHOULD reference the central `context.jsonld` rather than embedding ad-hoc contexts.
### Index Example (/index)
```json
{
  "@context": "context.jsonld",
  "@id": "/index",
  "@type": "rdfs:ResourceCatalog",
  "dc:title": "Root Knowledge Directory",
  "rdfs:seeAlso": [
    { "@id": "/datasets/sales" },
    { "@id": "/tables/orders" },
    { "@id": "/tables/customers" },
    { "@id": "/playbooks/incident-response" }
  ]
}

```
## 5. Log Files
The change log shifts from basic text formatting to an auditable lineage graph using the **W3C PROV-O (Provenance Ontology)** specification. Each modification entry tracking the bundle is defined as a prov:Activity. Log files SHOULD reference the central `context.jsonld`.
 * **File Changes:** Modifying or creating a concept represents a prov:Activity.
 * **The Concepts:** Concepts act as prov:Entity states that are prov:wasGeneratedBy or prov:wasInvalidatedBy activities.
 * **The Authors:** People or automated workflows executing the changes are defined as a prov:Agent.
### Log Example (/log)
```json
{
  "@context": "context.jsonld",
  "@graph": [
    {
      "@id": "/log",
      "@type": "prov:Bundle",
      "rdfs:label": "Knowledge Revision Log"
    },
    {
      "@id": "/activities/commit-ee67a5c",
      "@type": "prov:Activity",
      "prov:startedAtTime": "2026-05-28T14:30:00Z",
      "prov:endedAtTime": "2026-05-28T14:32:00Z",
      "dc:description": "Added new BigQuery table reference for Customer Metrics.",
      "prov:wasAssociatedWith": { "@id": "/agents/amirhormati" }
    },
    {
      "@id": "/tables/customer-metrics",
      "@type": "prov:Entity",
      "prov:wasGeneratedBy": { "@id": "/activities/commit-ee67a5c" }
    },
    {
      "@id": "/agents/amirhormati",
      "@type": "prov:Agent",
      "prov:type": "prov:Person",
      "rdfs:label": "Amir Hormati"
    }
  ]
}

```
## 6. Graph Relationships & Cross-Linking
Because S-OKF files are native JSON-LD, standard Markdown cross-links within prose (`rdfs:comment`) are fully compatible with graph parsing. More importantly, relationships between concepts in S-OKF SHOULD be modeled using ordinary RDF object properties — not restricted to `rdfs:seeAlso` alone.

Any predicate that is appropriate for the domain may be used to link nodes. For example, you can use domain-specific predicates such as `okf:dependsOn`, `okf:derivedFrom`, `schema:author`, or even a simple `likes` property. The following is a minimal, valid JSON-LD example showing a relationship expressed with a custom property:
```json
{
  "@context": "context.jsonld",
  "@id": "john",
  "likes": { "@id": "mary" }
}
```

Using explicit object properties (instead of only seeAlso) enables richer semantics, easier SPARQL queries, and clearer intent for automated consumption.

This approach lets consumption agents easily parse and ingest the repository natively as a global graph structure using standard graph engines without writing custom regex parser utilities.

