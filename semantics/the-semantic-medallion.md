# The Semantic Medallion: Building a Knowledge Graph-Powered Data Catalog

Soruce: https://moderndata101.substack.com/p/the-semantic-medallion

_How you can transform raw data sources into a unified knowledge graph in four lines of Python_

**Veronika Heimsbakk**  
_May 14, 2026_

---

Every data engineer knows the medallion architecture: **Bronze** (raw), **Silver** (cleaned), **Gold** (business-ready). It’s become the standard pattern for data lakehouses.

But what if your Gold layer wasn’t just "clean data in nice tables"? What if it were a knowledge graph, where every record understands its relationship to every other record, across all your sources?

> _The semantic upgrade from Gold to Gold Graph | Adapted from concepts shared by the Author, curated by Modern Data 101. This article draws from practical observations based on the work we have done for a client._

### The Problem: A Data Catalog That Actually Connects Things

The client had a familiar challenge: several data sources, each describing overlapping concepts from different angles, to a greater or lesser extent.

They are building a data catalog and exploring the wonders of knowledge graphs and semantic technologies. We’re not building a static list of tables and columns, but something that could answer questions like:

- _"Show me everything we know about this record."_
- _"Which data sources contain information about this agent?"_
- _"How does this record in System A relate to that record in System B?"_

Traditional data catalogs struggle with this. They can tell you _where_ data lives. They can’t easily tell you _how_ it connects.

> _The challenge of traditional catalogs | Adapted from concepts shared by the Author, curated by Modern Data 101_

### The Architecture: Medallion Meets Knowledge Graph

We kept the familiar medallion structure but redefined what each layer delivers:

#### Bronze: Harvest and Land

Standard practice. We used data orchestration tools with connectors to pull from various sources: APIs, databases, file shares, and external registries. Raw data lands as-is. No transformations yet.

#### Silver: Structure and Identify

Here’s where it gets interesting. Instead of just cleaning data and enforcing types, we added one crucial step: **minting IRIs (Internationalized Resource Identifiers) for every entity**.

Each record gets a stable, globally unique identifier. Not a database auto-increment ID. Not a GUID, which means nothing. An IRI that identifies the thing being described.

```python
# Silver layer: structured DataFrames with IRIs
customer_df = pl.DataFrame({
    "iri": ["https://example.org/customer/C-001", "https://example.org/customer/C-002"],
    "source_id": ["CRM-12345", "CRM-12346"],
    "name": ["Alice Johnson", "Bob Smith"],
    "email": ["alice@example.com", "bob@example.com"]
})
```

The IRI is the join key that works across all systems. Not just within one database, but across your entire data landscape.

#### Gold: Harmonize and Publish as RDF

This is where the magic happens. We map Silver DataFrames to a shared ontology: a common vocabulary that describes what things mean and how they relate. Then we publish as RDF, ready for graph queries.

> _Silver DataFrame to Gold Graph transition in Gold layer | Adapted from concepts shared by the Author, curated by Modern Data 101_

**Four lines of Python:**

```python
from maplib import Model

m = Model()
m.add_template(customer_template)
m.map("ex:Customer", customer_df)
```

That’s it! DataFrame to knowledge graph.

### What the Gold Layer Actually Looks Like

**Traditional Gold layer (Parquet/Delta):**

```text
gold/
  customers.parquet
  contracts.parquet
  billing_accounts.parquet
  external_registry.parquet
```

Four separate tables. You write joins to connect them. You maintain the join logic in dbt or SQL scripts. When relationships change, you update code.

**Our Gold layer (RDF):**

```turtle
<https://example.org/record/C-001> a :Customer ;
    :name "Alice Johnson" ;
    :email "alice@example.com" ;
    :hasBillingAccount <https://example.org/billing/B-001> ;
    :hasContract <https://example.org/contract/K-2024-001> ;
    :registeredIn <https://example.org/registry/company/12345> .

<https://example.org/billing/B-001> a :BillingAccount ;
    :accountHolder <https://example.org/customer/C-001> ;
    :outstandingBalance 0 ;
    :paymentMethod "Credit Card" .
```

