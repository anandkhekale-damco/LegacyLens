# LegacyLens -- AS400 Modernization Analysis

LegacyLens is a set of Claude Code skills that analyze AS400/iSeries programs and produce structured modernization reports for solutions architects.

## Skills

### `/legacylens-analyze <ProgramName>`

Full modernization analysis. Traces the complete call stack, resolves file dependencies (logical-to-physical file mapping), identifies data areas, QTEMP temporary files, message files, dynamic calls, and AS400-specific constructs. Produces a markdown report in `output/`.

Example: `/legacylens-analyze TF410CL`

### `/legacylens-trace <ProgramName>`

Lightweight call-tree trace. Shows the call hierarchy with per-program file and data area counts directly in the conversation. Good for quick exploration before running a full `/legacylens-analyze`.

Example: `/legacylens-trace AR300`

## Project Structure

```
as400/{Client}/                     # One folder per client (self-contained)
    {Library}/{SourceType}/         # Source code by library and type
        {Program}.{Extension}       # Extensions: .CLP, .CLLE, .RPG, .RPGLE,
                                    # .SQLRPG, .SQLRPGLE, .PF, .LF, .DSPF,
                                    # .PRTF, .CMD, .OCL, .OCL36
    metadata/                       # Client-specific cross-reference data
        xref_export_*.csv           # Program cross-reference (calls, files, data areas)
        pflf_export_*.csv           # Physical file to logical file mappings
    msgf/                           # Client-specific message file exports (JSON)
        *.json

output/                             # Generated analysis reports
    {ProgramName}_analysis.md
```

## Key Conventions

- The skills auto-discover the project layout -- no hardcoded client names or paths.
- All metadata (xref, pflf) and message files live under the client folder alongside the source code.
- Synon/2E generated systems have program object names that differ from source member names (e.g., xref says Y2SNMGC but source is Y2SNMSC.CLP). The skills handle this with fuzzy name matching.
- Reports are written for non-AS400 architects -- all AS400 terms include plain-language explanations.

## Adding a New Client

1. Create `as400/{ClientName}/`
2. Place AS400 source under `as400/{ClientName}/{LibraryName}/{SourceType}/`
3. Place cross-reference CSVs in `as400/{ClientName}/metadata/`
4. Optionally place MSGF JSON exports in `as400/{ClientName}/msgf/`
5. Run `/legacylens-analyze <ProgramName>` -- the skills will auto-discover the new client
