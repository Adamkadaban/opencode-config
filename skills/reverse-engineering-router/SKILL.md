---
name: reverse-engineering-router
description: Decision guide for choosing Ghidra, radare2, JADX, apktool, Frida, and related reverse-engineering tools, including repo-local shared Ghidra/radare2 project coordination for subagents. Use when a request mentions reverse engineering, decompilation, binary analysis, tool choice, persistent annotations, multi-agent RE work, or asks whether to use radare2 vs Ghidra.
allowed-tools: Bash Read Glob Grep
---

# Reverse Engineering Tool Router

Use this skill to choose the right reverse-engineering backend before deep work. Prefer the most precise tool that is available and appropriate, but do not spend time building elaborate infrastructure for a quick triage task.

## First Question

Classify the task:

- Quick triage: identify file type, strings, imports, symbols, mitigations, packer clues.
- Static decompilation: explain functions, recover control flow, types, structures, xrefs.
- IL/data-flow: reason about variable definitions, possible values, SSA, instruction semantics.
- Patching: modify branches, bytes, constants, or save a modified binary.
- Android app analysis: resources, manifest, Java/Kotlin, native libraries.
- Dynamic analysis: runtime hooks, syscalls, debugger traces, device interaction.
- Firmware/container extraction: embedded filesystems, blobs, firmware layouts.

## Shared Workspace Default

For any task beyond one-shot triage, all agents must use the repository root/current working directory as the collaboration point:

```text
.re/
  ghidra/              # Ghidra .gpr/.rep projects
  radare2/projects/    # r2 projects via dir.projects override
  radare2/cache/       # r2 cache override for cache-backed annotations when supported
  RE-NOTES.md          # cross-tool project index, target hashes, owners, findings, open questions
```

Before starting Ghidra or radare2 analysis, check for `.re/RE-NOTES.md`, `.re/ghidra`, and `.re/radare2/projects`. Reuse existing projects and notes. Do not create duplicate per-agent projects for the same binary.

Use a single-writer model for local Ghidra and radare2 projects. One agent performs imports, heavy analysis, decompilation sweeps, comments, renames, type changes, project saves, and r2 `Ps`; other subagents should do read-only triage, consume exported snippets, or return proposed annotations for the writer to apply. If true concurrent Ghidra editing is required, use a Ghidra Server shared project instead of a local `.gpr` project.

Do not commit `.re/` artifacts unless the user explicitly asks. Use `.re/RE-NOTES.md` to track targets, sha256 hashes, project names, function-range owners, findings, and open questions.

Also check existing `report/artifacts/**/README.md`, decompiler exports, decoded-resource directories, helper scripts, pcaps, and previous protocol notes before launching new heavy analysis. Existing exports should be treated as a cache and index, not as proof; verify important claims against the shared project, disassembly, or runtime evidence.

## Default Routing

Use Ghidra when:

- You need deep static analysis, decompilation, cross-references, type recovery, or long-running annotation work.
- You need broad architecture/loader support, SLEIGH, p-code, or mature batch headless workflows.
- You are working on firmware, embedded formats, unusual processors, or cases where Ghidra import works better.
- You need project-style analysis artifacts: symbols, labels, comments, types, structs, bookmarks, signatures, and recovered analysis state.
- You need Python-driven analysis with PyGhidra or Ghidra headless scripts.
- You want persistent GUI or headless decompiler evidence without a commercial license.

Use radare2 when:

- You need fast CLI triage, small scripted probes, or lightweight persistent state through r2 projects.
- You need `rabin2` metadata, imports, strings, mitigations, sections, byte search, or gadget search.
- You need quick patch prototyping or command-line reproducibility.
- You want quick pseudocode through `pdc` or Ghidra-backed pseudocode through `pdg`/r2ghidra.
- The full Ghidra project workflow is unavailable or overkill.

Use JADX when:

- The target is an APK/DEX and the question is about Java/Kotlin application logic.
- You need readable Android source, class names, methods, strings, resources references, or high-level app behavior.

Use apktool when:

- The task is APK unpacking, AndroidManifest.xml, resources, smali, or repackaging.
- You need resource IDs, layouts, or manifest permissions rather than Java decompilation.

Use Frida/dynamic tooling when:

- The question depends on runtime values, anti-debug behavior, decrypted strings, generated requests, hooks, or live app state.
- Static analysis cannot resolve the behavior due to packing, runtime decryption, reflection, or environment-dependent branches.

Use firmware tools when:

- The file is a firmware image, filesystem blob, UEFI/BIOS image, or raw flash dump.
- Prefer `ffind`, `binwalk`, `unblob`, `chipsec`, and filesystem extraction before function-level decompilation.

