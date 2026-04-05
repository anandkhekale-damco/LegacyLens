# LegacyLens -- AS400 Modernization Analysis

LegacyLens is a set of Claude Code skills that analyze AS400/iSeries programs and produce structured modernization reports for solutions architects.

## Skills

### `/analyze <ProgramName>`

Full modernization analysis. Traces the complete call stack, resolves file dependencies (logical-to-physical file mapping), identifies data areas, QTEMP temporary files, message files, dynamic calls, and AS400-specific constructs. Produces a markdown report in `output/`.

Example: `/analyze TF410CL`

### `/trace <ProgramName>`

Lightweight call-tree trace. Shows the call hierarchy with per-program file and data area counts directly in the conversation. Good for quick exploration before running a full `/analyze`.

Example: `/trace AR300`

## Project Structure

```
as400/{Client}/{Library}/{SourceType}/{Program}.{Extension}
    Source code organized by client, library, and source type.
    Extensions: .CLP, .CLLE, .RPG, .RPGLE, .SQLRPG, .SQLRPGLE,
                .PF, .LF, .DSPF, .PRTF, .CMD, .OCL, .OCL36

as400/{Client}/msgf/*.json
    Message file (MSGF) exports as JSON arrays.
    Used to look up actual message text for message IDs found in source.

metadata/xref_export_*.csv
    Program cross-reference. Columns: Caller Object, Called Object,
    File Usage (0-11), File Type (*FILE, *PGM, *SRVPGM, *DTAARA),
    Confidence, Origin, Location, Active.

metadata/pflf_export_*.csv
    Physical File to Logical File cross-reference.
    Columns: Base Physical File, Logical File, Updated, Created.

output/
    Generated analysis reports ({ProgramName}_analysis.md).
```

## Key Conventions

- The skills auto-discover the project layout -- no hardcoded client names or paths.
- Synon/2E generated systems have program object names that differ from source member names (e.g., xref says Y2SNMGC but source is Y2SNMSC.CLP). The skills handle this with fuzzy name matching.
- Reports are written for non-AS400 architects -- all AS400 terms include plain-language explanations.

## Adding a New Client

1. Place AS400 source under `as400/{ClientName}/{LibraryName}/{SourceType}/`
2. Place cross-reference CSVs in `metadata/` (same column format as existing files)
3. Optionally place MSGF JSON exports in `as400/{ClientName}/msgf/`
4. Run `/analyze <ProgramName>` -- the skills will auto-discover the new client
