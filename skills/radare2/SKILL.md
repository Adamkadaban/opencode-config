---
name: radare2
description: Reverse engineering with radare2 (r2) including repo-local shared r2 projects, disassembly, decompilation via r2ghidra and r2garlic, binary analysis, patching, debugging, annotations, and scripting. Use when analyzing binaries (ELF, PE, DEX, Mach-O, .sys drivers), coordinating subagents on one r2 project, decompiling functions, finding vulnerabilities, extracting strings, diffing binaries, or performing any reverse engineering task with r2. Triggers on radare2, r2, r2ghidra, r2garlic, pdg, disassemble, decompile binary, reverse engineer, binary analysis.
allowed-tools: Bash Read Glob Grep
---

# Radare2 Reverse Engineering

You are helping the user perform reverse engineering using radare2 (r2) and its ecosystem of tools and plugins.

## Local Setup

- Preferred r2: `/opt/radare2/binr/radare2/radare2`
- Preferred rabin2: `/opt/radare2/binr/rabin2/rabin2`
- Fallback r2/rabin2: `/usr/local/bin/r2`, `/usr/local/bin/rabin2`

Prefer the `/opt/radare2` binaries when available so agents use the same local build.

## When to Use

- Analyzing unknown binaries (ELF, PE, Mach-O, DEX, firmware, Windows `.sys`/`.dll`/`.exe`)
- Decompiling native functions with r2ghidra (`pdg`) or DEX classes with r2garlic (`pd:G`)
- Extracting strings, imports, exports, symbols, or cross-references from a binary
- Searching for patterns, ROP gadgets, or specific byte sequences
- Patching binaries (NOPing checks, redirecting jumps)
- Triaging security mitigations (canary, NX, PIE, RELRO)
- Diffing two versions of a binary
- Scripting batch analysis with r2pipe or r2 command files
- Maintaining lightweight persistent analysis state with r2 projects

## When NOT to Use

- Interactive GUI-based RE workflows -- use standalone Ghidra at `/opt/ghidra_*/ghidraRun`
- Quick APK-to-Java decompilation without r2 context -- use the `jadx` skill instead
- Dynamic Android instrumentation at runtime -- use the `frida-waydroid` skill instead
- APK unpacking/resource extraction only -- use the `apktool` skill instead
- Static analysis for vulnerability patterns (SAST) -- use the `semgrep` or `codeql` skills instead

## Essential Principles

### 0. Use One Repo-Local Shared Workspace

For any non-trivial RE task, create or reuse this workspace under the repository root/current working directory:

```text
.re/
  radare2/projects/   # r2 dir.projects override
  radare2/cache/      # r2 dir.cache override for cache-backed annotations when supported
  ghidra/             # Ghidra projects when Ghidra is used
  RE-NOTES.md         # cross-tool index of targets, findings, owners, and open questions
```

Every r2 command that creates or uses a project must set `dir.projects` to `$PWD/.re/radare2/projects`. Do not use the default `~/.local/share/radare2/projects`, because subagents will not reliably discover each other's work there.

Use one r2 project name per target or case. Before running `aaa`, `pdg`, or saving a project, check whether `.re/radare2/projects` and `.re/RE-NOTES.md` already describe existing work.

### 1. Always Use `-q` for Scripted Commands

r2 without `-q` enters interactive mode and hangs. Every automated invocation must use quiet mode.

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; afl' /path/to/binary       # correct
/opt/radare2/binr/radare2/radare2 -e scr.color=0 -qc 'afl' binary      # correct, no ANSI escapes
```

### 2. Analyze Before Decompiling

Without analysis, `pdg`/`pdf`/`afl`/`axt` produce garbage or nothing. Always run `aaa` (or at minimum `aa`) first. This populates the function list, cross-references, and type info that the decompiler depends on.

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; s main; pdg' binary    # correct
/opt/radare2/binr/radare2/radare2 -qc 'pdg @ main' binary          # wrong -- no analysis done
```

### 3. Prefer JSON for Structured Data