The relationships are _in the data_. Not in join logic. Not in SQL scripts. In the data itself.

> _Traditional Gold Layer vs. Semantic Gold Layer | Adapted from concepts shared by the Author, curated by Modern Data 101_

### The Catalog Experience

With the knowledge graph in place, the data catalog becomes genuinely useful.

**Query: “Show me everything about customer C-001”**

```sparql
SELECT ?property ?value
WHERE {
    <https://example.org/customer/C-001> ?property ?value .
}
```

_Result:_ Every fact from every source, unified under one identifier.

**Query: “Which sources contribute to our customer records?”**

```sparql
SELECT ?source (COUNT(?customer) as ?records)
WHERE {
    ?customer a :Customer ;
        :sourceSystem ?source .
}
GROUP BY ?source
```

**Query: “Find customers with billing issues who have active contracts”**

```sparql
SELECT ?customer ?name ?balance ?contractStatus
WHERE {
    ?customer a :Customer ;
        :name ?name ;
        :hasBillingAccount ?account ;
        :hasContract ?contract .
    ?account :outstandingBalance ?balance .
    ?contract :status "Active" .
    FILTER(?balance > 0)
}
```

No joins. No hunting through five different tables. The relationships are navigable directly.

> _Structural Catalogs vs. Semantic Catalogs | Adapted from concepts shared by the Author, curated by Modern Data 101_

### The “Four Lines of Python” in Detail

Let me show you what those four lines actually do.

**Step 1: Define a template**

We use OTTR templates (stOTTR syntax) to map DataFrame columns to ontology properties:

```turtle
@prefix ex: <https://example.org/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

ex:Customer [?iri, ?name, ?email, ?billing_account] :: {
    ottr:Triple(?iri, a, ex:Customer) ,
    ottr:Triple(?iri, ex:name, ?name) ,
    ottr:Triple(?iri, ex:email, ?email),
    ottr:Triple(?iri, ex:hasBillingAccount, ?billing_account)
} .
```

**Step 2: Map and expand**

```python
from maplib import Model
import polars as pl

# Load your Silver DataFrame
customers = pl.read_parquet("silver/customers.parquet")

# Create model and add template
m = Model()
m.add_template(template)

# Map DataFrame to RDF
m.map("ex:Customer", customers)

# Write to file
m.write("gold/customers.ttl", format="turtle")
```

That’s the complete Silver-to-Gold transformation. The template captures the semantics. The DataFrame provides the data. `maplib` does the rest.

### Why This Works for Data Catalogs

Traditional data catalogs have a fundamental limitation: they describe data _structurally_. A knowledge graph-powered catalog describes data _semantically_. What things mean, how they relate, and what you can ask.

**Three capabilities we gained:**

1. **Entity resolution across sources**  
   When the same customer appears in CRM, billing, and an external registry, we link them: Query any one, get information from all three.

2. **Semantic search**  
   “Find all entities related to financial compliance” doesn’t require knowing which tables contain compliance data. The ontology encodes that `:ComplianceOfficer`, `:AuditRecord`, and `:RegulatoryFiling` are all subtypes of `:ComplianceRelated`.

3. **Impact analysis**  
   “What would be affected if we change the customer identifier format?” Trace all relationships in the graph to find every downstream dependency.

> _Impact of knowledge graph-powered catalog | Adapted from concepts shared by the Author, curated by Modern Data 101_

### Enter DCAT: The Standard for Data Catalogs

Here’s where it gets even better. We didn’t invent a custom vocabulary for describing our datasets; we used **DCAT (Data Catalog Vocabulary)**, a W3C standard designed exactly for this purpose.

DCAT gives you a ready-made ontology for describing:

- **Catalogs (`dcat:Catalog`)**: collections of datasets
- **Datasets (`dcat:Dataset`)**: logical groupings of data
- **Distributions (`dcat:Distribution`)**: how you can access the data (API, file download, SPARQL endpoint)
- **Data Services (`dcat:DataService`)**: APIs and endpoints that serve data

> _DCAT used to transition traditional medallion into semantic enrichment layers | Adapted from concepts shared by the Author, curated by Modern Data 101_

