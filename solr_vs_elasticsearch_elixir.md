# Solr vs Elasticsearch — Zero to Hero with Elixir

> **Scope:** Full conceptual breakdown, architecture differences, pro/con analysis, Elixir code examples for both engines, and a complete Phoenix LiveView demo app backed by Elasticsearch.
> **Date:** June 2026 | Elasticsearch 8.x | Elixir 1.17+ | Phoenix 1.7+

---

## Table of Contents

1. [Zero to Hero — What Are These Engines?](#1-zero-to-hero)
2. [Architecture Deep Dive](#2-architecture-deep-dive)
3. [Core Differences at a Glance](#3-core-differences)
4. [Pro/Cons Comparison](#4-procons)
5. [Elixir + Solr — Full Examples](#5-elixir--solr)
6. [Elixir + Elasticsearch — Full Examples](#6-elixir--elasticsearch)
7. [Demo Application: Phoenix LiveView + Elasticsearch](#7-demo-application)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Zero to Hero

### What is Apache Solr?

Apache Solr is an open-source enterprise search platform built on top of **Apache Lucene** — the same underlying text-search library that also powers Elasticsearch. Solr was created at CNET in 2004, open-sourced in 2006, and donated to the Apache Software Foundation. For over a decade it was the *de facto* choice for production full-text search.

Solr stores documents in **cores** (or **collections** in SolrCloud mode). Every field must either be declared in a schema or matched by a dynamic field rule. Queries are submitted as URL-encoded HTTP requests returning XML or JSON.

```
┌─────────────────────────────────────┐
│               Apache Solr           │
│                                     │
│  ┌─────────┐    ┌────────────────┐  │
│  │  Core 1 │    │  Core 2        │  │
│  │ products│    │ articles       │  │
│  └────┬────┘    └───────┬────────┘  │
│       │                 │           │
│  ┌────▼─────────────────▼────────┐  │
│  │         Apache Lucene         │  │
│  │  (Inverted Index on disk)     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### What is Elasticsearch?

Elasticsearch was released in 2010 by Shay Banon (built on Lucene, like Solr) as a distributed, RESTful search and analytics engine. It became the backbone of the **Elastic Stack** (formerly ELK: Elasticsearch, Logstash, Kibana) and is today the most widely used search engine in the world.

Unlike Solr, Elasticsearch was designed from day one with **distributed operation**, **schemaless ingestion**, and a **JSON-native REST API** as first-class concerns.

```
┌──────────────────────────────────────────────────────┐
│                  Elasticsearch Cluster               │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │   Node 1   │  │   Node 2   │  │   Node 3   │     │
│  │ (master +  │  │ (data)     │  │ (data)     │     │
│  │  data)     │  │            │  │            │     │
│  └─────┬──────┘  └──────┬─────┘  └──────┬─────┘    │
│        │                │               │            │
│  ┌─────▼────────────────▼───────────────▼──────┐    │
│  │          Index: "products" (5 shards)        │    │
│  │  Shard 0   Shard 1   Shard 2   Shard 3 ...  │    │
│  │  [primary] [replica] [primary] [replica]     │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### The Shared Foundation: Apache Lucene

Both engines sit on top of the same inverted index:

```
Document:  "The quick brown fox"

Inverted Index:
  "the"   → [doc_1, doc_5, doc_12]
  "quick" → [doc_1, doc_8]
  "brown" → [doc_1, doc_3]
  "fox"   → [doc_1, doc_7, doc_9]
```

When you search for "quick fox", Lucene looks up both terms and intersects (or unions) the postings lists, then ranks by TF-IDF or BM25 relevance score.

---

## 2. Architecture Deep Dive

### Solr Architecture

| Layer | Component | Detail |
|---|---|---|
| Index unit | **Core** (standalone) / **Collection** (SolrCloud) | One Lucene index per core |
| Coordination | **ZooKeeper** (mandatory for SolrCloud) | Leader election, config distribution |
| Sharding | Explicit shard assignment | You define shard count at collection creation; cannot change without re-indexing |
| Replication | Pull replication (standalone) or ZK-managed (SolrCloud) | |
| Schema | `schema.xml` (traditional) or managed schema via API | Schemaless mode exists but is limited |
| Query parser | **Lucene**, **DisMax**, **eDisMax** | eDisMax is dominant; very flexible for relevance tuning |
| Config | `solrconfig.xml` per core | XML-heavy; powerful but verbose |
| Admin UI | Solr Admin UI (built in, port 8983) | |

### Elasticsearch Architecture

| Layer | Component | Detail |
|---|---|---|
| Index unit | **Index** | Can hold billions of documents |
| Coordination | Built-in consensus (Raft-like from ES 8.x, previously Zen discovery) | No external ZooKeeper required |
| Sharding | Primary + replica shards | Configured per index; dynamic rebalancing |
| Replication | Synchronous replica writes | All replicas must acknowledge before success |
| Schema | Dynamic mapping (auto-detect) or explicit mappings | True schemaless ingestion possible |
| Query DSL | **Query DSL** (JSON) | Extremely expressive; leaf + compound queries |
| Config | Index settings + cluster settings via REST API | Almost everything configurable at runtime |
| Admin UI | **Kibana** (separate), or `_cat` APIs, or Elastic Cloud Console | |

---

## 3. Core Differences

| Dimension | Solr | Elasticsearch |
|---|---|---|
| **Origin** | 2004 (Apache) | 2010 (Elastic NV) |
| **Native API format** | XML and JSON over HTTP | JSON over HTTP (REST-first design) |
| **Cluster coordination** | ZooKeeper (external process required) | Built-in (no extra process) |
| **Schema** | Strict by default; managed schema available | Dynamic mapping by default |
| **Near real-time (NRT)** | ~1 second refresh by default | ~1 second by default; configurable to milliseconds |
| **Distributed joins** | Not supported natively | Not supported (use denormalization or parent-child) |
| **Aggregations** | Facets / JSON Facet API | Full aggregations framework (metric, bucket, pipeline) |
| **Full-text relevance tuning** | eDisMax, function queries, Solr Learning-to-Rank | Rescore, function score, Learning-to-Rank (paid), vector search |
| **Vector / kNN search** | Dense Vector field (limited, added late) | kNN search (`dense_vector`), first class since 8.x |
| **Analytics / Time-series** | Limited without extensive plugins | Data Streams, ILM, Rollups, ES|QL |
| **Security** | Ranger / plugin-based | Built-in TLS, roles, API keys, LDAP (free in 8.x) |
| **Ecosystem** | Apache ecosystem (Nutch, Tika, etc.) | Elastic Stack (Kibana, Logstash, Beats, APM) |
| **Commercial support** | Apache foundation; Lucidworks commercial | Elastic NV (Elastic Cloud, subscription features) |
| **License** | Apache 2.0 | SSPL / Elastic License 2.0 (self-managed); Apache 2.0 for some clients |
| **Operational complexity** | Higher (ZooKeeper required for HA) | Lower for small clusters; ZK-free since ES 7.x |
| **Learning curve** | Steeper (XML config, Solr-specific concepts) | Gentler for REST API users |

---

## 4. Pro/Cons

### Apache Solr

**Pros**
- Battle-hardened for enterprise document search (PDFs, Office docs via Tika)
- eDisMax query parser offers very fine-grained relevance control without extra cost
- Full Apache License 2.0 with no cloud-restricted features
- Mature faceting for e-commerce-style catalog search
- Rich text analysis pipeline (char filters → tokenizers → token filters) easily configured in XML
- SolrCloud proven in very high-volume crawl/index pipelines (web search)

**Cons**
- ZooKeeper dependency adds operational overhead (separate process, separate monitoring)
- XML-centric configuration is verbose and harder to automate with modern IaC tools
- Schema changes require careful migration; re-indexing is often unavoidable
- Weaker first-class support for vector/kNN search compared to ES 8.x
- Admin UI is functional but dated; no Kibana-equivalent in the open-source stack
- Community momentum has shifted; talent pool for Solr is shrinking

### Elasticsearch

**Pros**
- JSON-native REST API — trivial to integrate from any language
- No external coordination process; cluster bootstrapping is simple
- Dynamic mapping reduces time-to-first-query dramatically
- Best-in-class aggregation framework for analytics workloads
- First-class kNN / dense_vector search (hybrid BM25 + vector since 8.8)
- Elastic Stack integration: Kibana dashboards, Logstash pipelines, Beats agents
- Extensive runtime configuration without restarts
- Strong security model included for free since ES 8.0

**Cons**
- SSPL license on some server-side features creates legal uncertainty for SaaS products
- Memory hungry: JVM heap + off-heap field data caches; sizing requires experience
- Open-ended dynamic mapping can create "mapping explosion" if not guarded
- High cardinality `keyword` fields in aggregations can exhaust JVM heap
- ES|QL and ESRE (Elastic Relevance Engine) features require paid subscription
- Upgrade paths (especially major versions) require careful index compatibility checks
- Hot-warm-cold architecture requires careful ILM policy tuning

---

## 5. Elixir + Solr

There is no widely maintained official Solr Elixir library. The correct approach is to use **Tesla** (or HTTPoison) as an HTTP client against Solr's JSON API.

### 5.1 Mix Dependencies

```elixir
# mix.exs
defp deps do
  [
    {:tesla, "~> 1.11"},
    {:hackney, "~> 1.20"},   # Tesla adapter
    {:jason, "~> 1.4"}
  ]
end
```

### 5.2 Solr Client Module

```elixir
defmodule MyApp.Search.SolrClient do
  @moduledoc """
  Thin HTTP client for Apache Solr JSON API.
  Configured via application environment:

      config :my_app, MyApp.Search.SolrClient,
        base_url: "http://localhost:8983/solr",
        core: "products"
  """

  use Tesla

  plug Tesla.Middleware.BaseUrl, base_url()
  plug Tesla.Middleware.JSON
  plug Tesla.Middleware.Logger

  # Returns the base URL from application config.
  defp base_url do
    :my_app
    |> Application.get_env(__MODULE__, [])
    |> Keyword.get(:base_url, "http://localhost:8983/solr")
  end

  # Returns the configured core name.
  defp core do
    :my_app
    |> Application.get_env(__MODULE__, [])
    |> Keyword.get(:core, "products")
  end

  @doc """
  Indexes a list of documents into Solr. Each document is a map with string keys.
  Commits immediately by appending ?commit=true.

  ## Example

      iex> SolrClient.index([%{"id" => "1", "title_t" => "Elixir Programming"}])
      {:ok, %{"responseHeader" => %{"status" => 0}}}
  """
  @spec index(list(map())) :: {:ok, map()} | {:error, any()}
  def index(documents) when is_list(documents) do
    case post("/#{core()}/update?commit=true", documents) do
      {:ok, %Tesla.Env{status: 200, body: body}} -> {:ok, body}
      {:ok, %Tesla.Env{status: status, body: body}} -> {:error, {status, body}}
      {:error, reason} -> {:error, reason}
    end
  end

  @doc """
  Executes a full-text search using Solr's select handler.

  ## Parameters

    - `query`   – Lucene query string (e.g. `"elixir programming"`)
    - `opts`    – keyword list of extra Solr parameters:
      - `:rows`   – number of results (default 10)
      - `:start`  – offset for pagination (default 0)
      - `:fl`     – comma-separated field list (default `"*"`)
      - `:fq`     – filter query string
      - `:sort`   – sort expression (e.g. `"score desc"`)

  ## Example

      iex> SolrClient.search("elixir", rows: 5, fl: "id,title_t,score")
      {:ok, %{"response" => %{"numFound" => 3, "docs" => [...]}}}
  """
  @spec search(String.t(), keyword()) :: {:ok, map()} | {:error, any()}
  def search(query, opts \\ []) do
    params =
      opts
      |> Keyword.put_new(:rows, 10)
      |> Keyword.put_new(:start, 0)
      |> Keyword.put_new(:fl, "*")
      |> Keyword.put(:q, query)
      |> Keyword.put(:wt, "json")
      |> Enum.map(fn {k, v} -> {Atom.to_string(k), to_string(v)} end)

    case get("/#{core()}/select", query: params) do
      {:ok, %Tesla.Env{status: 200, body: body}} -> {:ok, body}
      {:ok, %Tesla.Env{status: status, body: body}} -> {:error, {status, body}}
      {:error, reason} -> {:error, reason}
    end
  end

  @doc """
  Deletes documents matching a Lucene query.

  ## Example

      iex> SolrClient.delete_by_query("category_s:obsolete")
      {:ok, %{"responseHeader" => %{"status" => 0}}}
  """
  @spec delete_by_query(String.t()) :: {:ok, map()} | {:error, any()}
  def delete_by_query(query) do
    payload = %{"delete" => %{"query" => query}}

    case post("/#{core()}/update?commit=true", payload) do
      {:ok, %Tesla.Env{status: 200, body: body}} -> {:ok, body}
      {:ok, %Tesla.Env{status: status, body: body}} -> {:error, {status, body}}
      {:error, reason} -> {:error, reason}
    end
  end

  @doc """
  Fetches facet counts for a given field.

  ## Example

      iex> SolrClient.facet("elixir", "category_s")
      {:ok, %{"facet.field" => %{"category_s" => [["books", 12], ["videos", 3]]}}}
  """
  @spec facet(String.t(), String.t()) :: {:ok, map()} | {:error, any()}
  def facet(query, field) do
    params = [
      {"q", query},
      {"wt", "json"},
      {"rows", "0"},
      {"facet", "true"},
      {"facet.field", field}
    ]

    case get("/#{core()}/select", query: params) do
      {:ok, %Tesla.Env{status: 200, body: body}} ->
        counts =
          body
          |> get_in(["facet_counts", "facet_fields", field])
          |> then(fn list ->
            list
            |> Enum.chunk_every(2)
            |> Enum.map(fn [k, v] -> {k, v} end)
            |> Map.new()
          end)

        {:ok, %{"facet.field" => %{field => counts}}}

      {:ok, %Tesla.Env{status: status, body: body}} ->
        {:error, {status, body}}

      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

### 5.3 Schema-Aware Document Struct

```elixir
defmodule MyApp.Search.SolrDocument do
  @moduledoc """
  Represents a Solr document for the `products` core.
  Field names follow Solr's dynamic field naming convention:
    _t  = TextField (analyzed, full-text)
    _s  = StringField (not analyzed, exact match / faceting)
    _i  = IntField
    _f  = FloatField
    _dt = DateTimeField
  """

  @enforce_keys [:id]
  defstruct [
    :id,
    :title_t,
    :description_t,
    :category_s,
    :price_f,
    :created_at_dt
  ]

  @type t :: %__MODULE__{
          id: String.t(),
          title_t: String.t() | nil,
          description_t: String.t() | nil,
          category_s: String.t() | nil,
          price_f: float() | nil,
          created_at_dt: DateTime.t() | nil
        }

  @doc """
  Converts an Ecto schema struct or plain map to a Solr-ready map.
  DateTime values are serialized to ISO 8601 with a trailing Z (Solr requirement).
  """
  @spec to_solr_map(t()) :: map()
  def to_solr_map(%__MODULE__{} = doc) do
    doc
    |> Map.from_struct()
    |> Enum.reject(fn {_k, v} -> is_nil(v) end)
    |> Enum.map(fn
      {:created_at_dt, %DateTime{} = dt} ->
        {"created_at_dt", DateTime.to_iso8601(dt)}
      {k, v} ->
        {Atom.to_string(k), v}
    end)
    |> Map.new()
  end
end
```

### 5.4 Usage Example

```elixir
alias MyApp.Search.{SolrClient, SolrDocument}

# Index documents
docs = [
  %SolrDocument{
    id: "prod-001",
    title_t: "Elixir in Action",
    description_t: "A pragmatic guide to the Elixir programming language.",
    category_s: "books",
    price_f: 39.99,
    created_at_dt: ~U[2024-01-15 10:00:00Z]
  },
  %SolrDocument{
    id: "prod-002",
    title_t: "Programming Phoenix LiveView",
    description_t: "Build interactive real-time web apps with Phoenix LiveView.",
    category_s: "books",
    price_f: 44.99,
    created_at_dt: ~U[2024-03-01 10:00:00Z]
  }
]

solr_maps = Enum.map(docs, &SolrDocument.to_solr_map/1)
{:ok, _} = SolrClient.index(solr_maps)

# Full-text search
{:ok, result} = SolrClient.search("phoenix liveview", rows: 10, fl: "id,title_t,score")
result["response"]["docs"]
# => [%{"id" => "prod-002", "title_t" => "Programming Phoenix LiveView", "score" => 1.42}]

# Faceted search
{:ok, facets} = SolrClient.facet("*:*", "category_s")
facets["facet.field"]["category_s"]
# => %{"books" => 2}
```

---

## 6. Elixir + Elasticsearch

The standard Elixir library for Elasticsearch is **`elasticsearch-elixir`** (`{:elasticsearch, "~> 1.0"}`), authored by Daniel Berkompas. It provides cluster management, index management, bulk indexing, and a Query DSL builder.

### 6.1 Mix Dependencies

```elixir
# mix.exs
defp deps do
  [
    {:elasticsearch, "~> 1.0"},
    {:jason, "~> 1.4"},
    {:hackney, "~> 1.20"}
  ]
end
```

### 6.2 Cluster Configuration

```elixir
# config/config.exs
config :my_app, MyApp.Search.Cluster,
  url: "http://localhost:9200",
  api: Elasticsearch.API.HTTP,
  json_library: Jason,
  indexes: %{
    products: %{
      settings: "priv/elasticsearch/products.json",
      store: MyApp.Search.ProductStore,
      sources: [MyApp.Product],
      bulk_page_size: 5_000,
      bulk_wait_interval: 0
    }
  }
```

### 6.3 Cluster Module

```elixir
defmodule MyApp.Search.Cluster do
  @moduledoc """
  Elasticsearch cluster process. Registered under this module's name.
  Start it in your application supervision tree:

      children = [MyApp.Search.Cluster]
  """
  use Elasticsearch.Cluster, otp_app: :my_app
end
```

### 6.4 Index Settings & Mapping

```json
// priv/elasticsearch/products.json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "synonym_filter"]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["laptop, notebook", "phone, mobile, smartphone"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id":          { "type": "keyword" },
      "title":       { "type": "text", "analyzer": "product_analyzer" },
      "description": { "type": "text", "analyzer": "product_analyzer" },
      "category":    { "type": "keyword" },
      "price":       { "type": "float" },
      "in_stock":    { "type": "boolean" },
      "tags":        { "type": "keyword" },
      "created_at":  { "type": "date" },
      "embedding":   { "type": "dense_vector", "dims": 1536, "index": true, "similarity": "cosine" }
    }
  }
}
```

### 6.5 Store Module (Bulk Indexing Data Source)

```elixir
defmodule MyApp.Search.ProductStore do
  @moduledoc """
  Implements the Elasticsearch.Store behaviour so the library can
  bulk-index records directly from the database.
  """

  @behaviour Elasticsearch.Store

  alias MyApp.Repo
  alias MyApp.Product

  @impl true
  def load(Product, offset, limit) do
    # Fetches a paginated batch of products from Postgres.
    # Called repeatedly during a full re-index (`mix elasticsearch.build`).
    Product
    |> Ecto.Query.offset(^offset)
    |> Ecto.Query.limit(^limit)
    |> Repo.all()
  end
end
```

### 6.6 Document Serialization

```elixir
defimpl Elasticsearch.Document, for: MyApp.Product do
  @moduledoc """
  Serializes an Ecto Product struct into an Elasticsearch document map.
  """

  def id(%MyApp.Product{id: id}), do: id
  def routing(_product), do: false

  def encode(%MyApp.Product{} = product) do
    %{
      id: product.id,
      title: product.title,
      description: product.description,
      category: product.category,
      price: product.price,
      in_stock: product.in_stock,
      tags: product.tags || [],
      created_at: DateTime.to_iso8601(product.inserted_at)
    }
  end
end
```

### 6.7 Search Context Module

```elixir
defmodule MyApp.Search.ProductSearch do
  @moduledoc """
  Encapsulates all Elasticsearch queries for the products index.
  """

  alias MyApp.Search.Cluster

  @index "products"

  @doc """
  Full-text search with optional filters and pagination.

  ## Parameters

    - `query`    – search string; empty string returns all documents
    - `opts`:
      - `:category`  – keyword filter on `category` field
      - `:min_price` – minimum price (float)
      - `:max_price` – maximum price (float)
      - `:page`      – 1-based page number (default 1)
      - `:per_page`  – results per page (default 20)

  ## Returns

      {:ok, %{hits: [%{...}], total: integer, took_ms: integer}}
      {:error, reason}
  """
  @spec search(String.t(), keyword()) :: {:ok, map()} | {:error, any()}
  def search(query, opts \\ []) do
    page     = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 20)
    from     = (page - 1) * per_page

    es_query = build_query(query, opts)

    body = %{
      from: from,
      size: per_page,
      query: es_query,
      highlight: %{
        fields: %{
          title: %{number_of_fragments: 0},
          description: %{fragment_size: 150, number_of_fragments: 3}
        }
      },
      aggs: %{
        categories: %{terms: %{field: "category", size: 20}},
        price_range: %{
          stats: %{field: "price"}
        }
      }
    }

    case Elasticsearch.post(Cluster, "/#{@index}/_search", body) do
      {:ok, raw} ->
        {:ok, parse_response(raw)}

      {:error, %Elasticsearch.Exception{} = err} ->
        {:error, err.message}

      {:error, reason} ->
        {:error, reason}
    end
  end

  @doc """
  Returns the top-N most similar products using kNN (vector search).
  Requires the `embedding` field to be populated on the indexed documents.

  ## Parameters

    - `vector`  – list of floats representing the query embedding (1536 dims)
    - `opts`:
      - `:k`          – number of nearest neighbours (default 10)
      - `:num_candidates` – candidate pool size (default 100)
      - `:category`   – optional keyword pre-filter

  ## Returns

      {:ok, %{hits: [...], total: integer}}
  """
  @spec knn_search(list(float()), keyword()) :: {:ok, map()} | {:error, any()}
  def knn_search(vector, opts \\ []) when is_list(vector) do
    k              = Keyword.get(opts, :k, 10)
    num_candidates = Keyword.get(opts, :num_candidates, 100)

    knn_body =
      %{
        field: "embedding",
        query_vector: vector,
        k: k,
        num_candidates: num_candidates
      }
      |> maybe_add_filter(opts)

    body = %{knn: knn_body, size: k}

    case Elasticsearch.post(Cluster, "/#{@index}/_search", body) do
      {:ok, raw} -> {:ok, parse_response(raw)}
      {:error, reason} -> {:error, reason}
    end
  end

  @doc """
  Indexes a single product document. Suitable for real-time updates triggered
  by Ecto changesets after a create/update lifecycle event.
  """
  @spec index_product(MyApp.Product.t()) :: :ok | {:error, any()}
  def index_product(%MyApp.Product{} = product) do
    case Elasticsearch.put_document(Cluster, product, @index) do
      {:ok, _} -> :ok
      {:error, reason} -> {:error, reason}
    end
  end

  @doc """
  Removes a product from the index by its ID.
  """
  @spec delete_product(binary()) :: :ok | {:error, any()}
  def delete_product(id) do
    case Elasticsearch.delete_document(Cluster, %MyApp.Product{id: id}, @index) do
      {:ok, _} -> :ok
      {:error, reason} -> {:error, reason}
    end
  end

  # Builds the core query DSL based on the search string and filter options.
  defp build_query("", opts), do: %{bool: %{filter: build_filters(opts)}}

  defp build_query(query_str, opts) do
    %{
      bool: %{
        must: %{
          multi_match: %{
            query: query_str,
            fields: ["title^3", "description", "tags^2"],
            type: "best_fields",
            fuzziness: "AUTO"
          }
        },
        filter: build_filters(opts)
      }
    }
  end

  # Assembles a list of ES filter clauses from keyword options.
  defp build_filters(opts) do
    []
    |> maybe_add_category_filter(opts)
    |> maybe_add_price_range(opts)
  end

  defp maybe_add_category_filter(filters, opts) do
    case Keyword.get(opts, :category) do
      nil -> filters
      cat -> [%{term: %{category: cat}} | filters]
    end
  end

  defp maybe_add_price_range(filters, opts) do
    min = Keyword.get(opts, :min_price)
    max = Keyword.get(opts, :max_price)

    if min || max do
      range = %{}
      range = if min, do: Map.put(range, :gte, min), else: range
      range = if max, do: Map.put(range, :lte, max), else: range
      [%{range: %{price: range}} | filters]
    else
      filters
    end
  end

  defp maybe_add_filter(knn_body, opts) do
    case Keyword.get(opts, :category) do
      nil -> knn_body
      cat -> Map.put(knn_body, :filter, %{term: %{category: cat}})
    end
  end

  # Transforms a raw Elasticsearch response into a structured Elixir map.
  defp parse_response(raw) do
    hits = get_in(raw, ["hits", "hits"]) || []
    total = get_in(raw, ["hits", "total", "value"]) || 0
    took_ms = Map.get(raw, "took", 0)
    aggs = Map.get(raw, "aggregations", %{})

    parsed_hits =
      Enum.map(hits, fn hit ->
        source = Map.get(hit, "_source", %{})
        highlights = Map.get(hit, "highlight", %{})
        score = Map.get(hit, "_score")

        source
        |> Map.put("_score", score)
        |> Map.put("_highlight", highlights)
      end)

    %{
      hits: parsed_hits,
      total: total,
      took_ms: took_ms,
      aggregations: aggs
    }
  end
end
```

---

## 7. Demo Application

### Architecture

```
Browser
   │  WebSocket (Phoenix LiveView)
   ▼
Phoenix Endpoint
   │
   ├── Router → SearchLive (LiveView)
   │               │
   │               ├── handle_event("search", ...)
   │               │       └── ProductSearch.search/2
   │               │               └── Elasticsearch HTTP → localhost:9200
   │               │
   │               └── assigns: results, loading, facets
   │
   └── Static assets (TailwindCSS)
```

### 7.1 Application Supervision Tree

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      MyApp.Repo,
      MyAppWeb.Endpoint,
      MyApp.Search.Cluster   # Elasticsearch connection pool
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 7.2 Router

```elixir
# lib/my_app_web/router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  scope "/", MyAppWeb do
    pipe_through :browser
    live "/", SearchLive, :index
    live "/search", SearchLive, :index
  end
end
```

### 7.3 SearchLive LiveView

```elixir
# lib/my_app_web/live/search_live.ex
defmodule MyAppWeb.SearchLive do
  use MyAppWeb, :live_view

  alias MyApp.Search.ProductSearch

  @default_per_page 20

  @impl true
  def mount(_params, _session, socket) do
    {:ok,
     assign(socket,
       query: "",
       results: [],
       total: 0,
       took_ms: 0,
       aggregations: %{},
       loading: false,
       page: 1,
       per_page: @default_per_page,
       selected_category: nil,
       min_price: nil,
       max_price: nil,
       error: nil
     )}
  end

  @impl true
  def handle_params(%{"q" => query} = params, _uri, socket) do
    category = Map.get(params, "category")
    page = params |> Map.get("page", "1") |> String.to_integer()

    {:noreply, do_search(socket, query, category: category, page: page)}
  end

  def handle_params(_params, _uri, socket), do: {:noreply, socket}

  @impl true
  def handle_event("search", %{"query" => query}, socket) do
    socket =
      socket
      |> assign(page: 1, selected_category: nil)
      |> push_patch(to: ~p"/search?q=#{query}")

    {:noreply, socket}
  end

  def handle_event("filter_category", %{"category" => cat}, socket) do
    query = socket.assigns.query
    socket = push_patch(socket, to: ~p"/search?q=#{query}&category=#{cat}")
    {:noreply, socket}
  end

  def handle_event("clear_category", _params, socket) do
    query = socket.assigns.query
    socket = push_patch(socket, to: ~p"/search?q=#{query}")
    {:noreply, socket}
  end

  def handle_event("paginate", %{"page" => page_str}, socket) do
    page = String.to_integer(page_str)
    query = socket.assigns.query
    cat = socket.assigns.selected_category
    path = if cat, do: ~p"/search?q=#{query}&category=#{cat}&page=#{page}", else: ~p"/search?q=#{query}&page=#{page}"
    {:noreply, push_patch(socket, to: path)}
  end

  # Executes the Elasticsearch query and updates assigns accordingly.
  defp do_search(socket, query, opts) do
    search_opts =
      opts
      |> Keyword.put(:per_page, socket.assigns.per_page)
      |> Keyword.reject(fn {_k, v} -> is_nil(v) end)

    socket = assign(socket, loading: true, error: nil, query: query)

    case ProductSearch.search(query, search_opts) do
      {:ok, %{hits: hits, total: total, took_ms: took_ms, aggregations: aggs}} ->
        assign(socket,
          results: hits,
          total: total,
          took_ms: took_ms,
          aggregations: aggs,
          loading: false,
          selected_category: Keyword.get(opts, :category),
          page: Keyword.get(opts, :page, 1)
        )

      {:error, reason} ->
        assign(socket,
          results: [],
          total: 0,
          loading: false,
          error: "Search failed: #{inspect(reason)}"
        )
    end
  end

  @impl true
  def render(assigns) do
    ~H"""
    <div class="min-h-screen bg-gray-50">
      <header class="bg-white shadow-sm">
        <div class="max-w-5xl mx-auto px-4 py-6">
          <h1 class="text-3xl font-bold text-gray-900">Product Search</h1>
          <p class="text-sm text-gray-500 mt-1">Powered by Elasticsearch + Phoenix LiveView</p>
        </div>
      </header>

      <main class="max-w-5xl mx-auto px-4 py-8">
        <%!-- Search form --%>
        <form phx-submit="search" class="flex gap-3 mb-8">
          <input
            type="text"
            name="query"
            value={@query}
            placeholder="Search products..."
            class="flex-1 rounded-lg border border-gray-300 px-4 py-3 text-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
            autofocus
          />
          <button
            type="submit"
            class="px-6 py-3 bg-blue-600 text-white rounded-lg font-semibold hover:bg-blue-700 transition-colors"
          >
            Search
          </button>
        </form>

        <%= if @error do %>
          <div class="mb-6 rounded-lg bg-red-50 border border-red-200 p-4 text-red-700">
            {@error}
          </div>
        <% end %>

        <div class="flex gap-8">
          <%!-- Facets sidebar --%>
          <%= if map_size(@aggregations) > 0 do %>
            <aside class="w-56 shrink-0">
              <h2 class="font-semibold text-gray-700 mb-3">Categories</h2>
              <ul class="space-y-1">
                <%= if @selected_category do %>
                  <li>
                    <button phx-click="clear_category" class="text-sm text-blue-600 hover:underline">
                      ✕ Clear filter
                    </button>
                  </li>
                <% end %>
                <%= for bucket <- get_in(@aggregations, ["categories", "buckets"]) || [] do %>
                  <li>
                    <button
                      phx-click="filter_category"
                      phx-value-category={bucket["key"]}
                      class={"text-sm w-full text-left px-2 py-1 rounded hover:bg-blue-50 #{if @selected_category == bucket["key"], do: "font-bold text-blue-700 bg-blue-50", else: "text-gray-700"}"}
                    >
                      {bucket["key"]}
                      <span class="text-gray-400">({bucket["doc_count"]})</span>
                    </button>
                  </li>
                <% end %>
              </ul>
            </aside>
          <% end %>

          <%!-- Results --%>
          <div class="flex-1">
            <%= if @loading do %>
              <div class="flex items-center gap-2 text-gray-500 py-12">
                <svg class="animate-spin h-5 w-5" viewBox="0 0 24 24" fill="none">
                  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"/>
                  <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"/>
                </svg>
                Searching...
              </div>
            <% else %>
              <%= if @total > 0 do %>
                <p class="text-sm text-gray-500 mb-4">
                  {@total} results in {@took_ms}ms
                  <%= if @selected_category do %>
                    in category <span class="font-semibold">{@selected_category}</span>
                  <% end %>
                </p>
              <% end %>

              <ul class="space-y-4">
                <%= for hit <- @results do %>
                  <li class="bg-white rounded-xl border border-gray-200 p-5 shadow-sm hover:shadow-md transition-shadow">
                    <div class="flex items-start justify-between gap-4">
                      <div class="flex-1">
                        <h3 class="text-lg font-semibold text-gray-900">
                          {raw(get_in(hit, ["_highlight", "title", Access.at(0)]) || hit["title"])}
                        </h3>
                        <p class="text-sm text-gray-600 mt-1">
                          {raw(get_in(hit, ["_highlight", "description", Access.at(0)]) || hit["description"])}
                        </p>
                        <div class="flex items-center gap-3 mt-3">
                          <span class="text-xs bg-gray-100 text-gray-600 px-2 py-1 rounded-full">
                            {hit["category"]}
                          </span>
                          <%= for tag <- (hit["tags"] || []) do %>
                            <span class="text-xs bg-blue-50 text-blue-600 px-2 py-1 rounded-full">
                              {tag}
                            </span>
                          <% end %>
                        </div>
                      </div>
                      <div class="text-right shrink-0">
                        <p class="text-xl font-bold text-blue-600">
                          €{:erlang.float_to_binary(hit["price"] || 0.0, decimals: 2)}
                        </p>
                        <p class={"text-xs mt-1 #{if hit["in_stock"], do: "text-green-600", else: "text-red-500"}"}>
                          {if hit["in_stock"], do: "In stock", else: "Out of stock"}
                        </p>
                      </div>
                    </div>
                  </li>
                <% end %>
              </ul>

              <%!-- Pagination --%>
              <%= if @total > @per_page do %>
                <div class="flex items-center justify-center gap-2 mt-8">
                  <%= if @page > 1 do %>
                    <button
                      phx-click="paginate"
                      phx-value-page={@page - 1}
                      class="px-4 py-2 rounded-lg border border-gray-300 text-sm hover:bg-gray-50"
                    >
                      ← Previous
                    </button>
                  <% end %>

                  <span class="px-4 py-2 text-sm text-gray-600">
                    Page {@page} of {ceil(@total / @per_page)}
                  </span>

                  <%= if @page * @per_page < @total do %>
                    <button
                      phx-click="paginate"
                      phx-value-page={@page + 1}
                      class="px-4 py-2 rounded-lg border border-gray-300 text-sm hover:bg-gray-50"
                    >
                      Next →
                    </button>
                  <% end %>
                </div>
              <% end %>

              <%= if @total == 0 and @query != "" do %>
                <div class="text-center py-16 text-gray-500">
                  <p class="text-xl">No results for "<strong>{@query}</strong>"</p>
                  <p class="text-sm mt-2">Try different keywords or clear your filters.</p>
                </div>
              <% end %>
            <% end %>
          </div>
        </div>
      </main>
    </div>
    """
  end
end
```

### 7.3 Seed / Re-index Task

```elixir
# lib/mix/tasks/search.build_index.ex
defmodule Mix.Tasks.Search.BuildIndex do
  @moduledoc """
  Bulk-indexes all products from the database into Elasticsearch.

      mix search.build_index
  """
  use Mix.Task

  @shortdoc "Bulk-index all products into Elasticsearch"

  @impl Mix.Task
  def run(_args) do
    Mix.Task.run("app.start")

    alias MyApp.Search.{Cluster, ProductStore}
    alias MyApp.Product

    index = "products"
    batch_size = 5_000

    IO.puts("Starting bulk index of products...")

    Stream.resource(
      fn -> 0 end,
      fn offset ->
        batch = ProductStore.load(Product, offset, batch_size)
        if batch == [], do: {:halt, offset}, else: {batch, offset + length(batch)}
      end,
      fn _ -> :ok end
    )
    |> Stream.chunk_every(batch_size)
    |> Enum.each(fn batch ->
      Elasticsearch.bulk_index_batch(Cluster, batch, index)
      IO.write(".")
    end)

    IO.puts("\nDone.")
  end
end
```

---

## 8. Key Takeaways

| Question | Answer |
|---|---|
| **Choose Solr when…** | You need proven enterprise document search with Tika, your team knows Solr, or Apache License 2.0 is a hard requirement for all features |
| **Choose Elasticsearch when…** | You need JSON-native API, Kibana analytics, time-series data, kNN/vector search, or you want the simplest HA cluster setup |
| **Elixir + Solr** | Use Tesla/HTTPoison against the JSON API; no specialized library needed |
| **Elixir + Elasticsearch** | Use `elasticsearch-elixir` for cluster management, bulk indexing, and document lifecycle |
| **Phoenix LiveView fit** | Excellent — handle_event triggers real queries, results update instantly over WebSocket without full page reload |
| **Real-time indexing** | Hook into Ecto changesets (`after_create`/`after_update`) to call `ProductSearch.index_product/1` |
| **Vector search** | ES 8.x supports kNN natively; the `embedding` field + `knn_search/2` example above is production-ready |

---

## Related Topics

- [`../rag/vector_databases.md`](../rag/vector_databases.md) — Vector stores and embedding-based retrieval
- [`../embeddings/`](../embeddings/) — How to generate vectors to populate the `embedding` field
- [`../deployment/`](../deployment/) — Scaling Elasticsearch in production (ILM, hot-warm, sharding strategy)