Appending `j` to most commands returns JSON. This avoids parsing column-aligned text and reduces hallucination when processing output programmatically.

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; aflj' binary           # function list as JSON
/opt/radare2/binr/radare2/radare2 -qc 'iIj' binary                 # binary info as JSON
```

### 4. Disable Colors When Capturing Output

r2 emits ANSI escape codes by default. Always disable when piping or capturing:

```bash
/opt/radare2/binr/radare2/radare2 -e scr.color=0 -qc 'aaa; pdg @ main' binary
```

### 5. All PE Formats Are Equal

Windows `.sys` kernel drivers, `.dll` libraries, and `.exe` programs are all PE format. r2ghidra decompiles them identically -- there is no special handling needed for `.sys` files.

### 6. Seek Sets the Decompilation Target

The decompiler operates on the function at the current seek. Use `s` to seek or `@` for temporary seek:

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; pdg @ main' binary             # temp-seek
/opt/radare2/binr/radare2/radare2 -qc 'aaa; s sym.target; pdg' binary       # persistent seek
```

### 7. Use Projects for Iterative Work

r2 projects persist analysis state across sessions, including flags, names, comments, types, analysis metadata, and patches. Use them when the investigation will last longer than one command, when you are annotating functions, or when you need to preserve recovered context.

```bash
mkdir -p .re/radare2/projects .re/radare2/cache
/opt/radare2/binr/radare2/radare2 -q \
  -e dir.projects="$PWD/.re/radare2/projects" \
  -e dir.cache="$PWD/.re/radare2/cache" \
  binary                                   # open target
aaa; afn validate_key @ 0x401230; CCu checks license key @ 0x401230
Ps case-name                                # save project
/opt/radare2/binr/radare2/radare2 -q \
  -e dir.projects="$PWD/.re/radare2/projects" \
  -p case-name                              # reopen saved project
```

Use `Po` to open a project from inside r2, `Ps name` to save, and `P- name` only when deliberately deleting a project.

### 8. Serialize Heavy r2 Work

Do not spawn multiple subagents that all run `aaa`, `pdg`, `pd:Ga`, or `Ps` against the same target. r2 projects are saved as scripts/SDB data, and concurrent write sessions can overwrite or fork analysis state. Use one writer for project-changing work and let other agents consume saved project output, JSON exports, or `.re/RE-NOTES.md`.

If parallel subagents are needed, partition by target binary or by read-only tasks. For the same target, assign function ranges in `.re/RE-NOTES.md` and have subagents return proposed names/comments/decompiler observations for the writer to apply.

### 9. Store Notes Where Other Agents Will See Them

Use project comments (`CCu`, `CCa`, `CCf`), function names (`afn`), types (`td`), and saved projects (`Ps`) for durable r2 state. Use `Pn` project notes for r2-specific notes when working interactively, but also keep `.re/RE-NOTES.md` as the cross-tool index because Ghidra users and non-r2 subagents can read it.

The `ano` function annotation feature is cache-backed and useful for caching function notes or decompiler output. If using it, start r2 with `-e dir.cache="$PWD/.re/radare2/cache"`; still treat project comments and `.re/RE-NOTES.md` as the primary shared record.

### 10. Prefer Targeted Reuse Over Reanalysis

Before running `aaa`, `pdg`, `pd:Ga`, or bulk xref/decompiler sweeps on a large target, check `.re/RE-NOTES.md`, `.re/radare2/projects`, and any `report/artifacts/**/README.md` for prior exports. If useful artifacts exist, run targeted read-only queries against specific functions/addresses instead of redoing full analysis.

For malware or unknown binaries, use offline byte-level extraction scripts for strings, resources, configs, and encodings. Record file offsets, virtual addresses, keys, and decode confidence; do not execute samples or contact network infrastructure from r2 scripts.

### 11. Triage Overlays and Packed Blobs Carefully

For high-entropy overlays or large appended data, record section end, overlay offset/size/hash, entropy, structural signature hits, known-key decrypt attempts, and xrefs or file-I/O references to that range. Treat raw `MZ` byte-pair hits in random data as noise unless a structurally valid PE header, coherent sections, and plausible references are present.

