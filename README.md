# LegacyLens

**AS400/iSeries Modernization Analysis Toolkit**

LegacyLens is a set of AI-powered analysis skills for [Claude Code](https://claude.ai/code) that help solutions architects understand and plan the modernization of AS400/iSeries applications. Given a program name, LegacyLens traces the full call stack, resolves all data dependencies, and produces a structured report -- written in plain language for architects who may not have AS400 experience.

---

## What It Does

LegacyLens analyzes AS400 source code and metadata to produce modernization reports covering:

- **Program Call Stack** -- Full call hierarchy including dynamic calls (variable-based CALL, dispatchers, QCMDEXC), resolved by scanning both cross-reference metadata and source code
- **File Dependencies** -- Every database file used by every program, with logical file (view/index) to physical file (table) resolution
- **Data Areas** -- Global variables and Local Data Area (*LDA) position maps showing cross-program state sharing
- **Temporary Files (QTEMP)** -- Working files created during execution, with lifecycle tracking
- **Message Files** -- Message IDs resolved to actual human-readable text from MSGF JSON exports
- **Platform-Specific Constructs** -- File overrides, dynamic queries, job submissions, error handling, indicator logic, S/36 OCL patterns
- **Modernization Recommendations** -- Each AS400 construct categorized as Replace, Redesign, or Eliminate with specific modern equivalents

## Quick Start

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI, desktop app, or IDE extension
- AS400 source code and metadata (see [Project Structure](#project-structure))

### Usage

Open a terminal in the project root and start Claude Code, then:

```
/legacylens-analyze <ProgramName>
```

Produces a full modernization report saved to `output/<ProgramName>_analysis.md`.

```
/legacylens-trace <ProgramName>
```

Quick call-tree trace output directly to the conversation -- useful for exploration before a full analysis.

### Example

```
/legacylens-analyze TF410CL
```

This will:
1. Locate the TF410CL source file across all client libraries
2. Query the cross-reference metadata for call chains and file usage
3. Read each program's source to detect dynamic calls, data areas, QTEMP usage, and message references
4. Resolve all logical files to their underlying physical files
5. Look up message text from MSGF JSON exports
6. Generate a structured markdown report at `output/TF410CL_analysis.md`

---

## Project Structure

```
LegacyLens/
|
+-- as400/                          # AS400 source code
|   +-- {ClientName}/               # One folder per client
|       +-- {Library}/              # AS400 library (e.g., PRKSLIB, PRKSYNSRC)
|       |   +-- QCLSRC/            # CL programs (.CLP, .CLLE)
|       |   +-- QRPGSRC/           # RPG programs (.RPG)
|       |   +-- QRPGLESRC/         # ILE RPG programs (.RPGLE, .SQLRPGLE)
|       |   +-- QDDSSRC/           # Data definitions (.PF, .LF, .DSPF, .PRTF)
|       |   +-- QSQLSRC/           # SQL source (.SQL, .YSQL)
|       |   +-- QCMDSRC/           # Command definitions (.CMD)
|       |   +-- QMNUSRC/           # Menu definitions
|       |   +-- QPNLSRC/           # Panel group definitions
|       |   +-- QTXTSRC/           # Text/generated source
|       |   +-- ...                 # Other source types as needed
|       +-- msgf/                   # Message file exports (JSON)
|           +-- HSMSGF_*.json       # Application message catalog
|           +-- ...
|
+-- metadata/                       # Cross-reference data
|   +-- xref_export_*.csv           # Program cross-reference (calls, files, data areas)
|   +-- pflf_export_*.csv           # Physical file to logical file mappings
|
+-- output/                         # Generated analysis reports
|   +-- {ProgramName}_analysis.md
|
+-- .claude/                        # Claude Code configuration
|   +-- settings.json               # Team-shared permissions
|   +-- skills/                     # LegacyLens analysis skills
|       +-- legacylens-analyze/     # Full modernization report skill
|       +-- legacylens-trace/       # Quick call-tree trace skill
|
+-- CLAUDE.md                       # Project guide for Claude Code
+-- README.md                       # This file
```

---

## Supported Source Types

| Extension | Type | Description |
|-----------|------|-------------|
| `.CLP`, `.CLLE` | CL Program | Orchestration/scripting programs (like shell scripts) |
| `.RPG` | RPG/400 | Business logic programs (fixed-format, legacy) |
| `.RPGLE` | ILE RPG | Business logic programs (free-format, modern RPG) |
| `.SQLRPG`, `.SQLRPGLE` | Embedded SQL RPG | RPG programs with inline SQL statements |
| `.PF` | Physical File | Database table definitions |
| `.LF` | Logical File | Database view/index definitions |
| `.DSPF` | Display File | Screen/form layout definitions |
| `.PRTF` | Print File | Report layout definitions |
| `.CMD` | Command | Custom command definitions |
| `.OCL`, `.OCL36` | S/36 OCL | System/36 Operation Control Language (oldest layer) |

---

## Metadata Format

### Cross-Reference CSV (`xref_export_*.csv`)

| Column | Description |
|--------|-------------|
| Caller Object | Program making the call or file reference |
| Called Object | Program being called, file being accessed, or data area being used |
| File Usage | Numeric code: 0=Call/binding, 1=Input, 2=Output, 3=Update, 4=Delete, 5=Range, 6=Chain/Seek, 7=Open, 8=Multi-I/O |
| File Type | `*PGM` (program), `*FILE` (database file), `*SRVPGM` (service program/shared library), `*DTAARA` (data area/global variable) |
| Confidence | CONFIRMED, HIGH, MEDIUM, or PENDING |
| Origin | SBFDEV, STATIC_ANALYSIS, or DYNAMIC_PATTERN |
| Location | Library where the object resides |
| Active | Yes/No |

### PF-LF Mapping CSV (`pflf_export_*.csv`)

| Column | Description |
|--------|-------------|
| Base Physical File | The underlying database table |
| Logical File | The view/index built over the table |
| Updated | Last update timestamp |
| Created | Creation timestamp |

### Message File JSON (`msgf/*.json`)

Array of message objects:
```json
[
  {
    "Message ID": "HS00100",
    "Message Text": "Equipment type &1 already exists.",
    "Severity": 0,
    "Fields": [
      { "Field": "&1", "Data Type": "*CHAR", "Length": 5 }
    ],
    "Recovery": "Optional recovery instructions"
  }
]
```

---

## Adding a New Client

1. **Source code**: Place under `as400/{ClientName}/{LibraryName}/{SourceType}/`
2. **Cross-reference**: Export program cross-reference and PF-LF mappings as CSV files into `metadata/`. Use the same column format as existing files.
3. **Message files** (optional): Export MSGF contents as JSON into `as400/{ClientName}/msgf/`
4. **Run analysis**: `/legacylens-analyze <ProgramName>` -- the skills auto-discover the new client's directory structure

---

## How It Works

### Analysis Pipeline

```
  Input: Program Name
         |
         v
  +-------------------+
  | Phase 1: Discovery |  Auto-detect project layout, validate artifacts,
  |                     |  locate the entry point source file
  +-------------------+
         |
         v
  +-------------------+
  | Phase 2: Call Tree |  Build call hierarchy from cross-reference metadata,
  |                     |  then scan source code to resolve dynamic calls
  |                     |  (CALL PGM(&var), dispatchers, QCMDEXC, S/36 OCL)
  +-------------------+
         |
         v
  +-------------------+
  | Phase 3: Source    |  Read each program's source to extract file usage,
  |          Analysis  |  data areas, QTEMP files, messages, indicators,
  |                     |  subroutines, and platform-specific patterns
  +-------------------+
         |
         v
  +-------------------+
  | Phase 4: File      |  Resolve logical files to physical files using
  |          Resolution|  PF-LF metadata; trace file override chains
  +-------------------+
         |
         v
  +-------------------+
  | Phase 5: Report    |  Generate structured markdown report with
  |          Generation|  plain-language explanations of all AS400 terms
  +-------------------+
         |
         v
  +-------------------+
  | Phase 6: Verify    |  Cross-check xref vs source, flag gaps,
  |                     |  document unresolved references
  +-------------------+
         |
         v
  Output: output/{ProgramName}_analysis.md
```

### Dynamic Call Resolution

LegacyLens goes beyond static cross-reference data. It reads source code to resolve:

- **Variable-based CALL**: `CALL PGM(&PGM)` -- traces the variable back to its origin (PARM, CHGVAR, RTVDTAARA)
- **Dispatcher pattern**: Programs that receive a program name as a parameter and execute it -- LegacyLens traces upstream to find what callers actually pass
- **QCMDEXC**: Dynamic command execution API -- reads the command string construction to find embedded CALL PGM() targets
- **String-built names**: Program names constructed via `*TCAT`, `*CAT`, `%SST` string operations
- **Synon/2E naming**: Compiled program names differ from source member names (e.g., Y2SNMGC -> Y2SNMSC.CLP). LegacyLens applies fuzzy matching to find the correct source.

### Report Audience

Reports are written for **modern solutions architects who may not have AS400 experience**. Every AS400 term includes a plain-language explanation in parentheses:

> *"Physical Files (database tables) and Logical Files (views/indexes over tables)"*
>
> *"SBMJOB (submits a program for background/batch execution, like a job queue)"*
>
> *"Indicators *INxx (numbered boolean flags used for screen conditioning and flow control)"*

---

## Team Setup

The repo includes shared Claude Code configuration in `.claude/settings.json` that pre-allows the necessary tool permissions. Team members just need Claude Code installed -- no additional configuration required.

To enable auto mode (skip permission prompts), create a personal `.claude/settings.local.json`:

```json
{
  "permissions": {
    "defaultMode": "auto"
  }
}
```

This file is gitignored and won't affect other team members.

---

## License

Internal use only. Contact repository owner for access and licensing.
