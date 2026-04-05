---
name: legacylens-init
description: >
  Check and guide the user on what inputs are needed in the as400/ folder for
  /legacylens-analyze to work. Validates existing folder structure, metadata files,
  source code, and message files. Reports what is present, what is missing, and
  exactly how to fix gaps. Use when the user asks to "init", "setup", "check setup",
  "what do I need", or wants to onboard a new client for analysis.
argument-hint: "[ClientName] (optional -- checks all clients if omitted)"
allowed-tools: ["Read", "Glob", "Grep", "Bash"]
---

# LegacyLens: Setup Validator & Guide

You are checking whether the `as400/` folder has everything needed for
`/legacylens-analyze` to run successfully. Walk through each check below, report
findings clearly, and give actionable guidance for any gaps.

If the user provides a client name, scope all checks to that client. Otherwise
check every client found under `as400/`.

---

## Step 1: Discover Clients

Glob for `as400/*/` to find client folders. Ignore files (e.g., `.DS_Store`).

If no client folders exist, stop and print the **New Client Setup** guide
(see bottom of this file).

For each client found, run Steps 2-5.

---

## Step 2: Check Source Code

Glob for source files under `as400/{Client}/*/*/*.{CLP,CLLE,RPG,RPGLE,SQLRPG,SQLRPGLE,PF,LF,DSPF,PRTF,CMD,OCL,OCL36,RPG36,DFU36}`.

Report:
- Number of libraries found (distinct second-level folder names)
- Number of source files by type (extension)
- List library names

If zero source files are found, flag as **CRITICAL -- no source code**.

---

## Step 3: Check Cross-Reference Metadata

### 3.1 Program Cross-Reference (CRITICAL)

Glob for `as400/{Client}/metadata/xref_export_*.csv`.

If found:
- Read the first 3 lines to verify the header row contains:
  `Caller Object,Called Object,File Usage,File Type,Confidence,Origin,Location,Active`
- Count total rows (excluding header)
- Report filename and row count

If missing, flag as **CRITICAL** and show the expected format:

```
Caller Object,Called Object,File Usage,File Type,Confidence,Origin,Location,Active
"PROG1","PROG2","0","*PGM","CONFIRMED","","","Yes"
"PROG1","FILE1","1","*FILE","CONFIRMED","","","Yes"
```

Explain the columns:
| Column | Description | Example Values |
|--------|-------------|----------------|
| Caller Object | Program making the reference | `"BB955"` |
| Called Object | Target program, file, or data area | `"GSDTEDIT"`, `"BBPRCE"` |
| File Usage | `0`=Call, `1`=Input, `2`=Output, `3`=Update, `4`=Delete, `5`=Range, `6`=Chain, `7`=Open, `8`=Multi-I/O | `"0"`, `"1"` |
| File Type | `*PGM`, `*FILE`, `*SRVPGM`, `*DTAARA` | `"*PGM"` |
| Confidence | CONFIRMED, HIGH, MEDIUM, PENDING | `"CONFIRMED"` |
| Origin | Source of the reference (optional) | `"SBFDEV"`, `""` |
| Location | Library name (optional) | `""` |
| Active | Whether the reference is active | `"Yes"` |

### 3.2 PF-LF Mapping (RECOMMENDED)

Glob for `as400/{Client}/metadata/pflf_export_*.csv`.

If found:
- Read the first 3 lines to verify the header row contains:
  `Base Physical File,Logical File,Updated,Created`
- Count total rows
- Report filename and row count

If missing, flag as **WARNING** and show the expected format:

```
Base Physical File,Logical File,Updated,Created
"GGSTABL","GSTABL","2026-03-09T06:03:19.746Z","2026-02-26T16:32:55.402Z"
```

Explain: Without this file, `/legacylens-analyze` cannot resolve logical files
(views/indexes) back to their underlying physical files (tables). The report will
still work but file resolution will be incomplete.

---

## Step 4: Check Message Files (OPTIONAL)

Glob for `as400/{Client}/msgf/*.json`.

If found:
- List each JSON file
- Read the first 20 lines of each to verify the structure contains
  `messageFile` and `messages` array with `messageId` and `messageText` fields

If missing, flag as **INFO** -- message text lookup will be skipped. Show expected format:

```json
{
  "messageFile": "MYMSGF",
  "messages": [
    {
      "messageId": "ERR0001",
      "messageText": "Invalid value entered for &1"
    }
  ]
}
```

---

## Step 5: Cross-Check Source vs Metadata

For a quick sanity check:
1. Extract 10 sample `Caller Object` values from the xref CSV
2. For each, check if a matching source file exists under `as400/{Client}/`
3. Report the match rate

If match rate is very low, warn that source code and metadata may be from
different systems or libraries.

---

## Step 6: Summary Report

Print a summary table per client:

```
## LegacyLens Setup Status: {ClientName}

| Check | Status | Details |
|-------|--------|---------|
| Source code | OK / MISSING | {N} files across {M} libraries |
| Cross-reference (xref) | OK / MISSING | {N} rows in {filename} |
| PF-LF mapping (pflf) | OK / MISSING | {N} rows in {filename} |
| Message files (msgf) | OK / NONE | {N} files |
| Source-metadata alignment | OK / LOW | {X}% match on sample |
```

Then:
- If everything is OK: print "Ready to analyze. Run `/legacylens-analyze <ProgramName>`"
- If critical items are missing: print exactly what needs to be added and where
- If optional items are missing: note what will be limited in the report

---

## New Client Setup Guide

Print this when no clients exist or the user is onboarding a new client:

```
## Setting Up a New Client

1. Create the client folder:
   as400/{ClientName}/

2. Add AS400 source code organized by library and source type:
   as400/{ClientName}/{LibraryName}/QCLSRC/       -- CL programs (.CLP, .CLLE)
   as400/{ClientName}/{LibraryName}/QRPGSRC/      -- RPG programs (.RPG)
   as400/{ClientName}/{LibraryName}/QRPGLESRC/    -- ILE RPG programs (.RPGLE, .SQLRPGLE)
   as400/{ClientName}/{LibraryName}/QDDSSRC/      -- Data definitions (.PF, .LF, .DSPF, .PRTF)
   as400/{ClientName}/{LibraryName}/QSRC/         -- Mixed source (any type)
   as400/{ClientName}/{LibraryName}/QS36PRC/       -- S/36 OCL procedures (.OCL, .OCL36)

   The folder names are flexible -- the skills search by file extension,
   not folder name. The important thing is that source files have the
   correct extension.

3. Add cross-reference metadata (CRITICAL):
   as400/{ClientName}/metadata/xref_export_{timestamp}.csv

   This CSV maps every program-to-program call and program-to-file reference.
   Export it from your AS400 using DSPPGMREF or equivalent tooling.

4. Add PF-LF mapping metadata (RECOMMENDED):
   as400/{ClientName}/metadata/pflf_export_{timestamp}.csv

   This CSV maps logical files to their base physical files.
   Export it from your AS400 using DSPDBR or equivalent tooling.

5. Add message file exports (OPTIONAL):
   as400/{ClientName}/msgf/{MSGFNAME}_messages.json

   Export message file contents as JSON for message text resolution.

6. Run: /legacylens-init {ClientName}
   to verify everything is in place.

7. Run: /legacylens-analyze <ProgramName>
   to generate a modernization analysis report.
```
