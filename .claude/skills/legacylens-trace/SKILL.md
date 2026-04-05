---
name: legacylens-trace
description: >
  Quick call-stack trace for an AS400/iSeries program. Uses cross-reference metadata
  as a seed then reads source code to detect dynamic calls (CALL PGM(&var), QCMDEXC,
  dispatcher patterns, S/36 OCL). Outputs the call tree to the conversation with
  per-program file and data area counts. Use when the user asks to "trace", "show call
  stack", "show dependencies", or wants a quick overview before running full /analyze.
argument-hint: "<ProgramName> (e.g., TF410CL, AR300, HP140)"
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
---

# LegacyLens: Quick Call Stack Trace

You are performing a quick call-stack trace of an AS400/iSeries program. This is a
lightweight analysis that focuses on the call tree and provides summary counts of
file and data area dependencies per program. It does NOT produce a full report file.

Output results directly to the conversation.

---

## Step 1: Discovery

1. Glob for `as400/*/*` to find the directory structure.
2. Find the requested program's source file: `find as400/ -iname "{ProgramName}.*"`
3. From the client folder where the program is found, glob for
   `as400/{Client}/metadata/xref_export_*.csv` (use most recent if multiple).
4. **Synon/2E naming fallback**: If a program is not found by exact name, the compiled
   program object name (in xref) may differ from the source member name. Common:
   suffix `GC` → source `SC`, suffix `GR` → source `SR`. Try:
   `find as400/ -iname "{ProgramName_minus_last_char}*"` and
   `find as400/ -iname "{ProgramName_minus_last_2_chars}*"`

If the xref CSV or program source cannot be found, tell the user what's missing and stop.

---

## Step 2: Build Call Tree (Xref + Source Scanning)

### 2.1 Xref Seed

Grep the xref CSV for program calls:
```
grep '^"{PROGRAMNAME}",' as400/{Client}/metadata/xref_export_*.csv
```
Filter to `*PGM` entries with File Usage `0`. Recurse for each called program
(max depth 10, maintain visited set).

**Dedup**: Prefer rows with Origin `"SBFDEV"` or `"STATIC_ANALYSIS"` over empty Origin.

### 2.2 Source-Code Dynamic Call Detection

For each program in the tree, read its source and detect:

**CLP/CLLE:**
- `CALL PGM(&variable)` -- trace variable to PARM, CHGVAR, or RTVDTAARA
- **Dispatcher pattern**: If program receives `&PGM` as PARM and calls `PGM(&PGM)`:
  - Grep xref for callers of this dispatcher
  - Read callers' source to find the literal program name passed
  - Add resolved targets as "via dispatcher {name}"
- `SBMJOB CMD(CALL PGM(&var))` -- trace upstream, note SBMJOB context
- `CALL PGM(QCMDEXC)` -- read &CMD construction for embedded CALL PGM(xxx)
- String-built names: `*TCAT`, `*CAT`, `*BCAT`, `%SST` on program name variables

**RPG/RPGLE:**
- `CALL` with non-literal Factor 2 (field name instead of quoted string)
- `CALLP` with `%PADDR` procedure pointers

**OCL/OCL36:**
- `// LOAD pgmname` -- program load = CALL
- `// RUN` -- program execution
- `CALLSUBR SUBR($name)` -- S/36 subroutine calls

### 2.3 Classify Each Edge

- `xref-confirmed` -- in both xref and source
- `source-only` -- in source but not xref
- `xref-only` -- in xref but not in source scan
- `dynamic-resolved` -- variable call resolved to literal
- `dynamic-unresolved` -- could not resolve
- `via-dispatcher` -- resolved through upstream caller analysis

---

## Step 3: Gather Summary Counts

For each program in the call tree, grep the xref for quick summary counts:
```
grep '^"{PROGRAMNAME}",' as400/{Client}/metadata/xref_export_*.csv
```

Count by File Type:
- `*FILE` entries (with usage code breakdown: Input/Output/Update/Delete)
- `*DTAARA` entries (list the data area names)
- `*SRVPGM` entries

---

## Step 4: Output to Conversation

Display the results directly (do NOT write to a file). Format:

### Call Tree
```
{ProgramName} ({SourceType})
  +-- CalledProgram1 ({Type}) [xref-confirmed]
  |   +-- SubProgram1 ({Type}) [xref-confirmed]
  |   +-- SubProgram2 ({Type}) [dynamic-resolved, via dispatcher OMNBPCLP]
  +-- CalledProgram2 ({Type}) [source-only]
  +-- &UNKNOWN ({Type}) [dynamic-unresolved: CALL PGM(&PGM) received as PARM]
```

### Summary Per Program

| Program | Source Type | Files (I/O/U/D) | Data Areas | SRVPGMs | Notes |
|---------|-----------|-----------------|------------|---------|-------|

Where I/O/U/D = count of Input/Output/Update/Delete file references.

### Quick Stats
- Total programs in call tree: {N}
- Total unique files referenced: {N}
- Total data areas referenced: {N}
- Dynamic calls resolved: {N}
- Dynamic calls unresolved: {N}
- Programs without source: {N}

If the user wants the full analysis with file details, message text lookup, QTEMP
analysis, and modernization notes, suggest they run `/analyze {ProgramName}`.
