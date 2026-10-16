# DevKnowledgeBase

Willkommen in der **DevKnowledgeBase**! Dieses Repository dient als zentralisierte, strukturierte Sammlung von technischem Wissen, Guides und Best Practices für Software-Entwickler. Es deckt ein breites Spektrum ab – von Web-Frameworks über Datenbankarchitekturen und Systemdesign bis hin zu mathematischen Grundlagen und künstlicher Intelligenz.

## More to come on AI Secirity and Embeddings, RAG etc...

---

## 📚 Inhaltsverzeichnis (Alphanumerisch nach Thema)

### 🤖 Artificial Intelligence & LLMs
* **[Agent Design Patterns](#agent-design-patterns)** (`agent_design_patterns.md`)
* **[Large Language Models (LLMs) Grundlagen](#large-language-models-llms-grundlagen)** (`what_is_an_llm.md`)
* **[Prompt Engineering: Zero to Hero](#prompt-engineering-zero-to-hero)** (`zero_to_hero_prompt_engineering.md`)

### 🔑 Authentication & Security
* **[Authentication: Zero to Hero](#authentication-zero-to-hero)** (`zero_to_hero_authentication.md`)
* **[Open APIs & Public Interfaces](#open-apis--public-interfaces)** (`open_apis.md`)

### 🎨 Frontend & Design
* **[Tailwind CSS Guide](#tailwind-css-guide)** (`guide_tailwindcss.md`)
* **[Vue.js Guide](#vuejs-guide)** (`vuejs_guide.md`)

### 🚀 Integration & Tooling
* **[Model Context Protocol (MCP) Fundamentals](#model-context-protocol-mcp-fundamentals)** (`mcp_servers_fundamentals.md`)
* **[Model Context Protocol (MCP) Implementation](#model-context-protocol-mcp-implementation)** (`mcp_servers_implementation.md`)
* **[Vite Build Tool Guide](#vite-build-tool-guide)** (`guide_vite.md`)

### ☕ Java Development
* **[Java Date & Time API](#java-date--time-api)** (`java-datetime.md`)
* **[Java Internal Features & Architecture](#java-internal-features--architecture)** (`java-internal-features.md`)

### 📐 Mathematics
* **[Linear Algebra: Zero to Hero](#linear-algebra-zero-to-hero)** (`linear_algebra_zero_to_hero.md`)

### 🗄️ Relational Databases & Search
* **[PostgreSQL & SQL Advanced Guide](#postgresql--sql-advanced-guide)** (`guide_postgres_sql.md`)
* **[PostgreSQL: Zero to Hero](#postgresql-zero-to-hero)** (`postgres-zero-to-hero.md`)
* **[Search Engines: Solr vs. Elasticsearch (Elixir)](#search-engines-solr-vs-elasticsearch-elixir)** (`solr_vs_elasticsearch_elixir.md`)

### 🛡️ Repository Meta
* **[GitHub Configurations](#github-configurations)** (`GITHUB/`)

---

## 📖 Detaillierte Inhaltsübersicht

### 🤖 Artificial Intelligence & LLMs

#### Agent Design Patterns
* **Datei:** `agent_design_patterns.md`
* **Beschreibung:** Ein umfassender Architektur-Leitfaden über Entwurfsmuster für KI-Agenten. Behandelt autonome Workflows, Werkzeugnutzung (Tool Calling), Planungs- und Reflexionszyklen (Reflection Loops) sowie Multi-Agenten-Systeme zur kollaborativen Problemlösung.

#### Large Language Models (LLMs) Grundlagen
* **Datei:** `what_is_an_llm.md`
* **Beschreibung:** Ein Einsteiger- und Fortgeschrittenen-Guide zur Funktionsweise von Large Language Models. Erklärt die zugrundeliegende Transformer-Architektur, Tokenisierung, Embeddings sowie den Unterschied zwischen Pre-training und Fine-tuning.

#### Prompt Engineering: Zero to Hero
* **Datei:** `zero_to_hero_prompt_engineering.md`
* **Beschreibung:** Ein praxisorientiertes Handbuch zur Optimierung von Prompts für generative KI. Von einfachen Textanweisungen bis zu fortgeschrittenen Techniken wie Few-Shot Prompting, Chain-of-Thought (CoT) und ReAct-Frameworks.

---

### 🔑 Authentication & Security

#### Authentication: Zero to Hero
* **Datei:** `zero_to_hero_authentication.md`
* **Beschreibung:** Ein tiefgehender Leitfaden zur Absicherung moderner Webanwendungen. Behandelt Core-Konzepte wie Session-based Authentication, Token-basierte Verfahren (JWT), OAuth 2.0, OpenID Connect (OIDC) sowie Best Practices zum Hashing von Passwörtern.

#### Open APIs & Public Interfaces
* **Datei:** `open_apis.md`
* **Beschreibung:** Dokumentation und Design-Prinzipien für offene und öffentliche Schnittstellen. Fokussiert auf RESTful-API-Design, API-Versionierung, Ratenbegrenzung (Rate Limiting) sowie Spezifikationsstandards wie OpenAPI / Swagger.

---

### 🎨 Frontend & Design

#### Tailwind CSS Guide
* **Datei:** `guide_tailwindcss.md`
* **Beschreibung:** Cheat Sheet und Referenzhandbuch für das Utility-First-CSS-Framework. Enthält Strategien für responsive Layouts, Grid- und Flexbox-Konfigurationen, Custom Themes in der `tailwind.config.js` sowie Performance-Optimierung durch Purging.

#### Vue.js Guide
* **Datei:** `vuejs_guide.md`
* **Beschreibung:** Entwickler-Leitfaden für Vue.js. Fokus auf das moderne Composition API, State Management (Pinia), Routing (Vue Router) sowie Best Practices für Single Page Applications (SPAs) und komponentenbasierte Softwarearchitektur.

---

### 🚀 Integration & Tooling

#### Model Context Protocol (MCP) Fundamentals
* **Datei:** `mcp_servers_fundamentals.md`
* **Beschreibung:** Einführung in das Model Context Protocol (MCP). Erklärt die theoretischen Grundlagen und Spezifikationen, wie Entwicklungswerkzeuge und IDEs (z. B. Cursor, Claude Desktop) kontextuelle Daten sicher an Sprachmodelle übertragen.

#### Model Context Protocol (MCP) Implementation
* **Datei:** `mcp_servers_implementation.md`
* **Beschreibung:** Ein technischer Implementierungsleitfaden für das Aufsetzen eigener MCP-Server. Enthält konkrete Codebeispiele in Node.js/TypeScript oder Python zur Anbindung lokaler Datenquellen und APIs an KI-Systeme.

#### Vite Build Tool Guide
* **Datei:** `guide_vite.md`
* **Beschreibung:** Eine Anleitung zum modernen Frontend-Build-Tool Vite. Behandelt die Konfiguration, das Hot Module Replacement (HMR) im Development-Server, Rollup-Bundling für die Production sowie die Migration von Webpack.

---

### ☕ Java Development

#### Java Date & Time API
* **Datei:** `java-datetime.md`
* **Beschreibung:** Praxis-Guide zur Handhabung von Datums- und Zeitangaben in modernen Java-Versionen (ab Java 8). Erklärt die Nutzung von `LocalDate`, `ZonedDateTime`, `Duration` sowie den sicheren Umgang mit Zeitzonen zur Vermeidung typischer Bugs.

#### Java Internal Features & Architecture
* **Datei:** `java-internal-features.md`
* **Beschreibung:** Ein tiefer Blick unter die Haube der Java Virtual Machine (JVM). Behandelt Speicherverwaltung (Heap vs. Stack), Garbage Collection-Algorithmen, Just-In-Time (JIT) Compilation sowie neuere Sprachfeatures (Records, Sealed Classes, Virtual Threads).

---

### 📐 Mathematics

#### Linear Algebra: Zero to Hero
* **Datei:** `linear_algebra_zero_to_hero.md`
* **Beschreibung:** Mathematisches Fundament für Softwareentwickler und Data Scientists. Erläutert Vektoren, Matrizen, lineare Transformationen, Eigenwerte und Eigenvektoren – verknüpft mit praktischen Beispielen aus der Computergrafik und dem Machine Learning.

---

### 🗄️ Relational Databases & Search

#### PostgreSQL & SQL Advanced Guide
* **Datei:** `guide_postgres_sql.md`
* **Beschreibung:** Fortgeschrittene SQL-Techniken speziell für PostgreSQL. Deckt komplexe Abfragen, Window Functions, Common Table Expressions (CTEs), JSONB-Datentypen sowie performantes Query-Tuning ab.

#### PostgreSQL: Zero to Hero
* **Datei:** `postgres-zero-to-hero.md`
* **Beschreibung:** Ein Leitfaden, der von den relationalen Datenbank-Basics bis hin zum Produktionsbetrieb führt. Behandelt Datenbankschemata, Indizierungsstrategien (B-Tree, GIN, GiST), Transaktions-Isolationsstufen und Replikation.

#### Search Engines: Solr vs. Elasticsearch (Elixir)
* **Datei:** `solr_vs_elasticsearch_elixir.md`
* **Beschreibung:** Ein Architektur-Vergleich der Suchmaschinen Apache Solr und Elasticsearch, eingebettet in das Ökosystem der Programmiersprache Elixir. Analysiert Performance, Skalierbarkeit, Volltextsuche und Cluster-Management.

---

### 🛡️ Repository Meta

#### GitHub Configurations
* **Ordner:** `GITHUB/`
* **Beschreibung:** Beinhaltet repository-spezifische Konfigurationen, Workflows oder Vorlagen (wie z. B. Issue Templates oder GitHub Actions Workflows) zur Automatisierung des Repositories.
