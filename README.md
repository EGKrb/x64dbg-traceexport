# x64dbg-traceexport

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![For x64dbg](https://img.shields.io/badge/for-x64dbg-444.svg)](https://github.com/x64dbg/x64dbg) [![Upstream discussion](https://img.shields.io/badge/upstream-%234006-success.svg)](https://github.com/x64dbg/x64dbg/issues/4006)

**`TraceExport`** converts an [x64dbg](https://github.com/x64dbg/x64dbg) execution trace (`.trace64` / `.trace32`) into a readable, line-oriented text file: **JSON**, **CSV**, or the text trace format of the **[Tenet](https://github.com/gaasedelen/tenet)** and **[Frinet](https://github.com/synacktiv/frinet)** trace explorers. The point is not to analyze traces inside the debugger, but to hand a recorded trace to external tooling without anyone writing a custom `.trace64` parser.

> **Scope.** This repository contains **only** the `TraceExport` feature. It is deliberately carved out of a larger personal toolkit so it can be reviewed, and ideally upstreamed, on its own. Nothing else from that toolkit (Python bridge, kernel/Hyper-V tooling, bundled plugins) is here.

This work is being discussed for upstream inclusion in x64dbg itself: **[x64dbg/x64dbg#4006](https://github.com/x64dbg/x64dbg/issues/4006)**.

---

## The command

```
TraceExport <trace_file>, <output_file>[, json|csv|tenet|frinet[, slide]]
```

- `trace_file` — the trace to read (`.trace64` with x64dbg, `.trace32` with x32dbg).
- `output_file` — created/overwritten.
- `format` — `json`, `csv`, `tenet` or `frinet`. Omitted: `csv` if the name ends in `.csv`, `frinet` if it ends in `.tenet` (the Frinet tracer's extension), otherwise `json`.
- `slide` — **`frinet` only**: the ASLR slide to write (the runtime base of the traced module, e.g. `mod.main()`). Omitted: the debugged module's base is used when the trace belongs to the live debuggee; otherwise no slide is written.

`$result` is set to the number of exported instructions. The command **does not need an active debuggee**, so it converts old traces and runs inside `headless.exe`.

```
StartTraceRecording "C:\traces\target.trace64"
TraceIntoConditional 0, .1000
StopTraceRecording
TraceExport "C:\traces\target.trace64", "C:\traces\target.json"
TraceExport "C:\traces\target.trace64", "C:\traces\target.tenet", frinet, mod.main()
```

Full command reference: [docs/TraceExport.command.md](docs/TraceExport.command.md).

### JSON — one instruction per line (greppable)

```json
{"index": 1, "thread": 21712, "address": "0x7FF757A81290", "module": "target.exe",
 "bytes": "E8 4B 02 00 00", "disasm": "call 0x00007FF757A814E0",
 "regs": {"rax": "0x7FF757A8128C", "rip": "0x7FF757A81290", "rflags": "0x202"},
 "mem": [{"address": "0x625451F978", "old": "0x0", "new": "0x7FF757A81295"}]}
```

Addresses and values are **hex strings** (a 64-bit value does not fit a standard JSON number — `int(v, 16)` in Python).

### CSV — one row per instruction

Columns `index,thread,address,module,bytes,disasm,<registers>,memory`. The `memory` column lists accesses separated by `;`: `address:value` for an unchanged read, `address:before->after` for a write. Opens directly in Excel / `pandas.read_csv`.

### Tenet / Frinet — drop straight into the trace explorers

Both emit one line per instruction, comma-separated `name=0xvalue` entries: the registers that changed since the previous line, the instruction pointer, and memory accesses as `mr=0xaddr:bytes` (read) / `mw=0xaddr:bytes` (write). Flags are omitted (these tools don't read them). The two differ in what a line describes:

- **`tenet`** — a line is the state *before* its instruction, with the memory accessed by the *previous* one. Loads directly in **Tenet** (IDA Pro, x86 and x64).
- **`frinet`** — an optional `slide=0x...` first line, then per-instruction: instruction pointer first, registers *after* the instruction and the memory it accessed. Index it with `frinet index --format tenet trace.tenet trace.db`, then open the DB in IDA. 64-bit only (the Frinet plugin reads `rip`/`pc`).

---

## Why this is worth exporting (and why it scales)

Dumping a whole trace to JSON and loading it into memory doesn't scale past small traces — a fair objection. The value of a **line-oriented text format** is as an *interchange* format for tools that **index** traces instead of loading them:

- **[Tenet](https://github.com/gaasedelen/tenet)** — the trace explorer for IDA (timeline, register/memory views, "go to the first/prev/next execution of this address").
- **[Synacktiv frinet](https://github.com/synacktiv/frinet)** — an enhanced Tenet whose Rust indexer ingests the Tenet text trace format and scales to **~2 billion instructions (~100 GB)**, with multi-threaded regex memory search.

x64dbg's trace already carries exactly what those tools need — per-step register deltas and memory reads/writes — so the `tenet` / `frinet` emitters map it over directly. JSON/CSV stay handy for quick `grep`/`jq`/pandas on small traces; the Tenet/Frinet output is what turns a large x64dbg trace into something a mature frontend can drive.

---

## Status & design

**Current state.** The exporter in [`src/`](src/) is the reference implementation and already emits all four formats (json / csv / tenet / frinet). It was written inside x64dbg's `dbg` module, so it currently links against x64dbg internals — the trace reader (`TraceRecord` / `TraceFileReader`), the Zydis disassembler wrapper (`zydis_wrapper.h`), and module/hash helpers. It streams the file (constant memory regardless of trace size), reconstructs register state from the delta-encoded blocks, reads memory accesses, disassembles with Zydis, honors the user-block convention (type ≥ `0x80` skipped), refuses to overwrite its input, and tolerates a trace still being recorded (the incomplete last block is skipped with a warning).

**Where it's going.** Per the upstream discussion ([#4006](https://github.com/x64dbg/x64dbg/issues/4006)), the clean split is:

1. a small **trace-iteration API/header in x64dbg core** (open a trace → iterate instructions with reconstructed register state + memory accesses), and
2. **`TraceExport` as a plugin** on top of it, emitting JSON / CSV / Tenet / Frinet.

The streaming reader here already performs exactly that iteration, so turning it into a public iterator is mostly extraction. See [DESIGN.md](DESIGN.md) for the proposed interface.

**Limits.** Exports GPRs + instruction pointer (+ flags for JSON/CSV) only; SSE/AVX/x87 are in the trace but not yet emitted. A 32-bit trace must be read with the 32-bit build and vice-versa; `frinet` is 64-bit only.

---

## Repository layout

```
src/TraceExport.cpp, .h          the exporter (reference implementation, all 4 formats)
docs/TraceExport.command.md      command reference
docs/tracefile.format.md         the .trace64/.trace32 binary format spec
DESIGN.md                        proposed core iteration API + plugin split
LICENSE                          GPL-3.0
```

The trace file format is documented in [docs/tracefile.format.md](docs/tracefile.format.md).

---

## License & credits

GPL-3.0, same as [x64dbg](https://github.com/x64dbg/x64dbg) (by mrexodia and contributors), on which this is based. The `.trace64` format and the trace reader are x64dbg's. Tenet is by [gaasedelen](https://github.com/gaasedelen/tenet); Frinet is by [Synacktiv](https://github.com/synacktiv/frinet).

Not affiliated with or endorsed by the x64dbg team.