Use isolated dynamic tooling when:

- Static analysis identifies crypto, compression, parser, or network boundary functions and runtime plaintext/artifacts would answer the question faster.
- Malware behavior depends on timing, privilege/elevation gates, anti-VM checks, scheduled tasks, GUI interaction, or staged execution.
- The sample is live malware; never execute it on the host. Use a disposable VM/snapshot and prefer local fake services/emulators unless real infrastructure contact is explicitly required and safe.

## Availability Checks

Before deep work, check only what matters:

- Ghidra GUI/headless: verify `/opt/ghidra_*/ghidraRun` or `/opt/ghidra_*/support/analyzeHeadless` exists.
- radare2: run `r2 -v` or use `rabin2` directly.
- Android tools: verify `jadx` or `apktool` only when the file is APK/DEX.

If a preferred tool fails, state the reason and use the next fallback. Do not hide fallback behavior.

## Tool Comparison

| Need | Best First Choice | Why |
| --- | --- | --- |
| Fast metadata, imports, strings | radare2/rabin2 | Fast, scriptable, low setup |
| Deep decompilation with persistent annotations | Ghidra | Projects preserve names, comments, labels, types, structs, bookmarks, signatures, and analysis state |
| Free mature decompiler | Ghidra | No commercial license, SLEIGH/p-code, broad support |
| Instruction semantics / IR reasoning | Ghidra p-code or radare2 ESIL | Use Ghidra for p-code/decompiler context; use r2 for quick CLI semantic probes |
| Firmware/odd architectures | Ghidra or firmware tools | Loader/processor coverage and extraction first |
| APK Java/Kotlin | JADX | Best readable app source |
| APK resources/smali | apktool | Manifest/resources/smali focus |
| Runtime secrets/behavior | Frida/debugger | Static tools cannot see runtime-only values |
| Byte-level patch prototype | radare2 | Fast, reproducible write commands and project state |

## Workflow Pattern

For unknown native binaries:

1. Start with cheap triage: `file`, hashes, `rabin2 -I`, `rabin2 -i`, `rabin2 -zz`, or radare2 JSON commands.
2. Create or reuse `.re/RE-NOTES.md` and the repo-local Ghidra/radare2 project before deeper static analysis.
3. Check existing artifact indexes and exported decompiler/resource/protocol notes before rerunning expensive analysis.
4. If deeper static analysis is needed, use one shared Ghidra project under `.re/ghidra` and preserve annotations there.
5. Use the decompiler/IL on a small prioritized set of functions, not the whole binary.
6. Cross-check decompiler claims with strings/imports/xrefs/disassembly.
7. Use dynamic tooling only when static evidence is insufficient; feed runtime-confirmed function roles, plaintext formats, and object layouts back into project comments and notes.

For packed, overlaid, or staged malware:

1. Inventory hashes, sections, overlays, resources, imports, and entropy without executing the sample.
2. Validate embedded-file candidates structurally before trusting raw signature hits.
3. Use static analysis to identify decrypt/decompress boundaries and keys.
4. Use offline scripts or isolated dynamic hooks to recover plaintext/configs, then annotate the static project with the confirmed boundary and data format.

For Android:

1. Use `apktool` for manifest/resources.
2. Use `jadx` for Java/Kotlin logic.
3. Use Ghidra or radare2 for native `.so` libraries.
4. Use Frida when behavior depends on runtime state, anti-tamper, or obfuscation.

For firmware:

1. Identify/extract containers and filesystems first.
2. Inventory binaries and configs.
3. Analyze native executables with Ghidra or radare2 after extraction.

## Reporting Discipline

- Name the tool used for each major conclusion.
- Include file offsets or virtual addresses when discussing code.
- Distinguish direct observations from inferred behavior.
- If the chosen tool fails, record the error and fallback.
- Do not claim Ghidra decompiler evidence if the result came only from radare2 strings/imports.
- Record the shared project path/name and whether `.re/RE-NOTES.md` was updated.
- Mark whether each major claim is raw observation, decompiler interpretation, decoded artifact, runtime observation, or inference.

## Local Notes

- Ghidra GUI: `/opt/ghidra_12.0.4_PUBLIC/ghidraRun`
- Ghidra headless analyzer: `/opt/ghidra_12.0.4_PUBLIC/support/analyzeHeadless`
- Existing local skills: `ghidra`, `radare2`
- Preferred radare2 tools: `/opt/radare2/binr/radare2/radare2` and `/opt/radare2/binr/rabin2/rabin2`.
- Fallback radare2 tools: `/usr/local/bin/r2` and `/usr/local/bin/rabin2`.