Useful commands include `iS`, `iIj`, `it`, `/x 4d5a`, `/x 504b0304`, `/x 4d534346`, xrefs to file APIs, and targeted `px`/`p8` reads around candidate offsets. Prefer validating candidate embedded files structurally before spending decompiler time on them.

## Command Quick Reference

### Info (`i`)

| Command | Description |
|---------|-------------|
| `iI` | Binary info (arch, bits, canary, nx, pic, stripped, relro) |
| `ii` | Imports |
| `iE` | Exports |
| `is` | Symbols |
| `iS` | Sections |
| `iz` / `izz` | Strings in data sections / all strings |
| `ie` | Entry points |
| `il` | Linked libraries |
| `ic` | Classes (Java/ObjC/C++) |
| `iR` | Resources |
| `ih` | Headers |
| `it` | File hashes |

### Analysis (`a`)

| Command | Description |
|---------|-------------|
| `aa` | Analyze functions at symbols |
| `aaa` | Full analysis (functions + refs + names + types) |
| `af` / `af @ addr` | Analyze function at current or specific offset |
| `afl` / `afll` | List functions / verbose with sizes |
| `aflc` | Count functions |
| `afn name @ addr` | Rename function |
| `axt addr` / `axf addr` | Xrefs TO / FROM addr |
| `aap` / `aac` | Find preludes / analyze calls (stripped binaries) |

### Print/Disassemble (`p`)

| Command | Description |
|---------|-------------|
| `pd N` / `pd -N` | Disassemble N instructions forward / backward |
| `pdf` | Disassemble current function |
| `pdc` | Built-in pseudo-C decompilation, available without r2ghidra |
| `pdg` | Ghidra decompilation through r2ghidra |
| `pdg*` | Decompiled code as r2 comment commands |
| `pdga` | Side-by-side disasm + decompile |
| `pdgj` | Ghidra decompilation as JSON |
| `pdgo` | Decompile with address offsets |
| `pdgx` / `pdgd` | XML output / debug XML |
| `pdgp` | Switch to Sleigh disassembly/analysis |
| `pdgs` / `pdgss` | List Sleigh languages / show auto-detected |
| `pdgsd N` | Sleigh disassemble N instructions with pcode |
| `pd:G` | Garlic decompile method at seek (DEX only) |
| `pd:Gc` | Garlic decompile full class (DEX only) |
| `pd:Ga` | Garlic decompile all classes (DEX only) |
| `pd:Gi` | DEX info dump (DEX only) |
| `pd:Gs` | Smali output for class at seek (DEX only) |
| `px N` / `pxr N` | Hexdump / hexdump with references |

### Seek (`s`), Search (`/`), Write (`w`), Debug (`d`)

| Command | Description |
|---------|-------------|
| `s addr` / `s sym.name` | Seek to address or symbol |
| `s-` / `s+` | Undo / redo seek |
| `sf` / `sf.` | Next function / beginning of current |
| `/ string` / `/x hex` | Search string / hex bytes |
| `/a asm` / `/R gadget` | Search instruction / ROP gadgets |
| `wx hex` / `wa asm` | Write hex bytes / assembled instruction |
| `dc` / `ds` / `dso` | Continue / step / step over |
| `db addr` / `dr` / `dm` | Breakpoint / registers / memory maps |

### Output Modifiers

| Suffix | Meaning | Example |
|--------|---------|---------|
| `j` | JSON output | `aflj`, `iij`, `pdfj` |
| `*` | r2 commands | `afl*` |
| `q` | Quiet/minimal | `aflq` |
| `~pattern` | Grep output | `afl~main` |
| `~[N]` / `~:N` | Column / row select | `afl~[0]`, `afl~:0` |
| `@` / `@@` | Temporary seek / iterate | `pdg @ main`, `pdf @@ sym.imp.*` |

## r2ghidra Config (from source)

