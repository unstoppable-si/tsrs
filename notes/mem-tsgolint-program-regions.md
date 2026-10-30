# mem-tsgolint-program-regions: the lint helper frees each program's regions after linting it (2026-10-29)

Follow-up to the [parallel program construction](perf-tsgolint-parallel-load.md) and
[worker](perf-tsgolint-workers.md) notes, which kept one set of loader regions and every lint worker's region for the
whole request. A request that spans several tsconfig programs therefore held every program's checkers until it
ended: freeing a program (`free_unshared_program`) released its `Program`, but its files, binder data, checkers and
types stayed in the request's regions.

## The change

`run_in_region` gives each program its own `RequestRegions` (loader regions and worker regions), declared before
the program so that it is dropped right after the program is freed. The program built from files without a tsconfig
gets its own set too. A request then holds one program's allocations at a time.

This drops the request-wide loader set that [perf-tsgolint-parallel-load](perf-tsgolint-parallel-load.md) shared
"to avoid a new arena tail per worker per tsconfig": each program now starts new region chunks on each thread it
uses, and gives them all back when it ends.

## Why nothing outlives a program's regions

[mem-build-free-programs](mem-build-free-programs.md) frees each `tsc -b` program's regions the same way, and records
the hazard: data filled lazily into an object that outlives the program goes to that program's region and is read
after the region is freed. In the lint helper:

- Every program parses its own tsconfig and gets its own compiler host, so no `ParsedCommandLine`, parse cache or
  source file is shared between programs. The only run-wide objects are the file system (`cachedvfs` and the source
  overrides, which cache heap strings, booleans and entries), the payload and the rule timings.
- Diagnostics are serialized as they are reported, inside the program, so none outlives it (unlike the `tsc -b`
  error summary, which keeps the program regions of a task that reported diagnostics).
- The process-wide caches that hold arena data (`REGEX_PATTERN_CACHE`, `empty_compiler_options`, the checker's
  primitive alias map) allocate in the thread arena (`enter_thread_arena`), never in a freeable region.
  `SIDE_FLOW` is read only for an identifier whose entry was written with it. The thread-local JSDoc parser keeps
  only its factory and emptied buffers after `recycle`, and the subtree-facts cache is cleared on every use.

## Checks

- `headless::programs_sharing_a_dependency_lint_the_same_after_earlier_programs_are_freed`: three programs that
  share a module (JSDoc, types the rules resolve) are linted in one request with `TSRS_ARENA_POISON=1` at one and two
  workers, and must produce exactly what linting each program alone produces. A deliberate read of the previous
  program's source file after it is freed makes the test fail (the helper crashes on the poisoned memory).
- A large private monorepo, linted through Oxlint (`OXLINT_TSGOLINT_PATH`) with its typed rules: with
  `TSRS_ARENA_POISON=1` and debug assertions, sorted, reversed and two shuffled program orders at 1, 2, 8 and 28
  workers, each request linting every program twice in one process; and an AddressSanitizer build (system
  allocator, so region drop lists free through ASan) at four order and worker combinations. No crash, assertion or
  ASan report; the second pass matched the first for every program, and diagnostics matched tsgolint. The same
  deliberate stale read crashes under poison mode, SIGBUS without it, and is reported by ASan as a heap
  use-after-free.

## Numbers

Apple M3 Ultra (28 logical CPUs, 96 GiB), release builds, the large private monorepo above through Oxlint 1.89.0,
default workers. Single runs on a loaded machine, not hyperfine; instructions retired were not measured.

| | wall | helper peak RSS |
| --- | --- | --- |
| base | 13–19 s | 7.5–12.2 GiB |
| head | 13.2–15.8 s | 3.9–4.1 GiB |

At 2 workers the head peaks at 2.1 GiB (31 s), at 8 workers at 3.0 GiB (11.4 s). A single-program request, such as
the `bench/lint.mjs` projects, allocates the same as before.
