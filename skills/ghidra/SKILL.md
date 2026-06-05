---
name: ghidra
description: Reverse engineering with Ghidra GUI, headless analyzer, PyGhidra, decompiler, p-code, repo-local shared Ghidra projects, and persistent project annotations. Use when analyzing binaries, firmware, native libraries, unusual architectures, coordinating subagents on one reverse-engineering target, or when you need decompilation, recovered types, symbols, comments, labels, bookmarks, or scriptable Ghidra project analysis.
allowed-tools: Bash Read Glob Grep
---

# Ghidra Reverse Engineering

Use Ghidra for deep static analysis when you need persistent analysis state, mature decompilation, broad loader/processor support, p-code, or Python-driven automation.

## When to Use

- Native binary decompilation for ELF, PE, Mach-O, firmware executables, drivers, and shared libraries
- Persistent annotation work: symbols, labels, comments, bookmarks, function signatures, types, structs, enums, and namespaces
- Ghidra project workflows where recovered context must survive across sessions
- Headless batch analysis with `analyzeHeadless`
- PyGhidra or Ghidra Python scripts for programmatic analysis
- P-code or SLEIGH-level reasoning for instruction semantics and architecture behavior
- Firmware, embedded targets, unusual processors, or loaders where Ghidra has stronger support than CLI tools

## When NOT to Use

- Quick strings/imports/mitigation triage where `rabin2` or `r2` is faster
- APK Java/Kotlin source review; use `jadx`
- APK resources, manifest, smali, or repackaging; use `apktool`
- Runtime behavior, decrypted values, or anti-tamper checks; use Frida/debugger tooling
- One-off byte patch probes where radare2 write commands are enough

## Local Setup

- GUI: `/opt/ghidra_12.0.4_PUBLIC/ghidraRun`
- Headless: `/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless`
- Scripts directory: `/opt/ghidra_scripts`

Prefer absolute paths because `ghidra` and `analyzeHeadless` may not be on `PATH`.

## Essential Principles

### 0. Use One Repo-Local Shared Workspace

For any non-trivial RE task, create or reuse the shared workspace under the repository root/current working directory, not `/tmp` and not a per-agent home directory:

```text
.re/
  ghidra/        # Ghidra .gpr/.rep projects
  radare2/       # r2 projects when radare2 is used
  RE-NOTES.md    # cross-tool index of targets, findings, owners, and open questions
```

Use one Ghidra project per target or case: `.re/ghidra/<case>.gpr` and `.re/ghidra/<case>.rep`. Before importing or analyzing, check whether `.re/ghidra` and `.re/RE-NOTES.md` already exist and reuse them.

Do not create duplicate Ghidra projects for the same binary unless the user asks for an experimental branch. Duplicate projects cause subagents to redo imports, analysis, renames, and comments.

### 1. Preserve Analysis in Projects

Treat Ghidra projects as the durable analysis database. They preserve imported programs, analysis results, symbols, labels, comments, bookmarks, types, structs, enums, function signatures, namespaces, and other recovered context.

Use the repo-local shared project whenever the task involves more than quick triage or when findings should be revisited later.

Ghidra's local project is not the same as a Ghidra Server shared repository. Local projects are the right default for a single repo workspace, but they should be treated as single-writer. If a Ghidra GUI/headless session is already working on the project, other subagents should read `.re/RE-NOTES.md`, exported decompiler snippets, or tool output rather than opening another write session.

### 2. Annotate While You Learn

Rename functions and variables, apply function signatures, create structs, add comments, and bookmark suspicious locations. The value of Ghidra is not just pseudocode; it is the accumulated analysis state.

Prefer comments that capture evidence and uncertainty:

- Good: `Length is attacker-controlled; bounds checked in validate_len before allocation.`
- Good: `Likely ioctl dispatcher; command values from switch at 0x...`
- Bad: `interesting` or `check this`

Name functions by observed role and evidence boundary: `rc4_set_input`, `http_post_wrapper`, `command_dispatcher`, `admin_elevation_path`, `config_resource_decrypt`. When the name is uncertain, prefer `suspected_` or document the uncertainty in the comment rather than encoding a guess as fact.

### 3. Use the Decompiler, but Verify Against Disassembly

Ghidra pseudocode is evidence, not ground truth. Cross-check suspicious behavior with disassembly, xrefs, data references, imports, and p-code when semantics matter.

### 4. Use Headless for Repeatability

Use `analyzeHeadless` for reproducible imports, batch processing, and scripted extraction. Keep scripts in a project or temp working directory and report the command used.

```bash
/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless "$PWD/.re/ghidra" case-name \
  -import /path/to/binary \
  -analysisTimeoutPerFile 300 \
  -max-cpu 1
```

Never use `-deleteProject` for shared work. Use `-max-cpu 1` or another explicit low value when multiple agents may be active so one import does not starve the machine. If the project may already be open in Ghidra, expect headless processing to fail; do not work around that by creating a second project.

### 5. Use Python APIs for Structured Analysis

Use Ghidra scripts or PyGhidra when you need programmatic functions, xrefs, symbols, decompiler output, p-code, types, or annotations.

Typical APIs and concepts:

- `currentProgram`, `getFunctionAt`, `getFunctions`, `getReferencesTo`, `getReferencesFrom`
- `FlatProgramAPI` helpers for symbols, comments, labels, memory, and functions
- `DecompInterface` for pseudocode and high-level decompiler information
- `HighFunction` and p-code ops for lower-level semantic inspection
- Data type managers for structs, enums, typedefs, and function signatures

