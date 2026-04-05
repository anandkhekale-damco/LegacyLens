---
name: legacylens-analyze
description: >
  Analyze an AS400/iSeries program for modernization. Produces a full dependency report
  with call stack (including dynamic call resolution), file usage with LF-to-PF mapping,
  data areas, QTEMP files, message files, S/36 OCL patterns, and AS400-specific constructs.
  Use when the user asks to "analyze", "assess", "review for modernization", or provides
  an AS400 program name for modernization analysis.
argument-hint: "<ProgramName> (e.g., TF410CL, AR300, HP140)"
allowed-tools: ["Read", "Write", "Glob", "Grep", "Bash", "Agent"]
---

# LegacyLens: AS400 Modernization Analysis

You are performing a comprehensive modernization analysis of an AS400/iSeries program.
The user has provided a top-level program name. Your job is to trace the full call stack,
resolve all dependencies, and produce a structured report for a modern solutions architect.

Follow these phases exactly. Do NOT skip phases. Do NOT hardcode client names, library
names, or paths -- discover everything dynamically.

### STRICT RULES -- FULL COVERAGE IS MANDATORY

These rules are non-negotiable. Violation produces an incomplete report that is unusable
for modernization planning.

1. **EVERY program in the call tree MUST have its source file located, read, and analyzed
   -- regardless of depth.** There is no concept of "Not searched" or "analysis deferred."
   Depth-2, depth-3, depth-4 programs receive the same analysis as depth-1 programs.

2. **Source resolution happens for ALL programs in a single pass.** After building the
   complete call tree (Phase 2), immediately find source files for every program at every
   depth before starting Phase 3. Use parallel agents to handle volume.

3. **SBMJOB targets and dynamic call targets are first-class programs.** Programs
   discovered via SBMJOB commands (in message text, CLP source, or QCMDEXC patterns)
   and programs resolved from dynamic calls (variable-based CALL) must also have their
   source located and analyzed. Their downstream calls (e.g., print programs called from
   a SBMJOB CLP) must also be traced.

4. **The Call Details table must never contain "Not searched" or "Not analyzed".** Every
   row must show one of:
   - Source type + "Yes" (source found and analyzed)
   - "No (Synon/2E shipped runtime)" -- for standard Synon objects without source
   - "No (IBM system API)" -- for Q* system programs
   - "No (not in source repository)" -- source genuinely missing, flagged for follow-up
   Print programs, report programs, and batch programs receive the same treatment as
   interactive programs -- find the source, read it, extract F-specs/calls/data areas.

5. **File resolution covers ALL depths.** Every *FILE reference from every program in the
   call tree must be resolved through the PFLF CSV. Do not resolve only depth-1 files.

---

## Phase 1: Discovery & Validation

### 1.1 Discover Project Layout

1. Glob for `as400/*/*` to find the client/library directory structure.
   Expected pattern: `as400/{Client}/{Library}/{SourceType}/{Program}.{Extension}`
   Each client folder may also contain:
   - `as400/{Client}/metadata/` for cross-reference CSVs
   - `as400/{Client}/msgf/` for message file exports

2. Glob for `as400/*/metadata/xref_export_*.csv` and `as400/*/metadata/pflf_export_*.csv`.
   If multiple files match per client, use the most recent (by filename timestamp).

3. Glob for `as400/*/msgf/*.json` to find message file JSON exports.

4. Record all discovered paths for use in subsequent phases.
   Note: All metadata and source are scoped to the client folder where the requested
   program is found. When the program is located, use that client's metadata and msgf.

### 1.2 Validate Required Artifacts

Check that these exist. For each missing artifact, report its status:

| Artifact | How to Find | Critical? |
|----------|------------|-----------|
| AS400 source tree | `as400/{Client}/` with at least one library containing source files | YES -- stop if missing |
| Program cross-reference | `as400/{Client}/metadata/xref_export_*.csv` | YES -- stop if missing |
| PF-LF cross-reference | `as400/{Client}/metadata/pflf_export_*.csv` | NO -- warn, skip LF-to-PF resolution |
| Message file exports | `as400/{Client}/msgf/*.json` | NO -- warn, skip message text lookup |
| The requested program | Find `{ProgramName}.*` in the source tree | YES -- stop if missing |

