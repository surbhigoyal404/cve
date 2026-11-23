Review: Static CVE Agent

The file you sent is the design spec for the Static CVE Agent, not the docAssist RAG in this folder. The agent's code is not on this machine, so this review is based on the spec. Where it says "if the code does X", that still needs checking in the code.

The short answer

The core design is sound: only the version and build rules (rungs R1–R3) can close a CVE, the AI has no authority, and every claim must cite evidence. The problems come from five places:

The test setup can't see wrong answers. The acceptance target ("31 closed / 3 / 0 / 66 with 0 errors") counts which bucket each CVE lands in. It never checks whether a verdict was right.
Version rules treat "not listed" as "safe". That leads to wrong fixed / not_applicable verdicts.
Absence checks miss copies. Statically linked copies, mixed Linux + VxWorks targets and the wrong base image can all make a component look absent when it isn't.
The kernel "is this code built?" check parses Kbuild Makefiles by hand. Real build data would be exact.
The reachability tools are weak. This is the "not working well" part, and why 66 of 100 CVEs need a person. Joern is usually off, the caller walk only matches names, and the AI protocol is too heavy for 10 steps.
1. Measure wrong answers first
The eval sample already has a ground_truth column. Report every closed verdict against it, split by rung (R1/R2/R3). Any closed verdict that disagrees with a person's label is the top-priority bug.
Turn every wrong case into a fixture test.
"0 errors" may be false comfort. The source and recipe_source stages catch all exceptions and record a limitation, so a real bug shows up as a quiet undetermined. Keep expected errors (missing tool, unreadable artifact) separate from unexpected ones. Count unexpected ones as eval errors, and fail tests on them in a strict mode.
2. Causes of wrong closures (most dangerous first)
#	Problem	Fix
a	"Outside the listed range" is treated as unaffected. In the CVE JSON 5 schema, a record with no defaultStatus means unlisted versions are unknown. Many upstreams list only the branches they still support, so an end-of-life branch you ship may simply be missing.	Close R2 / R3-version only on an explicit unaffected entry or defaultStatus: unaffected.
b	NVD has no "unaffected" concept. Its versionStartIncluding is often a guess, so "below the start version → not introduced" is unreliable.	Never auto-close "not introduced" from NVD alone.
c	Version parser bug. I ran the spec's regex: 1.1.1a and 1.1.1b come out as alpha/beta builds that sort before 1.1.1, while 1.1.1c sorts after it. 2.0b and 1.2.3a also read as pre-releases. That can wrongly give "not introduced" for OpenSSL. 1.0.2zf and 5.15.0-rt17 can't be parsed (safe, but they never close).	Use a version scheme per component, not one regex: univers (Apache-2.0, many schemes) or repology's libversion. Record the scheme in components.yaml. Test against real release lists.
d	Non-exact plan. With only --release, the agent may analyse LATEST_SUCCESSFUL_PLAN_ID, which may not be the build that shipped.	Don't auto-close unless the plan is exact.
e	Profile detection stops at the first match. A Product B target with both a Yocto manifest and a VxWorks _appl.elf is treated as Yocto only, so the VxWorks side is never inventoried.	Detect every profile that matches. Coverage is "complete" only when all of them are covered.
f	Overlay base image. It "uses the first safe directory" from requires, which may be the wrong base.	Require exactly one match, or check all of them.
g	Statically linked or vendored copies (zlib inside other binaries, Go/Rust binaries, firmware blobs) don't appear in manifests.	Before calling copy coverage complete, scan binaries for copies: cve-bin-tool string checkers, go version -m, cargo-auditable.
h	VxWorks R3 relies on a missing symbol name, which breaks with inlining, LTO, renames or stripping.	Also require the source/object to be missing from the linker map. When debug info exists, check DWARF for inlined copies.
i	The kernel Kbuild owner-chain parser misses files listed in several obj- lines, composite objects (foo-y), #include "x.c", and #ifdef inside a built file.	Use the real list of compiled files: DWARF compile units of vmlinux and every .ko, or Yocto's SPDX_INCLUDE_COMPILED_SOURCES. OE-core already has improve_kernel_cve_report.py, which does exactly this. Its authors report about 70% fewer false kernel CVEs (not independently measured).
j	Fix-patch picking uses a keyword score (≥6 and ≥2× the runner-up). A wrong commit means wrong files, which means a wrong R3.	Prefer data the CVE authority supplies: kernel CVE records carry programFiles and git commit ranges, and OSV carries fix commits. Never close R3 from a patch the heuristic chose.
k	Stale advisory cache. When the network fails it falls back to the cache, and CVE records do get corrected later.	Record the advisory's age in the verdict and reopen when the CVE record changes.
3. Biggest precision win: check fix commits, not version numbers
Kernel: the spec says the kernel cache is a blobless clone, which keeps full commit history. That makes git merge-base --is-ancestor <fix> <pinned-sha> cheap.
For vendor kernels that cherry-pick fixes, search the log for commit <sha> upstream or cherry picked from commit trailers, or compare with git patch-id --stable.
The kernel's own vulns repo pairs each CVE's introducing and fixing commits per stable branch.
Any component pinned to a git commit: run the same ancestry check with the fix commit from OSV.
Patch presence in modified copies: add Google Vanir (signature-based, C/C++, uses OSV data) as a second opinion next to git apply --check, before falling back to the AI patch judge. Google reports 2.72% false-alarm signatures.
4. Why reachability "doesn't work well"
Joern is effectively off. JOERN_BUILD_ON_DEMAND=false plus no prebuilt graph means a JoernError in most runs. c2cpg also parses badly without the real include paths and defines.
Fix: prebuild graphs nightly in CI for each pinned commit, and pass the real include paths and defines from the build (log.do_compile, bear, or CMake's compile_commands.json).
I couldn't confirm whether your Joern version's c2cpg can read a compile database directly. If not, --include and --define still work.
The caller walk is name-only (3 hops, 30 nodes, 40 s), so it misses function pointers, macros and constructors.
Cheap upgrade: GNU Global, or a clangd index built from compile commands.
Bigger upgrade: an LLVM call graph from bitcode captured during the build (gllvm), with PhASAR or SVF to resolve function pointers.
CodeQL is strong for C/C++, but automated use on closed-source code needs a GitHub Advanced Security licence.
The hand-written PowerPC decoder (ppc_calls.py) handles bl/b, lis/addi pairs and data words. It misses mtctr/bctrl indirect calls, jump tables and relocations. Replace it with Ghidra headless (Apache-2.0, good PowerPC and VxWorks support), or angr / Capstone. Analysing what actually ships also avoids source-vs-build mismatches.
Cut review time without breaking the rules. product.yaml already has policy.rungs_closable. Use it for a pre-approved pattern that a person confirms with one click. Example: a library CVE where no shipped binary imports the symbol, there's no dlopen string, no static copy and no language binding.
5. AI layer
The model is fighting the protocol. Taking the outermost {…} with one retry, plus 3 free invalid replies, is a sign of that. Switch to native tool calling with schema-constrained output (vLLM structured outputs), or wrap the existing Pydantic models with instructor or PydanticAI.
Too much bookkeeping for 10 steps. Each call needs a requirement_id and question, plus dimensions, 8 witness aspects and 5 timing stages. Let the orchestrator record tool results itself; the model just picks the next tool and proposes claims at the end.
Give it smaller jobs. Instead of free exploration, have it answer specific open questions the graph tools produce, such as "which functions can be stored in ops->handler?" or "is this length checked before memcpy?". Those answers are small, checkable and cacheable.
History trim drops working context. Keeping only the first 2 and last 8 messages loses mid-run tool results, which leads to repeated calls.
Cache key and model choice. Key the LLM cache on the actual model version, not the alias qwen. Compare models on labelled reachability cases; the Copilot transport already allows that.
6. Smaller engineering issues
The fingerprint uses a fixed AGENT_VERSION, so code fixes don't change it. Add the agent's git commit and tool versions.
Passing a raw CVE5 file as --advisory-file silently produces no advisory. It should fail loudly.
raw_applicability.py and binary_fixes.py are hard-coded for one zlib CVE. Move them into reviewed knowledge-base facts.
adapters/yocto.py is 1,573 lines and should be split.
7. Longer-term architecture: have the build produce the evidence

Yocto 5.1+ emits SPDX 3.0 SBOMs. Combined with vex.bbclass and the compiled-sources list, it records patched CVEs and compiled files at build time. sbom-cve-check from Bootlin is a cve-check replacement that works on that SBOM. Archive this small evidence bundle with each release and let the agent read it, instead of reverse-engineering NFS trees. That also fixes the problem of plans being purged.

Suggested order
A correctness eval with per-rung comparison against ground_truth.
The CVE5/NVD rules and the version parser (2a–2c). These are small changes with the highest risk.
The plan, profile and overlay guards (2d–2f).
Kernel: compiled-file list plus commit-ancestry checks.
The binary scan for copies.
Joern prebuild with real build flags, and Ghidra for VxWorks.
Simplifying the AI protocol.

To pin down the exact cause of the wrong answers you're seeing, I need the failing CVEs or the code. I can also turn this review into a shareable page if you want to pass it to your team.