| Variable | Default | Description |
|----------|---------|-------------|
| `r2ghidra.sleighhome` | `""` | Path to Sleigh specs dir (also `$SLEIGHHOME` env) |
| `r2ghidra.lang` | `""` | Override Sleigh language (e.g. `x86:LE:64:default`) |
| `r2ghidra.vars` | `true` | Honor r2's variable/argument analysis |
| `r2ghidra.cmt.cpp` | `true` | Use `//` vs `/* */` comments |
| `r2ghidra.indent` | `4` | Code indent increment |
| `r2ghidra.linelen` | `120` | Max line length |
| `r2ghidra.rawptr` | `true` | Show unknown globals as raw addresses |
| `r2ghidra.verbose` | `false` | Show warnings |
| `r2ghidra.casts` | `false` | Show type casts |
| `r2ghidra.fixups` | `false` | Apply shared-return jump fixups |
| `r2ghidra.roprop` | `0` | Read-only constant propagation (0-4) |
| `r2ghidra.timeout` | `0` | Kill after N ms (UNIX, forks child). 0=off |

Thread-safe via recursive mutex. Timeout uses `fork()`+`select()`+`kill(9)`.

## r2garlic Config & Behavior (from source)

| Variable | Default | Description |
|----------|---------|-------------|
| `r2garlic.show_imports` | `true` | Prepend package/import header in method mode |
| `r2garlic.threads` | `1` | Registered but unused (always single-threaded) |

**Class resolution**: runs `ic.` to get class at seek, falls back to `find_class_at_seek()` (largest `class_data_off` <= seek, skipping inner/anonymous), then first non-inner class. **Method resolution**: parses `ic.` for `.method.` marker; if not found, decompiles entire class. Handles up to 16 overloads. `pd:Ga` skips inner/anonymous classes (decompiled inline). File bytes are cached but DEX is re-parsed per command.

## Common Workflows

### Shared Project Setup

```bash
mkdir -p .re/radare2/projects .re/radare2/cache
/opt/radare2/binr/radare2/radare2 -q \
  -e scr.color=0 \
  -e dir.projects="$PWD/.re/radare2/projects" \
  -e dir.cache="$PWD/.re/radare2/cache" \
  -e prj.files=false \
  -e prj.vc=true \
  -qc 'aaa; Ps case-name' \
  /absolute/path/to/binary
```

Record the project name, binary path/hash, architecture, analysis status, and assigned function ranges in `.re/RE-NOTES.md`. Reopen with the same `dir.projects` override; otherwise r2 will look in the home-directory project store.

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

When producing reusable outputs, add a short artifact index entry with target hash, r2 project name, command used, exported path, relevant function/address range, and whether the output is raw observation or inferred behavior.

### Quick Triage

```bash
/opt/radare2/binr/rabin2/rabin2 -I binary                             # fast info, no r2 session
/opt/radare2/binr/radare2/radare2 -qc 'iI~canary,nx,pic,relro' binary # mitigations
/opt/radare2/binr/radare2/radare2 -qc 'ii' binary                     # imports
/opt/radare2/binr/radare2/radare2 -qc 'izz' binary                    # all strings
```

### Full Analysis + Decompilation

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; afl' binary                    # list functions
/opt/radare2/binr/radare2/radare2 -qc 'aaa; s main; pdg' binary            # decompile main
/opt/radare2/binr/radare2/radare2 -qc 'aaa; pdg @ sym.target' binary       # decompile by name
/opt/radare2/binr/radare2/radare2 -qc 'aaa; s 0x401000; af; pdg' binary    # analyze + decompile at addr
```

Use `pdc` for quick built-in pseudocode when plugin availability is unknown. Use `pdg` when r2ghidra is installed and you want Ghidra-backed decompiler output from inside r2.

### Persistent Annotation Session

```bash
/opt/radare2/binr/radare2/radare2 -q -e dir.projects="$PWD/.re/radare2/projects" binary
aaa
afn parse_header @ 0x401560
CCu validates magic and length before allocation @ 0x401560
"td struct packet { uint32_t magic; uint16_t len; }"
Ps packet-review
```

Prefer this over one-shot commands once you start creating names, comments, types, or patches that should survive the session.

### Cross-References

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; axt @ sym.dangerous_func' binary   # who calls this?
/opt/radare2/binr/radare2/radare2 -qc 'aaa; axf @ main' binary                 # what does main call?
/opt/radare2/binr/radare2/radare2 -qc 'aaa; axt @@ sym.imp.*' binary           # xrefs to all imports
```