Prefer read-only offline scripts for malware string/resource/config extraction. Scripts should operate on bytes or imported Ghidra program state, report source addresses, encodings, keys, and confidence, and avoid executing samples or making network connections.

## Common Workflows

### Shared Project Setup

```bash
mkdir -p .re/ghidra
/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless "$PWD/.re/ghidra" case-name \
  -import /absolute/path/to/binary \
  -analysisTimeoutPerFile 300 \
  -max-cpu 1
```

Record the project name, binary path/hash, architecture, analysis status, and assigned function ranges in `.re/RE-NOTES.md`. Keep durable discoveries in the Ghidra project itself as comments, symbols, types, and bookmarks; use the notes file as an index so subagents know where to look.

Use this minimal notes structure when creating `.re/RE-NOTES.md`:

```markdown
# Reverse Engineering Notes

## Targets
- `<binary>`: sha256 `<hash>`, Ghidra project `<case>`, r2 project `<case>`, status `<triage|analyzed|blocked>`

## Work Queue
- `<addr-or-function-range>`: owner `<agent/session>`, status `<todo|in-progress|done>`, notes `<short pointer>`

## Findings
- `<addr/function>`: `<tool>`, `<finding/evidence>`, `<open question or next step>`
```

Do not commit `.re/` artifacts unless the user explicitly asks; these project databases can be large and may include copied binaries or proprietary samples.

### Exported Evidence Artifacts

Before rerunning headless import or auto-analysis on a large binary, check for existing exported artifacts such as `.re/RE-NOTES.md`, `report/artifacts/**/README.md`, decompiler exports, decoded resources, helper scripts, pcaps, or prior function maps. Prefer targeted one-off queries against the shared project over repeating full analysis.

When producing exports for later agents, write a small index that includes target hash, original path, Ghidra project/program name, export purpose, key function names/addresses, and open questions. Exported snippets are not a substitute for project annotations; they are a cache so other agents can avoid recomputing expensive decompilation.

### Static-Dynamic Feedback

Use Ghidra to identify crypto, compression, parser, and network boundaries, then hand exact function names, addresses, argument expectations, and calling conventions to the dynamic/debugging phase. Runtime hooks at encryption/decryption input/output boundaries often recover plaintext faster than trying to reconstruct every caller statically.

If a dynamic breakpoint or hook fails because an object layout or argument offset was wrong, return to Ghidra to recover the struct layout/calling convention before continuing. Feed confirmed runtime observations back into the project as comments, labels, signatures, and `.re/RE-NOTES.md` entries.

### GUI Deep Dive

```bash
/opt/ghidra_12.0.4_PUBLIC/ghidraRun
```

Open the existing repo-local project from `.re/ghidra` if one exists. If no project exists, create it there, import the binary, run auto-analysis, then work from entry points, imports, strings, xrefs, and decompiler output. Save the project before switching tasks.

### Headless Import and Analyze

```bash
/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless "$PWD/.re/ghidra" sample-project \
  -import /path/to/binary \
  -analysisTimeoutPerFile 300 \
  -max-cpu 1
```

Use this for initial analysis that you can reopen in the GUI.

### Headless Script Run

```bash
/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless "$PWD/.re/ghidra" sample-project \
  -process binary-name \
  -scriptPath /opt/ghidra_scripts \
  -postScript script.py \
  -max-cpu 1
```

Use this when extracting function lists, xrefs, comments, or decompiler output for a report.

### Multi-Agent Coordination

- One agent should own Ghidra project writes at a time: import, auto-analysis, renames, type changes, comments, bookmarks, and saves.
- Other agents may consume exported snippets, `.re/RE-NOTES.md`, or read-only command output while the writer is active.
- Divide work by function ranges or named subsystems in `.re/RE-NOTES.md` before spawning subagents.
- Each subagent should add final durable findings back to the Ghidra project as comments/bookmarks or return exact commands/comments for the writer to apply.
- If true concurrent Ghidra editing is required, use a Ghidra Server shared project with checkout/checkin instead of a local `.gpr` project.

## Ghidra vs Radare2

Use Ghidra first for:

- Persistent project analysis and annotation-heavy work
- Decompiler-centric workflows
- Recovering and applying types, structs, enums, and function signatures
- P-code/SLEIGH reasoning and unusual architectures

Use radare2 first for:

- Fast metadata, imports, strings, sections, mitigations, and hashes
- Quick byte search, gadget search, and small patches
- One-shot CLI commands and lightweight scripted probes
- r2 projects when you need minimal persistent CLI annotations

## Reporting Discipline

- State whether evidence came from Ghidra GUI, `analyzeHeadless`, a Ghidra script, or radare2 cross-checking.
- Include function names and addresses when discussing behavior.
- Distinguish decompiler output from verified assembly/p-code observations.
- Mention unresolved type recovery or analysis failures when they affect confidence.
- Save durable annotations in the repo-local Ghidra project when continuing work later.
- Update `.re/RE-NOTES.md` with project name, target hash, analyzed functions, open questions, and which tool produced each major conclusion.

## Success Criteria

- [ ] File format, architecture, load address, and entry points identified
- [ ] Auto-analysis completed or limitations documented
- [ ] Repo-local project under `.re/ghidra` reused or created for non-trivial work
- [ ] `.re/RE-NOTES.md` checked before starting and updated before handoff
- [ ] Relevant functions renamed or bookmarked
- [ ] Key comments, types, structs, and signatures applied where useful
- [ ] Pseudocode claims checked against xrefs/disassembly for important findings
- [ ] Findings include addresses, function names, and evidence source