If a CRITICAL artifact is missing, STOP and tell the user exactly what is missing and
where it should be placed. Do not proceed with partial critical inputs.

If a non-critical artifact is missing, WARN the user and continue. Note the gap in the
final report's Verification section.

### 1.3 Resolve the Entry Point Program

1. Search for the program: `find as400/ -iname "{ProgramName}.*"` (case-insensitive).
2. **If not found by exact name, apply Synon/2E naming fallback**:
   In Synon/CA 2E generated systems, the compiled program object name (stored in xref)
   often differs from the source member name. Common patterns:
   - Program object suffix `GC` (Generated CLP) → source suffix `SC` (Source CLP)
   - Program object suffix `GR` → source suffix may be `SR` or `R`
   - The base name (before the last 1-2 characters) is usually the same
   
   Fallback searches:
   - Replace the last character: `find as400/ -iname "{ProgramName_minus_last_char}*"`
   - Replace the last 2 chars: `find as400/ -iname "{ProgramName_minus_last_2_chars}*"`
   - If matches are found, read the first 10 lines to confirm the program identity
     from the header (look for `Program name` or the function description).
   
   This is critical for Synon/2E utility programs (Y2*, YDD*, YWR*) where xref stores
   the *PGM object name but the source member uses a different naming convention.
3. Determine source type from extension:
   - `.CLP`, `.CLLE` = CL Program
   - `.RPG` = RPG/400 (fixed-format)
   - `.RPGLE` = ILE RPG (free-format)
   - `.SQLRPG` = RPG with embedded SQL
   - `.SQLRPGLE` = ILE RPG with embedded SQL
   - `.OCL`, `.OCL36` = System/36 OCL
4. If found in multiple libraries, prefer the production library (typically the one
   without "SYN" in the name). Note all locations found.
5. Read the first 50 lines to extract the module overview (system, purpose, programmer, date).

---

## Phase 2: Call Tree Building (Metadata + Source Code Hybrid)

The cross-reference CSV is the starting seed, but you MUST also scan source code to
resolve dynamic calls that the xref cannot capture.

### 2.1 Xref Seed

1. Grep the xref CSV for all rows where the first column (Caller Object) matches the
   program name:
   ```
   grep '^"{PROGRAMNAME}",' as400/{Client}/metadata/xref_export_*.csv
   ```

2. From those rows, extract entries where File Type = `"*PGM"` and File Usage = `"0"`.
   These are direct program CALL references.

3. **Dedup rule**: When duplicate rows exist for the same caller/called pair with
   different Origin values, prefer the row from `"SBFDEV"` or `"STATIC_ANALYSIS"`
   origin (these have populated File Usage codes). Skip rows with empty Origin and
   empty File Type (these are duplicates without usage detail).

4. Recurse: for each called program, repeat steps 1-3. Maintain a **visited set** to
   prevent infinite loops. Maximum recursion depth: 10 levels.

5. Note `"*SRVPGM"` entries (service program bindings) but do NOT recurse into them.
   These are typically system-level bindings (QRNXIE, QRNXIO, QRNXUTIL, QLEAWI, etc.).

### 2.2 Source-Code Dynamic Call Resolution

For EVERY program in the call tree, read its source file and detect these patterns:

#### CLP/CLLE Programs:

**a) Variable-based CALL: `CALL PGM(&variable)`**
- Identify the variable name (e.g., `&PGM`).
- Trace backwards in the source to find where it is set:
  - `PGM PARM(&PGM ...)` -- received as a parameter from the caller
  - `CHGVAR VAR(&PGM) VALUE('LITERAL')` -- set to a literal value
  - `RTVDTAARA ... RTNVAR(&PGM)` -- retrieved from a data area
- If resolved to a literal, add that program to the call tree as `dynamic-resolved`.
- If received as PARM, this is a **dispatcher pattern** -- see below.

**b) Dispatcher Pattern (parameter-passed program names)**
Many AS400 systems use a dispatcher architecture:
1. A CLP receives a program name as PARM: `PGM PARM(&PGM ...)`
2. It then executes: `CALL PGM(&PGM)`

