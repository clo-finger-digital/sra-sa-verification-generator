# Security Report Automation Pipeline Hub (Single-System / Non-Batch)

An integrated, browser-native client-side WebAssembly (WASM) engine for automating the compilation and generation of single-system **Verification Reports**, **Security Audit (SA) Reports**, and **Security Risk Assessment (SRA) Reports**.

This pipeline processes a single system's Work Assignment Brief (`.docx`), Vulnerability Follow Up Plan (`.xlsx`), and Source Code Review Log (`.xlsx`) to automatically inject metadata, findings, and metrics into pre-formatted standard Word document templates (`.docx`) without requiring a backend server or remote API.

---

## 📋 Table of Contents
- [Architecture Overview](#-architecture-overview)
- [Required Template Files](#-required-template-files)
- [System Features](#-system-features)
- [Code Structure & Breakdown](#-code-structure--breakdown)
  - [UI & Navigation](#1-ui--navigation)
  - [WebAssembly & Pyodide Engine Initialization](#2-webassembly--pyodide-engine-initialization)
  - [File I/O & Template Fetch Helpers](#3-file-io--template-fetch-helpers)
  - [WAB Auto-Extraction Engine (Metadata & Systems)](#4-wab-auto-extraction-engine-metadata--systems)
  - [Pipeline 1: Verification Report Execution](#5-pipeline-1-verification-report-execution)
  - [Pipeline 2: Security Audit (SA) Report Execution](#6-pipeline-2-security-audit-sa-report-execution)
  - [Pipeline 3: Integrated SRA Report Execution](#7-pipeline-3-integrated-sra-report-execution)
- [Data Parsing & Extraction Logic](#-data-parsing--extraction-logic)

---

## 🏗 Architecture Overview

The single-system pipeline operates completely client-side in the browser:
1. **Frontend**: Tailwind CSS with tabbed navigation for each individual report type.
2. **WASM Runtime**: **Pyodide** executes CPython compiled to WebAssembly.
3. **In-Memory Environment**: **openpyxl** parses single uploaded target Excel workbooks, and **python-docx** manipulates standard Word document templates inside Pyodide's virtual Emscripten file system (`pyodideModule.FS`).
4. **Export**: Compiled single-system documents are output directly as Blob downloads in the user's browser.

---

## 📁 Required Template Files

Ensure the following base standard Word templates exist in your repository root folder (or host root):

| Template Filename | Pipeline Usage |
| :--- | :--- |
| `verification template.docx` | Single-System Verification Report Pipeline |
| `SA template.docx` | Single-System Security Audit (SA) Report Pipeline |
| `new SRA template.docx` | Single-System Security Risk Assessment (SRA) Report Pipeline |

---

## 🔥 System Features

- **Focused Single-System Ingestion**: Tailored for generating individual system assessment reports with strict filename mapping and high precision.
- **WAB Auto-Parsing**: Automatically detects Testee Organization names and Major Information System names directly from the Work Assignment Brief (`.docx`) title block and body.
- **Finding Clustering & Deduplication**: Groups findings by system title, vulnerability name, and threat description to prevent duplicate rows across mapped target assets.
- **Contextual Token & Placeholder Injection**: Replaces placeholder keys across paragraphs, headers, footers, and table cells.
- **Automated Domain Controls & Summary Tallies**: Calculates High, Medium, Low, and AOI counts across 14 security domain controls and updates summary tables.

---

## 🔎 Code Structure & Breakdown

### 1. UI & Navigation

- **`switchTab(evt, tabId)`**:
  Toggles visual active states for navigation buttons and displays the selected tab content block (`tab-verification`, `tab-sa`, or `tab-sra`).

### 2. WebAssembly & Pyodide Engine Initialization

- **`initPyodideEngine()`**:
  - Bootstraps the WASM runtime (`loadPyodide()`).
  - Configures `micropip` and loads `pandas`.
  - Installs runtime dependencies (`openpyxl`, `python-docx`).
  - Enables execution buttons once the sandbox environment is active.
  - Registers WAB auto-extract event listeners.

- **Console Loggers (`logToVerificationConsole`, `logToSaConsole`, `logToSraConsole`)**:
  Redirects execution status logs to terminal output windows in the active tab UI.

### 3. File I/O & Template Fetch Helpers

- **`getFileBytes(elementId)`**: Reads a single uploaded file as a `Uint8Array`.
- **`fetchTemplateFile(filename)`**: Resolves template locations across relative, absolute, and root server routes to prevent HTTP 404 deployment errors.

### 4. WAB Auto-Extraction Engine (Metadata & Systems)

- **`attachWabAutoExtractListener(inputId, prefix, logFunction)`**:
  Attaches an `onchange` event listener to WAB input controls. Executes a regex parsing script in Pyodide (`extractScript`):
  - **Title Block Parsing**: Extracts Testee department names (e.g., *Education Bureau*).
  - **Abbreviation Search**: Searches body text for organizational acronyms (e.g., *EDB*).
  - **System Extraction**: Parses system names and abbreviations (e.g., *SEMIS*), excluding generic acronyms (*SRAA*, *SOA*, etc.).
- **`updateUIFields(prefix, deptName, deptAbbr, systemsJson)`**:
  Populates form fields and updates the dropdown selector with extracted systems.
- **`handleSystemSelect(prefix)`**:
  Automatically fills system name and abbreviation inputs when a user selects a detected system from the dropdown list.

---

### 5. Pipeline 1: Verification Report Execution

**`executeVerificationPipeline()`**
1. Gathers UI field inputs and reads uploaded WAB, Follow Up Plan, and Code Review files.
2. Fetches `verification template.docx` and writes all binary files into the Pyodide Virtual File System.
3. Executes embedded CPython logic:
   - **`BrowserLogRedirect`**: Captures Python `stdout`/`stderr` and routes it to the UI console.
   - **`format_asset_address(ip_raw, port_raw, default_proto)`**: Standardizes asset locations into `IP/Protocol/Port` formats (e.g., `192.168.1.1tcp80`).
   - **`parse_and_cluster_sources(xlsx_paths, prefix_filter, global_metrics)`**:
     - Processes the submitted target spreadsheet.
     - Filters observations by prefix (`V` for Vulnerability Assessment, `A` for Penetration Testing, `C` for Code Review).
     - Groups unique findings and tallies risk ratings into `global_metrics`.
   - **WAB Content Parser**: Extracts Scope of Services, Objectives, and Security Requirements via section boundary scanning.
   - **Placeholder Replacement**:
     - Scans document paragraphs, headers, footers, and table cells for placeholder keys (`*company name*`, `*information system name*`, dates, etc.).
     - Injects extracted scope and objectives as `List Bullet` paragraphs.
     - Appends findings rows to target tables using `apply_text_styling_and_borders()`.
     - Populates status and metric summary tables.
     - Performs a clean-up pass to remove stray asterisks.
4. Reads the generated output file (`Generated_Verification_Report_Final.docx`) and initiates an automated download.

---

### 6. Pipeline 2: Security Audit (SA) Report Execution

**`executeSecurityAuditPipeline()`**
1. Gathers input metadata and writes the uploaded WAB file and `SA template.docx` to Pyodide FS.
2. Executes Python logic:
   - **`BrowserRedirectStream`**: Directs Python execution logs to the SA console UI.
   - **WAB Boundary Parser**: Parses paragraph blocks to capture Section 4.1.1 Environment Descriptions, Scope, Objectives, and Government Security Standards.
   - **Replacement Engine (`process_token_replacements`)**:
     - Updates company, system, and testee tokens across all document sections.
     - Updates document control metadata (Creation/Revision tables).
     - Injects extracted WAB security requirements under Section 8.
     - Performs a clean-up pass to remove stray asterisks.
3. Exports and downloads `SA_Report_<ABBR>_Generated.docx`.

---

### 7. Pipeline 3: Integrated SRA Report Execution

**`executeSraPipeline()`**
1. Loads user inputs, WAB, Follow Up Plan, and Code Review log into Pyodide FS alongside `new SRA template.docx`.
2. Executes Python logic:
   - **`normalize_domain_key(text)`**: Maps arbitrary domain string labels from Excel sheets to one of 14 standard ISO/S17 domains (e.g., `management`, `policy`, `access`, `operations`, `development`, `compliance`).
   - **`parse_excel_followup_source(...)`**:
     - Maps each finding to specific security control domains.
     - Updates the 14-domain metric matrix (`domain_matrix`).
   - **`build_total_vulnerabilities_string(...)`**: Constructs human-readable summary statements (e.g., *"2 high, 1 medium, and 3 low risk"*).
   - **Document Generation**:
     - Replaces metadata tokens across body, headers, footers, and tables.
     - Injects system description paragraphs from WAB section 4.1.1.
     - Populates Vulnerability, Penetration Test, and Code Review tables.
     - Maps risk metrics into the 14 Domain Assessment matrix table.
     - Performs a clean-up pass to remove stray asterisks.
3. Exports and downloads `SRA_Report_<ABBR>_Generated.docx`.

---

## 🛠 Data Parsing & Extraction Logic Summary
+------------------------------------+
|  User Uploads (WAB, XLSX File)     |
+------------------------------------+
|
v
+------------------------------------+
|   Pyodide WASM Virtual FS Engine   |
+------------------------------------+
|
+-----------+-----------+
|                       |
v                       v
+------------------+   +--------------------+
|  WAB Docx Parser |   | Excel openpyxl     |
|  - Title Block   |   | - Single System    |
|  - System Names  |   | - Deduplication    |
|  - Scope & Req's |   | - Domain Mapping   |
+------------------+   +--------------------+
|                       |
+-----------+-----------+
|
v
+------------------------------------+
| python-docx Template Injection     |
| - Paragraphs, Headers, Footers     |
| - Dynamic Table Row Insertion      |
| - Style & Border Application       |
+------------------------------------+
|
v
+------------------------------------+
|    Client-Side Blob Download       |
+------------------------------------+
