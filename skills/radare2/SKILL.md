---
name: radare2
description: Reverse engineering with radare2 (r2) including disassembly, decompilation via r2ghidra and r2garlic, binary analysis, patching, debugging, and scripting. Use when analyzing binaries (ELF, PE, DEX, Mach-O, .sys drivers), decompiling functions, finding vulnerabilities, extracting strings, diffing binaries, or performing any reverse engineering task with r2. Triggers on radare2, r2, r2ghidra, r2garlic, pdg, disassemble, decompile binary, reverse engineer, binary analysis.
allowed-tools: Bash Read Glob Grep
---

# Radare2 Reverse Engineering

You are helping the user perform reverse engineering using radare2 (r2) and its ecosystem of tools and plugins.

## When to Use

- Analyzing unknown binaries (ELF, PE, Mach-O, DEX, firmware, Windows `.sys`/`.dll`/`.exe`)
- Decompiling native functions with r2ghidra (`pdg`) or DEX classes with r2garlic (`pd:G`)
- Extracting strings, imports, exports, symbols, or cross-references from a binary
- Searching for patterns, ROP gadgets, or specific byte sequences
- Patching binaries (NOPing checks, redirecting jumps)
- Triaging security mitigations (canary, NX, PIE, RELRO)
- Diffing two versions of a binary
- Scripting batch analysis with r2pipe or r2 command files

## When NOT to Use

- Interactive GUI-based RE workflows -- use standalone Ghidra at `/opt/ghidra_*/ghidraRun`
- Quick APK-to-Java decompilation without r2 context -- use the `jadx` skill instead
- Dynamic Android instrumentation at runtime -- use the `frida-waydroid` skill instead
- APK unpacking/resource extraction only -- use the `apktool` skill instead
- Static analysis for vulnerability patterns (SAST) -- use the `semgrep` or `codeql` skills instead

## Essential Principles

### 1. Always Use `-q` for Scripted Commands

r2 without `-q` enters interactive mode and hangs. Every automated invocation must use quiet mode.

```bash
r2 -qc 'aaa; afl' /path/to/binary       # correct
r2 -e scr.color=0 -qc 'afl' binary      # correct, no ANSI escapes
```

### 2. Analyze Before Decompiling

Without analysis, `pdg`/`pdf`/`afl`/`axt` produce garbage or nothing. Always run `aaa` (or at minimum `aa`) first. This populates the function list, cross-references, and type info that the decompiler depends on.

```bash
r2 -qc 'aaa; s main; pdg' binary    # correct
r2 -qc 'pdg @ main' binary          # wrong -- no analysis done
```

### 3. Prefer JSON for Structured Data

Appending `j` to most commands returns JSON. This avoids parsing column-aligned text and reduces hallucination when processing output programmatically.

```bash
r2 -qc 'aaa; aflj' binary           # function list as JSON
r2 -qc 'iIj' binary                 # binary info as JSON
```

### 4. Disable Colors When Capturing Output

r2 emits ANSI escape codes by default. Always disable when piping or capturing:

```bash
r2 -e scr.color=0 -qc 'aaa; pdg @ main' binary
```

### 5. All PE Formats Are Equal

Windows `.sys` kernel drivers, `.dll` libraries, and `.exe` programs are all PE format. r2ghidra decompiles them identically -- there is no special handling needed for `.sys` files.

### 6. Seek Sets the Decompilation Target

The decompiler operates on the function at the current seek. Use `s` to seek or `@` for temporary seek:

```bash
r2 -qc 'aaa; pdg @ main' binary             # temp-seek
r2 -qc 'aaa; s sym.target; pdg' binary       # persistent seek
```

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
| `pdc` | Built-in pseudo-C decompilation |
| `pdg` | Ghidra decompilation (r2ghidra) |
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

### Quick Triage

```bash
rabin2 -I binary                             # fast info, no r2 session
r2 -qc 'iI~canary,nx,pic,relro' binary      # mitigations
r2 -qc 'ii' binary                           # imports
r2 -qc 'izz' binary                          # all strings
```

### Full Analysis + Decompilation

```bash
r2 -qc 'aaa; afl' binary                    # list functions
r2 -qc 'aaa; s main; pdg' binary            # decompile main
r2 -qc 'aaa; pdg @ sym.target' binary       # decompile by name
r2 -qc 'aaa; s 0x401000; af; pdg' binary    # analyze + decompile at addr
```

### Cross-References

```bash
r2 -qc 'aaa; axt @ sym.dangerous_func' binary   # who calls this?
r2 -qc 'aaa; axf @ main' binary                 # what does main call?
r2 -qc 'aaa; axt @@ sym.imp.*' binary           # xrefs to all imports
```

### Windows PE / Driver

```bash
r2 -qc 'aaa; iI; ii; iE; afl' driver.sys
r2 -qc 'aaa; s entry0; pdg' driver.sys
r2 -qc 'aaa; ii~Nt\|Zw\|Ke' driver.sys          # kernel API imports
```

### Android DEX

```bash
r2 -qc 'ic~com/example' classes.dex              # list app classes
r2 -qc 's 0x1d0934; pd:Gc' classes.dex           # decompile class
r2 -qc 'pd:Ga' classes.dex                       # decompile all
r2 -qc 'izz~password\|api_key\|secret' classes.dex
```

### Patching

```bash
r2 -wqc 'wx 9090 @ 0x401234' binary             # NOP 2 bytes
r2 -wqc 'wa jmp 0x401300 @ 0x401234' binary      # redirect jump
```

### ROP Gadgets

```bash
r2 -qc '/R pop rdi;ret' binary
r2 -qc '/R/ .*pop.*ret' binary                   # regex search
```

## Companion Tools

| Tool | Usage |
|------|-------|
| `rabin2 -I file` | Binary info without r2 session |
| `rabin2 -z file` / `rabin2 -i file` | Strings / imports |
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
r2 -qc 'aaa; afl; s main; pdg' binary

# Script file
r2 -qi script.r2 binary

# r2pipe (Python)
import r2pipe
r2 = r2pipe.open("/path/to/binary")
r2.cmd("aaa")
functions = r2.cmdj("aflj")
r2.quit()
```

## Success Criteria

- [ ] Binary format and architecture correctly identified (`iI`)
- [ ] Analysis completed without errors (`aaa` or appropriate level)
- [ ] Target functions found and decompiled cleanly
- [ ] Cross-references traced for functions of interest
- [ ] Strings and imports reviewed for attack surface / behavior
- [ ] Security mitigations documented
- [ ] Findings communicated with specific addresses and function names
