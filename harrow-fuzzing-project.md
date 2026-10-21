# Harrow — coverage-guided fuzzing for Ada and SPARK, written in Ada

*Project document, 21 October 2026. "Harrow" is a working name, to check
against existing projects before publication.*

> **Status of every claim in this document.** Every technical fact about
> the compiler is **measured** (GNAT 13.3.0, `x86_64-linux-gnu-gcc-13`,
> Linux x86-64, 21 October 2026); the commands are in Appendix A. Claims
> about other tools (AFL++, libFuzzer, honggfuzz, commercial Ada fuzzers)
> are marked **[to confirm]**: they come from no measurement, and
> milestone M0 checks each against the tool's own documentation before
> anything rests on it.

---

## 1. Summary

Harrow finds the inputs that make an Ada parser wrong. It is
coverage-guided and runs in-process, and its whole engine is written in Ada:

- the **GCC instrumentation** GNAT already carries is used as it is:
  `-fsanitize-coverage=trace-pc,trace-cmp`, and `-fsanitize=address` when
  memory errors are in question;
- a **runtime written in Ada** receives the coverage callbacks;
- a **mutation engine** keeps every input that reaches new code;
- **oracles** say when an output is *wrong*, not only when the program
  crashes: round-trip, differential against a reference, contracts
  executed, properties.

A target is an Ada function from bytes to a result, wrapped in an
instance of one generic package. A failure becomes a reproducer, minimised
and written as a test that the project's suite runs from then on.

---

## 2. The problem

### 2.1 What proof gives, and what it does not

SPARK can prove that a parser has **no run-time error**: no index out of
range, no overflow, no exception, whatever the bytes. For code that reads
untrusted input, that is the first property needed, and it is a strong
one.

It does not prove that the parser **reads what the bytes mean**, unless
a postcondition states it, and functional postconditions are rare in
practice. A parser that refuses every valid message is proved exactly as
well as a correct one.

### 2.2 The defect shapes proof and unit tests miss

The following shapes recur in parsers written in Ada, proved or not.
Each is the kind of defect Harrow is built to find:

| shape | why ordinary tests pass | what finds it |
|---|---|---|
| **A slice's bounds taken as 1-based.** Ada slices keep their parent's bounds, so an offset applied "to correct" shifts every field by one. | Tests that only check refusals pass: a parser wrong in one direction refuses garbage just as well. | a differential oracle against a reference parser |
| **A numeric bound checked one step too late**: accumulating decimal digits while testing `Value > Integer'Last / 10` lets one more digit overflow. | No fixture drives the numeric limit. | mutation of a digit string, with the limit constant handed to the mutator by comparison tracing |
| **An index from `Ada.Strings.Fixed.Index` on a slice treated as relative.** The answer is already absolute, so the next slice runs past the end. | Hand-written fixtures happen to sit where the error cancels. | the first input of a realistic shape |
| **An encoder wrong only for certain bit patterns** (for example a base64 sextet wrong when a byte's two low bits are set). | Every fixture ends on a byte whose low bits are zero. | a round-trip oracle on random bytes |
| **A local declared as `String := ""` and assigned later**: a run-time length check fails, and a `when others` handler returns an empty result. | The handler turns the defect into a plausible "nothing found". | an oracle that reports a swallowed exception |
| **A guard that always returns the empty answer.** | Fixtures that all expect an empty answer agree with it. | a non-vacuity property: some input must produce a non-empty answer |

The two shapes that recur most, a strict parser that is wrong in one
direction and a total function that raises, are exactly what
coverage-guided fuzzing with oracles finds in minutes.

### 2.3 Why not an existing tool

- **Nothing in the Alire index fuzzes.** Measured with `alr search
  --crates`, 21 October 2026: no crate matches "fuzz".
- **libFuzzer's driver is out of reach of GCC.** GNAT 13.3 refuses
  `-fsanitize=fuzzer` (measured: "unrecognized argument"), and the
  instrumentation libFuzzer expects is Clang's **[to confirm]**.
- **AFL++** offers a GCC plugin mode **[to confirm: whether it accepts
  the Ada front end, and which GCC versions]**. It works on whole
  processes: for pure parsing functions, a fork per input costs orders of
  magnitude in throughput **[to measure]**. It knows nothing of Ada's
  exceptions, contracts or types.
- **Commercial Ada fuzzers exist** for proprietary toolchains **[to
  confirm: names, scope, licence]**. Harrow targets FSF GNAT, which
  anyone can install.

Harrow does not compete with AFL++. It can later export its corpus to it,
or take AFL++ as a back end (§11, M7).

---

## 3. Goals and non-goals

### Goals

1. **Fuzz any Ada subprogram from bytes to a result**, in process, with
   coverage feedback, on FSF GNAT 13 and 14, without Clang.
2. **Say when an answer is wrong**, not only when the program dies. The
   oracles are first-class, and a target without one is flagged as
   weakly checked.
3. **Turn every failure into a regression test**: minimised input, Ada
   test source, ready for the project's test directory.
4. **Run unattended in CI**, bounded in time and memory, with a stable
   exit code and a report a human reads in one minute.
5. **Stay small and readable**: the engine fits in one review sitting.
6. **Be honest about coverage**: report which targets were fuzzed, for how
   long, and how many edges each reached, so that "fuzzed" is a number and
   not a claim.

### Non-goals

- Fuzzing whole binaries, network services or the kernel. AFL++ and
  syzkaller exist for that.
- Grammar inference, symbolic execution, concolic testing.
- Replacing SPARK proof. Harrow covers what proof leaves open
  (correctness), proof covers what fuzzing samples (all inputs).
- Windows and macOS at first. Linux x86-64 first, then aarch64.

---

## 4. The measured foundations

All measured on 21 October 2026, GNAT 13.3.0 (Appendix A has the commands).

| question | answer |
|---|---|
| Does GNAT accept GCC's coverage instrumentation on Ada? | **yes**: `-fsanitize-coverage=trace-pc` compiles an Ada unit; the object references `__sanitizer_cov_trace_pc` |
| ... comparison tracing? | **yes**: `trace-cmp` emits `__sanitizer_cov_trace_cmp4`, `_cmp8`, `_const_cmp1`, `_const_cmp4` on a 15-line test unit (the sizes vary with the code) |
| Can the callbacks be written in Ada? | **yes**: a package exporting `__sanitizer_cov_trace_pc` with `Convention => C`, compiled **without** the flag, counted 17 calls in the instrumented test |
| AddressSanitizer? | **yes**: `-fsanitize=address` compiles, links and runs a GNAT program |
| UndefinedBehaviorSanitizer? | accepted, no runtime symbol on Ada (Ada's own checks cover that ground) |
| libFuzzer driver (`-fsanitize=fuzzer`)? | **no**: refused by GCC 13 |

Two design consequences:

1. **`trace-pc` gives the program counter of each basic block, not an
   edge.** Harrow hashes (previous PC, current PC) itself, the classic
   edge-coverage approximation, in a fixed-size map of counters.
2. **The runtime must not be instrumented**: it is a separate project
   compiled without the flag. That is how the measurement was made, and
   it is a rule the build enforces (§9).

---

## 5. Architecture

```
            ┌──────────────── target project (instrumented) ───────────────┐
            │  Parser under test ...        Harness instance (generic)     │
            │        ▲  __sanitizer_cov_trace_pc / _cmp*                   │
            └────────┼──────────────────────────────▲───────────────────────┘
                     │ callbacks                    │ calls Target (Data)
            ┌────────┴─────────── Harrow runtime (not instrumented) ───────┐
            │  Coverage map   Comparison log   Watchdog   Oracle results   │
            └────────┬──────────────────────────────▲───────────────────────┘
                     │ new edges?                   │ next input
            ┌────────┴──────────── Harrow engine ──────────────────────────┐
            │  Corpus ── Scheduler ── Mutators ── Dictionary (from cmp)    │
            │  Minimiser   Crash store   Reporter   Test writer            │
            └──────────────────────────────────────────────────────────────┘
```

### 5.1 Components

| component | role | Ada unit (proposed) |
|---|---|---|
| Coverage runtime | receives `trace_pc`, folds (prev, cur) into a 64 KiB counter map, bucketed hit counts (1, 2, 3, 4-7, 8-15, ...) | `Harrow.Coverage` |
| Comparison log | receives `trace_cmp*` and `const_cmp*`, keeps the last N operand pairs: the constants an input must match become dictionary entries | `Harrow.Compare` |
| Harness | the generic a target instantiates: the target function, its oracles, its seed directory | `Harrow.Harness` |
| Engine | the loop: pick, mutate, run, judge, keep | `Harrow.Engine` |
| Corpus | inputs that reached new coverage, on disk, content-addressed (SHA-256 name), with their edge count and size | `Harrow.Corpus` |
| Scheduler | weights inputs by recency, rarity of the edges they hold, and size | `Harrow.Schedule` |
| Mutators | byte-level, dictionary, splice, length-aware, structure-aware plug-ins | `Harrow.Mutate` (+ children) |
| Oracles | crash, exception, timeout, memory, contract, round-trip, differential, property | `Harrow.Oracle` (+ children) |
| Minimiser | shrinks a failing input while it still fails *the same way* | `Harrow.Minimise` |
| Reporter | text and JSON report, exit status | `Harrow.Report` |
| Test writer | emits an Ada test procedure and the input as a fixture | `Harrow.Regression` |
| Driver | command line: `harrow run`, `replay`, `minimise`, `merge`, `report` | `harrow` main |

### 5.2 The loop

```
load seeds; run each once; keep those that add coverage
loop until time budget, run budget, or a failure (unless --keep-going):
   I := Scheduler.Pick
   M := Mutators.Apply (I, Dictionary)   -- 1..k stacked mutations
   reset coverage map; arm watchdog
   Result := run Target (M) under the oracles
   if Result is a failure    -> Minimise, store, report
   elsif new edges or better counts -> Corpus.Add (M)
   feed Compare's operand pairs into Dictionary
```

### 5.3 In process, and what that costs

Running the target in the engine's own process is what makes a pure
reader fuzz quickly: no fork, no exec, no pipe. The price is that one
target's failure must not corrupt the engine:

- **An Ada exception** is caught around the call, and that is the common
  case in Ada.
- **`Storage_Error` from a stack overflow**: GNAT turns stack exhaustion
  into `Storage_Error` when stack checking is on (`-fstack-check`)
  **[to measure on GNAT 13 and 14, with tasks]**. The target runs in a
  task with a fixed stack, so a runaway recursion is caught there.
- **A hang**: a watchdog task aborts the target task after the timeout.
  Abort is deferred inside abort-deferred regions **[Ada RM 9.8, to read
  before relying on it]**. A target that cannot be aborted is reported as
  a timeout, and the engine restarts itself from its on-disk state.
- **Memory corruption**: there is none in SPARK and little in Ada, but a
  target may bind C. `--asan` rebuilds with AddressSanitizer, and any ASan
  report is a failure.
- **State left behind by a target** (globals, protected objects): a target
  must be a pure function of its input, or declare a `Reset` that the
  harness calls before each run. A determinism oracle (§6.7) catches the
  ones that are not.

A **fork mode** (`--fork`) runs each batch in a child process, for
targets that need it, at a cost to measure.

---

## 6. Oracles: how Harrow knows an answer is wrong

The heart of the project. A fuzzer that only catches crashes would find
little in SPARK units, which are proved not to crash. The value is in the
oracles.

### 6.1 Crash and exception (always on)

Any exception escaping the target is a failure, **including in non-SPARK
code where the exception is "expected"**: the target declares which
exceptions are its contract (`Raises => (Parse_Error'Identity)`), and any
other one, `Constraint_Error` above all, is a defect.

### 6.2 Contracts executed

Targets are built with `-gnata`, so preconditions, postconditions,
predicates and `Assert` are checked at run time. A postcondition proved
by SPARK cannot fail; one that is not proved, and every one in non-SPARK
code, can. A failed contract is reported with the contract's own location.

### 6.3 Round-trip

For a reader with a writer (an address-resolution request builder and
its reader, a base64 encoder and decoder, a file-system writer and
reader, protobuf, HPACK):

```
for any input X that Read accepts:  Read (Write (Read (X))) = Read (X)
for any value V:                    Read (Write (V)) = V
```

The second form needs a generator of values (§7.5).

### 6.4 Differential

Two implementations must agree. For network formats the references are
outside Ada: Wireshark's dissectors through `tshark -T json` **[to confirm
the fields' stability]**, Python's `scapy` **[to confirm]**, `rpmdev-vercmp`
for RPM version ordering, `dpkg --compare-versions` for Debian's. The
oracle runs the reference out of process, in batches, on the inputs that
reached **new coverage**, so throughput stays with the in-process loop.
A disagreement is a failure, and the report shows both answers field by
field.

The rule behind it: judge a parser against a third implementation, never
against its own output.

### 6.5 Properties

Statements the target's author writes in Ada, checked on every input:

```ada
--  A DHCP reader: a parsed option's span lies inside the frame.
Property => (for all F of Info.Fields => F.First >= Data'First
                                         and F.Last <= Data'Last)
```

And the **non-vacuity property**: across the run, the target must
produce at least one non-empty, accepted answer. A reader that refuses
everything fails the run, with "0 of N inputs accepted".

### 6.6 Handler-swallow

A target's own `when others => return Empty` hides the defect. In
`--strict-handlers` mode, Harrow's runtime is told when a handler is
entered in instrumented code **[to design: GNAT's `Ada.Exceptions` hooks
or a compiler switch]**, and an exception swallowed on an input that the
oracle classes as well-formed is reported.

### 6.7 Determinism

Each corpus input is re-run once at the end of a session. A different
answer for the same input means hidden state, and that is a failure.

### 6.8 Resource bounds

Per input: a time limit (default 1 s) and a memory high-water mark (the
peak resident size sampled by the watchdog). Over the bound is a failure:
it catches the parser whose cost is quadratic in its input.

---

## 7. Mutation

### 7.1 Byte level (always on)

Bit flips (1, 2, 4), byte flips, arithmetic ±1..35 on 8/16/32/64-bit
words in both byte orders, the interesting values (0, 1, 0x7F, 0x80,
0xFF, 0x7FFF, 0x8000, 0xFFFF, 2^31-1, 2^31, 2^32-1, ...), insertion,
deletion, duplication of a block, and overwriting with a block from
elsewhere in the input.

### 7.2 Dictionary from `trace-cmp`

Every comparison the target makes against a constant (`const_cmp*`) adds
the constant to the dictionary, in both byte orders. That is how a magic
number such as DHCP's `99.130.83.99` becomes reachable without a seed.
Two-operand comparisons (`cmp*`) add the side the input does not already
contain (input-to-state correspondence). User dictionaries are accepted
in the AFL format **[to confirm the format]**.

### 7.3 Splice

The head of one corpus input joined to the tail of another, which finds
the message that is valid up to a point and then switches structure.

### 7.4 Length-aware

Many network formats are TLV (BER, LLDP, CDP, DHCP options, HPACK,
protobuf). A plug-in told "a length of size S at offset O covers the rest
of the TLV" mutates the value and fixes the length, or breaks the length
on purpose: too short, too long, zero, maximum. The plug-ins are described
per format in a few lines of Ada.

### 7.5 Structure-aware (optional per target)

A target may provide a **generator** (`Gen (Rand) return Value`) and a
**writer** (`Write (Value) return Bytes`). Harrow then mutates values,
writes them, and feeds the bytes. That is the only way to go deep into
formats with checksums or nested lengths, and it is also what the
round-trip oracle's second form needs.

### 7.6 Seeds

Seeds come from a directory of raw inputs. `harrow seed pcap FILE` splits a
pcap or pcapng into one input per frame (link layer kept), so any capture
becomes a corpus.

---

## 8. The harness API

```ada
generic
   type Result (<>) is private;
   with function Target (Data : Byte_Array) return Result;
   --  Optional pieces, each defaulted to "none":
   with function Accepted (R : Result) return Boolean is Always;
   with function Same (L, R : Result) return Boolean is "=";
   with function Property (Data : Byte_Array; R : Result) return Boolean
     is Always_True;
   with function Reference (Data : Byte_Array) return String is No_Reference;
   with function Image (R : Result) return String is No_Image;
   Name : String;
package Harrow.Harness is
   procedure Main;   --  the command line of `harrow`, for this target
end Harrow.Harness;
```

A target file reads:

```ada
with Harrow.Harness;
with My_Parsers.DHCP;
procedure Fuzz_DHCP is
   package H is new Harrow.Harness
     (Result    => My_Parsers.DHCP.Lease_Info,
      Target    => My_Parsers.DHCP.Read,
      Accepted  => My_Parsers.DHCP.Is_Valid,
      Property  => Fields_Inside_Frame,
      Reference => Tshark_DHCP,
      Image     => My_Parsers.DHCP.Image,
      Name      => "dhcp");
begin
   H.Main;
end Fuzz_DHCP;
```

`Byte_Array` is Harrow's own (`array (Natural range <>) of
Interfaces.Unsigned_8`). A target taking its own array type converts in a
one-line adapter, or Harrow's generic is instantiated on the target's
element and index types (a second generic, `Harrow.Harness_For`).
Procedures with `out` parameters (`Parse (Buf, Info)`) are wrapped the
same way.

---

## 9. Build and integration

### 9.1 Two projects, always

```
harrow_runtime.gpr   the runtime and engine -- NEVER instrumented
fuzz_targets.gpr     with "harrow_runtime"; the code under test compiled
                     with -fsanitize-coverage=trace-pc,trace-cmp -gnata
                     (+ -fsanitize=address for --asan builds)
```

A project-level check refuses a build where a Harrow unit carries the
coverage flag: the callbacks would call themselves. A rule that matters
is enforced by a check, not left to a sentence in the README.

### 9.2 Alire

Published as the crate `harrow` **[name to check]**, with an `alr` action
that builds the target project in its fuzzing profile. FSF GNAT 13 and 14
from Alire's toolchain are the supported compilers, and the CI matrix
covers both.

### 9.3 In a user's project

- `fuzz/` at the project root: one `fuzz_<target>.adb` per parser, plus
  `fuzz.gpr`.
- A make target or an `alr` action: `fuzz TARGET=dhcp SECONDS=60`, and
  `fuzz-all SECONDS=30` for every target.
- **Nightly CI**: every target, a fixed budget each, the corpus kept
  between runs in the CI machine's cache. A failure fails the job and
  attaches the minimised input. Run on a schedule rather than on every
  push, so the push pipeline keeps its time.
- **On every push**, a short replay of the stored corpus (seconds): a
  regression is caught without fuzzing.
- **A control the project may adopt**: a parser unit without a fuzzing
  target is refused by the project's verification script.
- **A failure found becomes a test**: `harrow regression` writes the test
  and the fixture, reviewed like any other.

### 9.4 Which targets first

By exposure, then by what proof leaves open:

1. Readers of raw network frames: link layer, address resolution,
   neighbour discovery, DHCP, DNS. Anyone on the segment can send them.
2. Radio: radiotap, 802.11, Bluetooth Low Energy advertising. Anyone in
   range can transmit.
3. Answers to active probes: BER/SNMP, TLS, SSH identification, HTTP/2 and
   HPACK, protobuf. A reader that binds C gets an `--asan` build.
4. Files a user supplies: pcap, FAT, GPT.
5. Version-string comparators (RPM, Debian), with differential oracles
   against the distributions' own tools.

---

## 10. Reports

```
harrow: dhcp  60 s  1,284,113 runs (21,402/s)  edges 1,931 (+12 last 10 s)
        corpus 214 inputs (from 37 seeds)  dictionary 88 entries
        oracles: exception, contract, property, differential(tshark), non-vacuity
        accepted 41.2 % of inputs        FAILURES: 1
  [1] differential  fuzz-out/dhcp/fail-3f9a...bin (minimised 74 -> 19 bytes)
      field lease_time: ours 4294967295, tshark 0xffffffff "infinite"
      replay:  harrow replay dhcp fuzz-out/dhcp/fail-3f9a...bin
      test:    harrow regression dhcp fail-3f9a... > tests/...
```

The figures in that sample are illustrative, not a measurement. Throughput
targets are set in M2 from a measurement, never before.

The same fields are available as JSON, for CI and dashboards.

---

## 11. Roadmap

Each milestone ends on a measurement, never on a claim.

| milestone | content | done when |
|---|---|---|
| **M0** documentation | read GCC's `-fsanitize-coverage` documentation, AFL++'s (GCC plugin, dictionary format), libFuzzer's (corpus format, `-merge`), Ada RM 9.8 (abort), GNAT's stack checking | each [to confirm] point of this document answered from its source, and the document amended |
| **M1** runtime | `Harrow.Coverage` and `Harrow.Compare` with the callbacks, the edge map, a test target | a planted branch (`if B(1)='F' and B(2)='U' and B(3)='Z'`) is reached from an empty seed by dictionary and coverage alone, within a budget measured and written down |
| **M2** engine | corpus, scheduler, byte mutators, splice, exception and timeout oracles, in-process loop with watchdog task, `run` and `replay` | throughput measured on a small TLV reader and written in the README; the run is stable for one hour |
| **M3** oracles | contracts (`-gnata`), properties, non-vacuity, determinism, minimiser, `regression` | **the planted-defect benchmark** (§12) passes |
| **M4** a first real user | a set of published network parsers (see the companion project *spark-wire*), seeds from public captures, nightly CI | five targets fuzzed nightly for a week, every failure triaged |
| **M5** differential | the out-of-process reference oracle; `tshark`, `rpmdev-vercmp`, `dpkg` adapters | an RPM version comparator agrees with `rpmdev-vercmp` on a 10^6-input run, or the disagreements are filed |
| **M6** structure | length-aware plug-ins for BER, TLV, protobuf; generator and writer API; round-trip oracle's second form | HPACK and protobuf targets reach measurably more edges than byte-only mutation (the figure written down) |
| **M7** ecosystem | Alire crate, aarch64, fork mode, corpus export to AFL++ / libFuzzer formats [to confirm formats], `--asan` builds | published; the README's figures reproduced by a stranger's CI |

---

## 12. Testing Harrow itself: the planted-defect benchmark

A fuzzer is judged by the defects it finds, not by the edges it counts.
Harrow's own test suite **plants the defect shapes of §2.2**, each as a
small Ada unit with a correct twin:

| planted defect | oracle that must catch it | budget |
|---|---|---|
| slice taken as 1-based | differential against the correct twin | to set in M3 |
| digit bound checked at `'Last / 10` | exception | idem |
| `Index` on a slice taken as relative | exception | idem |
| base64 sextet wrong on low bits | round-trip | idem |
| empty `String` assigned later, handler returns empty | handler-swallow | idem |
| a guard that always returns `""` | non-vacuity | idem |

The budgets are set from the first measurement and then held: a change
that makes Harrow slower to find one of them fails Harrow's CI.

---

## 13. Risks and open questions

| risk | what would show it | mitigation |
|---|---|---|
| `trace-pc` without guards makes edge hashing costly or collision-prone | edge counts saturate early on large targets | 64 KiB map with hashing measured first; GCC's `trace-pc-guard` support **[to measure: not tested]** |
| Inlining hides coverage in units built with `-O2` | fewer edges than basic blocks | a fuzzing build profile at `-O1 -fno-inline` [cost to measure] |
| An aborted task leaves a protected object locked | the engine hangs after a timeout | fork mode for such targets; restart from disk state |
| Differential oracles report spurious differences (formatting, a reference bug) | the triage load grows | per-field normalisers; a known-difference file, reviewed |
| Scope creep | the engine stops fitting one review sitting | scope held by §3's non-goals |
| Name collision or a trademark | found before publication | check the name in M7 |

Open questions for the maintainers:

1. **The licence.** Apache-2.0 or MIT keep it open to companies and
   individuals alike; a copyleft licence would limit proprietary users.
2. **Hosting and CI capacity**: an hour a night across all of a user's
   targets is an assumption to measure.

---

## 14. Repository layout

```
harrow/
  alire.toml
  harrow_runtime.gpr          -- never instrumented (control in CI)
  src/
    harrow.ads                -- Byte_Array, common types
    harrow-coverage.ad[sb]    -- trace_pc callbacks, edge map
    harrow-compare.ad[sb]     -- trace_cmp callbacks, operand log
    harrow-engine.ad[sb]
    harrow-corpus.ad[sb]
    harrow-schedule.ad[sb]
    harrow-mutate*.ad[sb]
    harrow-oracle*.ad[sb]
    harrow-minimise.ad[sb]
    harrow-report.ad[sb]
    harrow-regression.ad[sb]
    harrow-harness.ad[sb]     -- the generic a target instantiates
    harrow_main.adb           -- `harrow` command line
  tools/
    seed-pcap                 -- pcap/pcapng -> one input per frame
    ref/                      -- reference oracle adapters (tshark, rpmdev-vercmp, dpkg)
  bench/
    planted/                  -- §12's planted defects and their twins
  examples/
    tlv/ base64/ version/     -- small targets, each with a README
  docs/
    design.md  oracles.md  writing-a-target.md  ci.md  measurements.md
  tests/
```

---

## 15. Glossary

- **Edge**: a pair of consecutive basic blocks. Edge coverage is what
  "new coverage" means.
- **Oracle**: what decides that an output is wrong.
- **Differential**: two implementations given the same input; a
  disagreement is a finding.
- **Round-trip**: write what was read, read it again, compare.
- **Non-vacuity**: the run must contain accepted inputs; a target that
  refuses everything is reported, not praised.
- **Seed**: an input given at the start; **corpus**: the inputs kept
  because they reached new coverage.
- **Minimise**: shrink a failing input while it fails the same way.

---

## Appendix A — the measurements behind §4

21 October 2026, Linux x86-64, in a scratch directory:

```bash
G=x86_64-linux-gnu-gcc-13
$G -c -fsanitize-coverage=trace-pc t.adb      # ok; nm: U __sanitizer_cov_trace_pc
$G -c -fsanitize-coverage=trace-cmp t.adb     # ok; cmp4, cmp8, const_cmp1, const_cmp4
$G -c -fsanitize=address t.adb                # ok; __asan symbols
$G -c -fsanitize=fuzzer t.adb                 # error: unrecognized argument
# callbacks in Ada: cov.ads exports __sanitizer_cov_trace_pc (Convention C),
# compiled WITHOUT the flag; t.adb compiled WITH it; main prints the count:
#   hits 17
gnatmake -f t.adb -cargs -fsanitize=address -largs -fsanitize=address && ./t   # runs
alr search --crates | grep -i fuzz            # nothing
```

`t.adb` is a 15-line procedure with a three-way `and then` on the input
bytes. The unit and the callback package are 20 lines together. They are
to be kept in `bench/` as the M1 smoke test.