When you detect this pattern:
1. Mark the current program as a "dispatcher".
2. Look UPSTREAM: grep the xref for this dispatcher as the Called Object:
   ```
   grep ',"DISPATCHER_NAME",' as400/{Client}/metadata/xref_export_*.csv
   ```
3. Read the upstream callers' source files to find what literal program name they
   pass to the dispatcher.
4. Add those resolved targets to the call tree with note: "via dispatcher {DispatcherName}".
5. Common dispatcher naming: `*PCLP`, `*UPC`, `*UPR` suffixes.

**c) SBMJOB with dynamic CALL**
- `SBMJOB CMD(CALL PGM(&variable) ...)` -- same upstream-tracing as dispatcher.
- Also note: `SBMJOB CMD(CALL PGM(&CLP) PARM(&PGM ...))` is TWO levels of
  indirection. Trace both `&CLP` and `&PGM`.
- Record SBMJOB parameters (JOBQ, JOBD, JOB name) for modernization notes.

**d) QCMDEXC Dynamic Execution**
- `CALL PGM(QCMDEXC) PARM(&CMD 512)` executes a command string dynamically.
- Read the preceding `CHGVAR` statements that build the `&CMD` variable.
- If `&CMD` contains `CALL PGM(xxx)`, add `xxx` to the call tree.
- Flag as "QCMDEXC dynamic execution".

**e) String-built program names**
- `*TCAT`, `*CAT`, `*BCAT` or `%SST` used to construct program names.
- Example: `CHGVAR VAR(&PGMO) VALUE(&PGM *TCAT '$')` appends `$` to the name.
- Document the construction pattern and attempt to resolve if possible.

#### RPG/RPGLE Programs:

**f) CALL with variable Factor 2**
- In fixed-format RPG: `C     CALL  FIELDNAME` where FIELDNAME is not a quoted literal.
- Trace the field to find where the program name is set (MOVEL, EVAL, parameter).

**g) Procedure pointer calls**
- `CALLP` with `%PADDR` or pointer-based invocation.
- Document but flag as requiring manual verification.

#### OCL/OCL36 Programs:

**h) S/36 OCL commands**
- `// LOAD pgmname` = program load (equivalent to CALL)
- `// RUN` = program execution
- `// FILE NAME-filename` = file declaration
- `CALLSUBR SUBR($name)` = subroutine call (dollar-sign prefix is S/36 convention)
- These represent the oldest migration layer. Flag all for modernization attention.

### 2.3 Merge & Classify

Combine the xref-derived tree with source-discovered calls. Classify each call edge:

| Classification | Meaning |
|---------------|---------|
| `xref-confirmed` | Found in both xref and source code |
| `source-only` | Found in source code but NOT in xref |
| `xref-only` | In xref but not found during source scan |
| `dynamic-resolved` | Variable-based call resolved to a literal target |
| `dynamic-unresolved` | Variable-based call that could not be resolved |
| `via-dispatcher` | Resolved by tracing upstream through a dispatcher program |

---

## Phase 3: Source Analysis

For each program in the call tree, read the source file and extract:

### CLP/CLLE:
- `CALL PGM(xxx)` -- all program calls (static and dynamic)
- `OVRDBF FILE(x) TOFILE(y)` -- file overrides (actual file = y)
- `OPNQRYF` / `OPNSQLF` -- dynamic query patterns (document the selection criteria)
- `CHGDTAARA` / `RTVDTAARA` / `CRTDTAARA` -- data area operations
- `SBMJOB` -- submitted jobs (document CMD, JOBQ, JOBD)
- `CRTDUPOBJ ... TOLIB(QTEMP)` / `CRTDUPOBJL` -- QTEMP file creation
- `DLTF` / `CLRPFM` targeting QTEMP -- QTEMP cleanup
- `MONMSG MSGID(xxx)` -- error monitoring
- `SNDPGMMSG MSGID(xxx) MSGF(yyy)` -- message sending (extract both MSGID and MSGF)
- `SNDUSRMSG MSGID(xxx) MSGF(yyy)` -- user message sending
- `RCVMSG ... MSGF(yyy)` -- message receiving
- `RTVMSG MSGID(xxx) MSGF(yyy)` -- message retrieval
- `OVRMSGF FILE(xxx) TOFILE(yyy)` -- message file override
- `%SST(*LDA pos len)` -- local data area position mapping
- `CHGDTAARA DTAARA(*LDA ...)` -- LDA writes
- `SNDRCVF` -- display file I/O
- `CRTDTAQ` / `SNDDTAQ` / `RCVDTAQ` -- data queue operations
- `SNDMSG` / `SNDBRKMSG` -- message queue operations

