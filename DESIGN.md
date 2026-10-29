# Design — trace iteration API + TraceExport plugin

This is a proposal, aligned with the upstream discussion in
[x64dbg/x64dbg#4006](https://github.com/x64dbg/x64dbg/issues/4006). The idea is to split the
current `TraceExport` into two pieces:

1. a **trace-iteration API** that lives in x64dbg (core or SDK), and
2. **`TraceExport` as a plugin** built on that API.

The reader that already exists in [`src/TraceExport.cpp`](src/TraceExport.cpp) does the whole
iteration today; the work is extracting it behind a stable interface so any plugin can reuse it.

## What one iteration step yields

Everything already present in a type-`0` trace block (see
[docs/tracefile.format.md](docs/tracefile.format.md)), with register state **reconstructed** from
the delta encoding so callers don't have to track it themselves:

- `index` — 0-based instruction number in the trace
- `threadId`
- `opcode[]` + `opcodeSize` (raw bytes; disassembly is the caller's choice)
- the general-purpose register file *before* the instruction (reconstructed), plus IP and flags
- memory accesses: for each, `address`, `oldValue`, `newValue`, and a read/write flag
- optionally, the recorded trace `hash` (so a caller can decide whether module resolution against
  the live debuggee is valid)

The register layout mirrors the words of `REGDUMP` that the format stores, so no information is
lost relative to the file.

## Two possible shapes

mrexodia's call on which fits the codebase better.

### A. Pull-style cursor (what the current reader is)

```c
typedef struct TraceReader_ TraceReader;

typedef struct {
    duint   index;
    DWORD   threadId;
    uint8_t opcode[16];
    uint8_t opcodeSize;
    duint   regs[TRACE_REG_WORDS];     // reconstructed, pre-instruction
    int     memoryCount;
    struct { duint address, oldData, newData; bool write; } memory[TRACE_MAX_MEM_OPS];
} TraceInstruction;

// Open/close. arch is validated against the file header.
TraceReader* TraceReaderOpen(const char* fileName, char* error, size_t errorSize);
void         TraceReaderClose(TraceReader* r);

// Metadata from the JSON header.
bool  TraceReaderArch(const TraceReader* r, int* bits);      // 32 / 64
duint TraceReaderHash(const TraceReader* r);

// Advance. Returns false at end of file; *truncated is set if the last block was incomplete.
bool TraceReaderNext(TraceReader* r, TraceInstruction* out, bool* truncated);
```

Consumer:

```c
TraceReader* r = TraceReaderOpen(path, err, sizeof(err));
TraceInstruction insn; bool truncated = false;
while(TraceReaderNext(r, &insn, &truncated)) { /* emit JSON/CSV/Tenet */ }
TraceReaderClose(r);
```

### B. Visitor / callback

```c
typedef bool (*TraceVisitor)(const TraceInstruction* insn, void* userdata); // return false to stop
bool TraceReaderForEach(const char* fileName, TraceVisitor cb, void* userdata,
                        char* error, size_t errorSize);
```

The cursor (A) composes better with exporters that interleave I/O; the visitor (B) is a smaller
surface. Either can be wrapped over the same internal loop.

## What `TraceExport` becomes

A plugin that registers the `TraceExport` command, calls the iteration API, and formats each step.
Formats:

- **json** — one object per line (greppable), hex strings for 64-bit-safe values;
- **csv** — one row per instruction, memory accesses packed into one column;
- **tenet** — the [Tenet](https://github.com/gaasedelen/tenet) text trace format, so the output
  drops straight into Tenet / [frinet](https://github.com/synacktiv/frinet). Register and memory
  deltas map directly from each step.

Disassembly: the plugin can use its own Zydis (the format stores raw opcode bytes), or the API can
expose the already-initialized wrapper — open question for review.

## Open questions for review

- Cursor (A) vs. visitor (B)?
- Should the API live in `x64dbg.dll` (exported) or in the plugin SDK as source?
- Expose reconstructed registers (convenient) or just the raw deltas (minimal)?
- Resolve the module for an address inside the API, or leave that to the caller (it's only valid
  when the trace matches the live debuggee)?
- Is a Tenet emitter in-scope for the core plugin, or better as a separate plugin?