#### 1. Interoperability out of the box

DCAT is used by data.gov, the European Data Portal, and thousands of organizations worldwide. When you describe your datasets with DCAT, you’re speaking a common language. Your internal catalog can federate with external catalogs. Your metadata is portable.

#### 2. Rich metadata without reinventing the wheel

Instead of designing your own “dataset” schema, DCAT gives you:

```turtle
<https://example.org/dataset/customers> a dcat:Dataset ;
    dct:title "Customer Master Data" ;
    dct:description "Unified customer records from CRM, billing, and registry" ;
    dct:publisher <https://example.org/org/data-team> ;
    dct:issued "2024-01-15"^^xsd:date ;
    dct:modified "2024-04-20"^^xsd:date ;
    dcat:theme <http://publications.europa.eu/resource/authority/data-theme/ECON> ;
    dcat:distribution <https://example.org/dataset/customers/sparql> , <https://example.org/dataset/customers/parquet> .

<https://example.org/dataset/customers/sparql> a dcat:Distribution ;
    dcat:accessURL <https://example.org/sparql> ;
    dct:format "application/sparql-query" .

<https://example.org/dataset/customers/parquet> a dcat:Distribution ;
    dcat:downloadURL <https://example.org/files/customers.parquet> ;
    dct:format "application/vnd.apache.parquet" .
```

Publishers, themes, access methods, formats, and lineage; all standardized.

#### 3. It connects the catalog TO the data

Here’s the key insight: DCAT describes your datasets as RDF. Your data is also RDF. They live in the same graph.

```sparql
# Find all datasets that contain customer information
SELECT ?dataset ?title
WHERE {
    ?dataset a dcat:Dataset ;
        dct:title ?title .

    # The dataset's distribution points to a graph
    ?dataset dcat:distribution ?dist .
    ?dist dcat:accessURL ?endpoint .

    # That graph contains Customer entities
    GRAPH ?endpoint {
        ?customer a :Customer .
    }
}
```

Your catalog isn’t a separate system pointing at data; it’s part of the same knowledge graph. Metadata and data, unified.

#### 4. Provenance and lineage are built-in

DCAT works seamlessly with PROV-O (the provenance ontology). You can trace where data came from:

```turtle
<https://example.org/dataset/customers> prov:wasDerivedFrom <https://example.org/dataset/crm-export> , <https://example.org/dataset/billing-extract> ;
    prov:wasGeneratedBy <https://example.org/activity/customer-harmonization> .
```

Data lineage as queryable triples, not a separate lineage tool with its own database.

### DCAT in the Architecture

DCAT becomes the glue between the medallion layers:

- **Bronze datasets**: Described with DCAT, linked to their raw sources
- **Silver datasets**: DCAT distributions pointing to structured Parquet files
- **Gold layer**: DCAT catalog describing the unified knowledge graph, with SPARQL endpoint as distribution

The catalog isn’t a separate application; it’s part of the graph itself. Query the data, query the catalog, same endpoint, same language.

### What We Learned

- **Start with IRIs early.** The biggest challenge wasn’t the RDF conversion; it was establishing stable identifiers in the Silver layer. Do this first.
- **Ontology design is iterative.** We didn’t design the perfect ontology up front. We started with what we had, mapped it, then refined as we discovered relationships.
- **The medallion pattern still applies.** Bronze-Silver-Gold is a useful mental model. We just redefined what “Gold” means: not “clean tables” but “connected knowledge“.
- **“Four lines of Python” is real.** `maplib` made the transformation tractable. Without it, we’d be writing plenty of lines of RDF serialization code.

### The Bigger Picture

This project convinced me of something: the medallion architecture isn’t just about _data quality_ layers. It’s about **semantic enrichment** layers.

- **Bronze:** Raw data (no semantics)
- **Silver:** Structured data with identifiers (local semantics)
- **Gold:** Harmonized data with shared vocabulary (global semantics)

The Gold layer becomes a knowledge graph not because graphs are trendy, but because that’s what _“fully enriched, connected, query-ready data”_ actually looks like.