### RPG/RPGLE:
- **F-specs**: File declarations. Parse:
  - File name (positions 7-16 in fixed-format, or `DCL-F` in free-format)
  - File type: I=Input, U=Update, O=Output, C=Combined
  - File designation: F=Full procedural, E=Externally described
  - K=Keyed access
  - DISK/WORKSTN/PRINTER/SPECIAL device
  - SFILE() for subfile declarations
  - RENAME() and PREFIX() clauses
- `CALL` / `CALLP` -- program calls
- `CHAIN` / `READ` / `READP` / `READE` / `READPE` -- file read operations
- `WRITE` / `UPDATE` / `DELETE` -- file modification operations
- `EXFMT` / `EXSR` -- display file format and subroutine execution
- `*INxx` indicators -- document which indicators are used and their purpose
- `BEGSR` / `ENDSR` -- subroutine structure
- `MOVE` / `MOVEL` / `MOVEA` -- legacy data movement operations
- D-spec `msgfil` or `MSGFIL` field with `INZ('msgfname')` -- message file reference
- I-spec literal initialization for MSGFIL (e.g., `I I 'HSMSGF' 1 10 MSGFIL`)
- `CALLSUBR SUBR($name)` -- S/36 legacy subroutine patterns

### SQLRPG/SQLRPGLE (additionally):
- `EXEC SQL` / `/EXEC SQL` -- embedded SQL statements
- `DECLARE ... CURSOR` -- cursor declarations
- `SELECT`, `INSERT`, `UPDATE`, `DELETE` -- SQL DML with table names
- `FROM`, `JOIN` -- table references in queries

### OCL/OCL36:
- `// LOAD pgmname` -- program load (= CALL)
- `// RUN` -- program execution
- `// FILE NAME-filename,RECORDS-n,RETAIN-x,PACK-y` -- file declarations
- `// PRINTER` / `// FORMS` -- print control
- `$COPY`, `$LABEL`, `$MAINC` -- S/36 subroutine patterns

### EBCDIC Detection:
If a source file contains non-printable characters or cannot be read as text, note it
as "EBCDIC-encoded, source analysis not available" and rely solely on xref metadata.

---

## Phase 3b: Message File Resolution

For each MSGID referenced in source code (from SNDPGMMSG, SNDUSRMSG, ERRMSGID, MSGFIL
fields, etc.):

1. Identify which MSGF the message belongs to (from the `MSGF()` parameter or MSGFIL
   variable value in the source).

2. Look up the message in the `msgf/` directory JSON files. The JSON files follow the
   naming pattern `{MSGF_NAME}_*.json` and contain arrays of objects:
   ```json
   {
     "Message ID": "HS00100",
     "Message Text": "Equipment type &1 already exists.",
     "Severity": 0,
     "Fields": [{"Field": "&1", "Data Type": "*CHAR", "Length": 5}],
     "Recovery": "optional recovery text"
   }
   ```

3. To find the right JSON file, match the MSGF name to the filename prefix
   (e.g., MSGF `HSMSGF` maps to `HSMSGF_*.json`).

4. Extract: Message Text, Severity, substitution Fields (type + length), and Recovery.

5. If MSGF JSON files are not available, note "MSGF exports not found -- message text
   not available" and document only the MSGID and MSGF name.

---

## Phase 4: File Resolution

### 4.1 Collect File References

For every program in the call tree, grep the xref CSV for `*FILE` entries:
```
grep '^"{PROGRAMNAME}",' as400/{Client}/metadata/xref_export_*.csv | grep '"*FILE"'
```