### Windows PE / Driver

```bash
/opt/radare2/binr/radare2/radare2 -qc 'aaa; iI; ii; iE; afl' driver.sys
/opt/radare2/binr/radare2/radare2 -qc 'aaa; s entry0; pdg' driver.sys
/opt/radare2/binr/radare2/radare2 -qc 'aaa; ii~Nt\|Zw\|Ke' driver.sys          # kernel API imports
```

### Android DEX

```bash
/opt/radare2/binr/radare2/radare2 -qc 'ic~com/example' classes.dex              # list app classes
/opt/radare2/binr/radare2/radare2 -qc 's 0x1d0934; pd:Gc' classes.dex           # decompile class
/opt/radare2/binr/radare2/radare2 -qc 'pd:Ga' classes.dex                       # decompile all
/opt/radare2/binr/radare2/radare2 -qc 'izz~password\|api_key\|secret' classes.dex
```

### Patching

```bash
/opt/radare2/binr/radare2/radare2 -wqc 'wx 9090 @ 0x401234' binary             # NOP 2 bytes
/opt/radare2/binr/radare2/radare2 -wqc 'wa jmp 0x401300 @ 0x401234' binary      # redirect jump
```

### ROP Gadgets

```bash
/opt/radare2/binr/radare2/radare2 -qc '/R pop rdi;ret' binary
/opt/radare2/binr/radare2/radare2 -qc '/R/ .*pop.*ret' binary                   # regex search
```

## Companion Tools

| Tool | Usage |
|------|-------|
| `/opt/radare2/binr/rabin2/rabin2 -I file` | Binary info without r2 session |
| `/opt/radare2/binr/rabin2/rabin2 -z file` / `/opt/radare2/binr/rabin2/rabin2 -i file` | Strings / imports |
| `rasm2 -a x86 -b 64 'nop'` | Assemble instruction |
| `rasm2 -a x86 -b 64 -d '90'` | Disassemble hex |
| `rahash2 -a sha256 file` | Hash file |
| `radiff2 -s a b` / `radiff2 -c a b` | Symbol / code diff |
| `rafind2 -s 'password' file` | String search in raw file |
| `ragg2 -P 200 -r` | De Bruijn pattern |
| `r2pm -ci plugin` / `r2pm -l` | Install plugin / list installed |

## Scripting

```bash
# One-shot
/opt/radare2/binr/radare2/radare2 -qc 'aaa; afl; s main; pdg' binary

# Script file
/opt/radare2/binr/radare2/radare2 -qi script.r2 binary

# r2pipe (Python)
import r2pipe
r2 = r2pipe.open("/path/to/binary")
r2.cmd("aaa")
functions = r2.cmdj("aflj")
r2.quit()
```

Use r2pipe for repeatable extraction and annotation, not just reads. Prefer JSON commands such as `aflj`, `agfj`, `pdfj`, `pdgj`, `axfj`, and `axtj` when building scripts.

## Success Criteria

- [ ] Binary format and architecture correctly identified (`iI`)
- [ ] Repo-local project directory `.re/radare2/projects` reused or created for non-trivial work
- [ ] `.re/RE-NOTES.md` checked before starting and updated before handoff
- [ ] Analysis completed without errors (`aaa` or appropriate level)
- [ ] Target functions found and decompiled cleanly
- [ ] Cross-references traced for functions of interest
- [ ] Strings and imports reviewed for attack surface / behavior
- [ ] Security mitigations documented
- [ ] Findings communicated with specific addresses and function names
