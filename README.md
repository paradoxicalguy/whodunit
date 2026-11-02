# whodunit

*your build changed. the source didn't. so who did it?*

> **work in progress** — this repository is in the early stages of development. the ideas below describe what whodunit is being built to do, not a list of features you can already install and run.

## what is this thing?

you build a compiler. you build it again. same source, same inputs, same configuration.

and somehow the two binaries are different.

cool. very cool. exactly what we wanted.

now you have two enormous artifacts, a pile of build logs, and a question that `sha256sum` is spectacularly bad at answering:

**what actually changed, and where did that difference come from?**

that's the question behind **whodunit**.

whodunit is an experimental tool for investigating **build reproducibility failures**, with a particular focus on the Rust compiler and its toolchain. the goal is to turn "these files have different hashes" into a useful explanation of *which part of the build introduced the difference*.

not just *something is different*. thank you, hash function. we noticed.

## wait, what's a reproducible build?

a build is reproducible when the same declared inputs and build process produce the same output bytes, even when somebody else runs the build in an equivalent environment.

sounds reasonable. computers are deterministic, right?

unfortunately, builds have a talent for remembering things nobody asked them to remember:

- absolute filesystem paths
- timestamps and archive metadata
- directory traversal or archive member ordering
- compiler and linker metadata
- debug information, including DWARF and CodeView
- details of the build environment that accidentally become part of the output

sometimes the compiled program behaves identically, but its binary representation doesn't. that still matters if you're trying to verify that an artifact really came from a particular source tree.

## the crime scene

imagine building the same Rust toolchain in two different places:

```text
/work/build-a/rust/
/work/build-b/extra1/extra2/extra3/rust/
```

both builds use the same source revision. ideally, the resulting artifacts agree byte for byte.

but suppose a compiler library doesn't match.

```text
build a: librustc_driver-....so  → hash A
build b: librustc_driver-....so  → hash B

hash A != hash B
```

well, great. we've established that two very large files are not the same file.

what we actually want to know is whether the mismatch comes from a source path embedded in `.debug_line`, an archive member appearing in a different order, a build-script input, or something else entirely.

**whodunit is being designed to help narrow that search.**

## what i'm trying to build

rather than treating every mismatched artifact as an opaque blob, the plan is to build up a chain of evidence:

1. **compare builds** — identify which outputs differ, rather than dumping a wall of hashes and calling it a day.
2. **look inside the artifacts** — inspect formats and metadata that commonly hide nondeterminism, including archives, object files, and debug information.
3. **explain the difference** — distinguish meaningful code changes from things like paths, timestamps, ordering, and build metadata where possible.
4. **connect it to Rust** — understand enough of Rust's bootstrap process and toolchain layout to make the findings useful to compiler contributors.
5. **keep checking** — work toward continuous reproducibility verification so regressions are noticed before they become archaeological expeditions.

this is the direction, **not a claim that all five steps are implemented**.

## why Rust?

because building a compiler once apparently wasn't enough suffering.

more seriously: i was looking for a GSoC project when i stumbled upon reproducible builds in Rust's GSoC README. at the time, i knew absolutely nothing about reproducibility. didn't even know it was a thing.
so naturally, like every sane person would do, i followed my curiosity.
and it took me to some horrific places;
like reading `.llvmbc` sections, staring at DWARF debug information, comparing binaries that were **supposed** to be identical, and trying to figure out why a compiler was apparently leaving little souvenirs of its build environment inside the final artifacts. or just trying to figure out why was i even doing this to myself.

and the deeper i went, the more i realized that finding a mismatch is _usually_ the easy part. figuring out where it came from is where the suffering begins.

which is more or less how whodunit happened.

i didn't originally set out to build a tool for investigating reproducibility failures. i just wanted to understand why two builds of the same compiler couldn't agree on their own bytes like my thought can't agree whether to eat fried rice or pizza for dinner.

unfortunately, i still want to know.

## how does this relate to repro-check?

there is related experimentation in my Rust fork around comparing toolchain builds from different source locations and diagnosing mismatched stage2 artifacts.

whodunit is a **separate project** growing out of that general reproducibility work. my work on the `repro-check` branch is an independent continuation and modification of an earlier experiment; i'm not claiming authorship of the original project.

- [Rust fork / reproducibility experiments](https://github.com/paradoxicalguy/rust/tree/repro-check)
- [Whodunit repository](https://github.com/paradoxicalguy/whodunit)

## project status

**early development / design in progress.**

there is no documented, stable CLI or installation procedure yet. if you're reading this hoping to `cargo install whodunit` and immediately interrogate a suspicious binary: not quite yet.

current areas of interest include Rust-aware artifact comparison, build-path leakage, archive determinism, DWARF/CodeView differences, LLVM-related variation, and reproducibility checks that can run repeatedly in CI.

as implementations land, this README will grow real examples, supported formats, commands, and limitations. anything else would just be writing documentation for a tool that exists primarily in my head, which seems like a bold strategy.

## contributing / following along

if you're working on reproducible builds, Rust bootstrap, binary formats, or toolchain diagnostics, feel free to follow the repository or open an issue to discuss a reproducibility failure.

concrete examples of mismatched artifacts and the conditions that produced them are especially useful. a reproducible reproducer, if you will. sorry.

## the actual goal

**a different hash should be the beginning of an investigation, not the end of it.**

whodunit is an attempt to answer the next question: *who—or what—put those bytes there?*