Map File Usage codes to human-readable labels:
- 1 = INPUT (read-only)
- 2 = OUTPUT (write/create)
- 3 = UPDATE (modify existing records)
- 4 = DELETE (remove records)
- 5 = Range operation
- 6 = CHAIN/SEEK (keyed direct access)
- 7 = OPEN/DECLARE (cursor/file initialization)
- 8 = Multi-I/O (combined operations)
- 9 = Control operation
- 10 = Shared access
- 11 = Advanced operation

### 4.2 Resolve Logical Files to Physical Files

For each file referenced, determine if it is a Logical File (LF) or Physical File (PF):
- Check the file extension in source: `.LF` = logical, `.PF` = physical
- Or grep the PFLF CSV for the file name in the Logical File column:
  ```
  grep ',"FILENAME",' as400/{Client}/metadata/pflf_export_*.csv
  ```
  If found, the first column is the Base Physical File.

### 4.3 Trace File Overrides

In CLP programs, `OVRDBF FILE(X) TOFILE(Y)` means that when a downstream program
opens file X, it actually reads from Y. Document these override chains to identify the
actual physical file being accessed at runtime.

### 4.4 Produce File Tables

Per program:
| File Name | PF/LF | Base Physical File | Usage | Access Pattern |

Consolidated across entire call tree:
| Physical File | Source Found | Programs Using | Operations |

---

## Phase 5: Report Generation

Write the report to `output/{ProgramName}_analysis.md`.

IMPORTANT: The report audience is modern solutions architects who may NOT have AS400
experience. Every AS400-specific term MUST be followed by a brief plain-language
explanation in parentheses on first use. Examples:
- "Service Programs (shared libraries of reusable procedures, similar to .dll/.so)"
- "Data Areas (system-level global variables stored outside programs)"
- "QTEMP (a per-job temporary library, analogous to temp tables or /tmp files)"
- "SBMJOB (submits a program for background/batch execution, like a job queue)"
- "Physical File / PF (a database table)"
- "Logical File / LF (a database view/index over a physical file)"
- "Display File / DSPF (a screen/form definition, similar to an HTML form template)"
- "Print File / PRTF (a report layout definition)"
- "OVRDBF (file override -- runtime redirection of file access, like dependency injection)"
- "OPNQRYF/OPNSQLF (dynamic query -- builds a filtered view at runtime, like a parameterized SQL query)"
- "MONMSG (error trap -- catches specific error codes, similar to try/catch)"
- "Message File / MSGF (a catalog of predefined messages with substitution parameters, like i18n resource bundles)"
- "CALL PGM (invokes another program, equivalent to a function/API call)"
- "*LDA / Local Data Area (a 1024-byte shared memory block scoped to the job, used for implicit parameter passing)"
- "Indicators (*INxx) (boolean flags set by position number, used for screen conditioning and flow control)"
- "MOVE/MOVEL (data assignment operations -- MOVEL=left-justified, MOVE=right-justified)"
- "EXFMT (write screen + wait for user input, like rendering a form and awaiting submit)"
- "Subfile (a scrollable list/grid of records on a 5250 terminal screen, like a data table component)"
- "CHAIN (keyed random read -- like SELECT WHERE primary_key = value)"
- "READ/READP (sequential read forward/backward through records)"
- "SNDPGMMSG (sends a message to a program message queue, like throwing/logging an error)"
- "QCMDEXC (executes a command string dynamically, like eval() or exec())"
- "CLP/CLLE (Command Language Program -- a scripting/orchestration program, like a shell script)"
- "RPG/RPGLE (Report Program Generator -- the primary business logic language on AS400)"
- "F-specs (file declarations in RPG, defining which database files/screens the program uses)"
- "BEGSR/ENDSR (subroutine boundaries -- internal procedures within an RPG program)"
- "Record format (a named set of fields within a file, like a row schema/struct)"
- "S/36 OCL (System/36 Operation Control Language -- the oldest scripting layer, predating CLP)"

Use this structure:

```markdown
# LegacyLens Modernization Analysis: {ProgramName}

**Generated**: {date}
**Entry Point**: {ProgramName} ({SourceType with explanation})
**Source Location**: {path}

---

## 1. Module Overview

{Extracted from source header: system name, program purpose, original programmer,
creation date, key modification history. Explain what the module does in plain
business terms.}

---

## 2. Program Call Stack

{Brief explanation: "On AS400, applications are built as chains of programs that call
each other. This section maps the complete call hierarchy starting from the entry point."}

### 2.1 Call Tree

{Indented tree view showing the full call hierarchy}

### 2.2 Call Details

| # | Program | Source Type | Source Found | Call Type | Depth |
|---|---------|------------|-------------|-----------|-------|

Call Type values: xref-confirmed, source-only, xref-only, dynamic-resolved,
dynamic-unresolved, via-dispatcher

### 2.3 Service Program Bindings (Shared Libraries)

{Explain: "Service Programs are shared libraries of reusable procedures, similar to
.dll or .so files. They are bound at compile time, not called at runtime."}

| Service Program | Purpose | Bound By |
|----------------|---------|----------|

---

## 3. File Usage by Program

{Brief explanation: "AS400 uses Physical Files (database tables) and Logical Files
(views/indexes over tables). Programs declare which files they use and how
(read, write, update, delete). Display Files define screen layouts."}

### 3.1 {ProgramName}
| File | Type (PF/LF/DSPF) | Base Table (if view) | Usage | Access Pattern |
|------|-------------------|---------------------|-------|----------------|

{Repeat for each program in the call tree}

### 3.N Consolidated Database Table List

| Physical File (Table) | Views/Indexes Used | Programs Using | Operations |
|----------------------|-------------------|----------------|------------|

---

## 4. Data Areas (Global Variables)

{Explain: "Data Areas are system-level global variables stored outside of programs.
They are used for cross-program configuration and state sharing. The Local Data Area
(*LDA) is a 1024-byte per-job memory block used for implicit parameter passing
between programs -- like session-scoped global state."}

| Data Area | Type (Global/Local) | Programs Referencing | Purpose |
|-----------|-------------------|---------------------|---------|

### 4.1 Local Data Area (*LDA) Position Map

{Explain: "The *LDA is a fixed 1024-byte buffer where different programs read/write
specific byte positions by convention. There is no formal schema -- the layout is
maintained by programmer agreement."}

| Position | Length | Value/Field | Set By | Read By |
|----------|--------|-------------|--------|---------|

---

## 5. Temporary Work Files (QTEMP Library)

{Explain: "QTEMP is a per-job temporary library (like /tmp or temp tables). Programs
create working copies of database files here for intermediate processing. These files
are automatically deleted when the job ends."}

| Temp File | Copied From (Template) | Created By | Used By | Purpose |
|-----------|----------------------|------------|---------|---------|

---

## 6. Platform-Specific Constructs

{Explain: "This section documents patterns that are specific to the AS400 platform and
will need special attention during modernization. Each subsection explains what the
construct does and why it exists."}

### 6.1 File Overrides & Dynamic Queries

{Explain: "File Overrides (OVRDBF) redirect file access at runtime -- like dependency
injection for database files. Dynamic Queries (OPNQRYF/OPNSQLF) build filtered views
on the fly, similar to parameterized SQL queries."}

| Program | Command | Description | Target | Selection Criteria |
|---------|---------|------------|--------|--------------------|

### 6.2 Background Job Submissions (SBMJOB)

{Explain: "SBMJOB submits a program for asynchronous/batch execution, similar to
publishing a message to a job queue or scheduling a background worker."}

| Program | Submitted Program | Job Queue | Job Description | Parameters |
|---------|------------------|-----------|-----------------|------------|

### 6.3 Error Handling (MONMSG)

{Explain: "MONMSG is the AS400 error trap mechanism, similar to try/catch. It catches
specific system message IDs and executes recovery actions."}

| Program | Error Code(s) | Action Taken |
|---------|--------------|-------------|

### 6.4 Message Files & User Messages

{Explain: "Message Files (MSGF) are catalogs of predefined messages with substitution
parameters, similar to i18n resource bundles or error code registries. Programs
reference messages by ID rather than hardcoding text."}

| Message File | Message ID | Message Text | Severity | Referenced By | Context |
|-------------|-----------|-------------|----------|--------------|---------|

### 6.5 Dynamic Program Calls & Runtime Command Execution

{Explain: "Some calls resolve the target program at runtime from a variable rather than
a hardcoded name. QCMDEXC is an API that executes an arbitrary command string -- like
eval() or subprocess.exec()."}

| Source Program | Target | Resolution | Pattern Description |
|---------------|--------|-----------|---------------------|

### 6.6 S/36 Legacy Patterns

{Explain: "System/36 (S/36) was the predecessor to AS400. Some codebases retain S/36
conventions like $-prefixed subroutine names and OCL (Operation Control Language)
commands. These represent the oldest code layer."}

| Program | Pattern | Description |
|---------|---------|-------------|

### 6.7 Indicator-Based Logic

{Explain: "AS400 RPG uses numbered boolean flags (*IN01 through *IN99) instead of named
variables for screen conditioning and flow control. Each indicator is a single bit
identified only by its number. This is one of the most significant patterns to redesign."}

| Program | Indicator(s) | Purpose |
|---------|-------------|---------|

### 6.8 Legacy Data Movement (MOVE/MOVEL)

{Explain: "MOVE and MOVEL are RPG assignment operations that copy data between fields
with implicit type conversion. MOVEL is left-justified, MOVE is right-justified.
These have no direct modern equivalent -- they rely on fixed-position field layouts."}

| Program | Operation | From | To | Notes |
|---------|-----------|------|----|-------|

---

## 7. Modernization Recommendations

Categorize each platform-specific construct found above:

### 7.1 Replace (direct modern equivalent exists)
{Items that have direct modern counterparts. For each, state the AS400 pattern and
its modern replacement. Example: "OPNQRYF dynamic queries → parameterized SQL",
"Data Areas → application configuration service / environment variables",
"MSGF message files → i18n resource bundles or error code enum"}

### 7.2 Redesign (needs architectural rethink)
{Items requiring fundamentally different approaches in modern architecture. Explain
WHY the AS400 approach won't work and WHAT the modern alternative looks like.
Example: "*LDA position-based parameter passing → explicit API request/response models",
"QTEMP workfile staging → CTEs or temp tables in SQL, or in-memory processing",
"Indicator-based screen logic → state management with named boolean properties"}

### 7.3 Eliminate (AS400-only, no modern equivalent needed)
{Items that exist only because of AS400 platform constraints and serve no purpose
in modern architecture. Explain WHY they can be dropped.
Example: "MONMSG CPF0000 generic trap → modern exception handling is more granular",
"OVRDBF scope management → not needed when using direct SQL connections",
"PUTOVR screen optimization → not applicable to web/API architecture"}

---

## 8. Verification

- **Program Coverage**: {X} of {Y} programs in call tree had source files available
- **Synon/2E Name Mappings**: {list any programs where xref name differed from source member name}
- **Cross-Reference vs Source Concordance**:
  - Calls in source but not in cross-reference: {list}
  - Calls in cross-reference but not found in source: {list}
- **Unresolved File References**: {list of file references not resolved to PF/LF}
- **EBCDIC/Unreadable Sources**: {list}
- **Dynamic Calls Requiring Manual Verification**: {list}
- **Missing Artifacts**: {any non-critical artifacts that were not available}
```

---

## Phase 6: Verification

Before finalizing the report, verify:

1. Every program in the xref call chain was resolved to a source file (or flagged as missing).
2. Every `*FILE` from xref was checked against PFLF for LF-to-PF resolution.
3. The call tree from xref matches CALL statements found in source code.
4. No circular references remain unhandled (the visited set should prevent this).
5. All dynamic calls are either resolved or explicitly flagged as unresolved.
6. The Verification section at the end of the report accurately reflects any gaps.

After writing the report, tell the user where it was saved and provide a brief summary
of key findings (number of programs in call tree, number of files, notable AS400-specific
constructs that will need attention during modernization).
