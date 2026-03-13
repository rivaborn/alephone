# Source_Files/Lua/lopcodes.h

## File Purpose
Defines the instruction format and complete opcode set for the Lua virtual machine. Provides macros for encoding, decoding, and manipulating 32-bit VM instructions with opcode and argument fields. Specifies all 38 opcodes and their argument modes.

## Core Responsibilities
- Define instruction bit layout (6-bit opcode + A/B/C/Bx/sBx/Ax argument fields)
- Encode/decode instruction fields via bit manipulation macros
- Define all Lua VM opcodes (arithmetic, table, control flow, function calls, etc.)
- Specify argument constraints (size limits, signed vs. unsigned, register vs. constant encoding)
- Provide instruction construction and field extraction helpers
- Declare opcode metadata tables and name strings (extern)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OpMode` | enum | Instruction format selector: iABC (three 8-9 bit args), iABx (one 8-bit + one 18-bit), iAsBx (signed variant), iAx (one 26-bit arg) |
| `OpArgMask` | enum | Argument usage mode: OpArgN (unused), OpArgU (used), OpArgR (register/jump), OpArgK (constant/register) |
| `OpCode` | enum | All 38 opcodes: OP_MOVE, OP_LOADK, OP_CALL, OP_JMP, OP_RETURN, OP_FORLOOP, OP_CLOSURE, etc. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luaP_opmodes` | `const lu_byte[]` | extern | Metadata byte per opcode: bits 0-1 = OpMode, bits 2-5 = arg masks, bit 6 = sets A, bit 7 = test op |
| `luaP_opnames` | `const char*[]` | extern | Null-terminated string names for each opcode (for debugging/disassembly) |

## Key Functions / Methods
None. File is purely macro definitions and type/constant declarations.

## Macro-based API

### Instruction Field Extraction / Mutation
- `GET_OPCODE(i)`, `SET_OPCODE(i, o)` ΓÇö extract/set 6-bit opcode
- `GETARG_A/B/C/Bx/sBx/Ax(i)`, `SETARG_*` ΓÇö extract/set individual fields
- `getarg(i, pos, size)`, `setarg(i, v, pos, size)` ΓÇö generic field helpers

### Instruction Construction
- `CREATE_ABC(o, a, b, c)` ΓÇö pack opcode + three fields into instruction
- `CREATE_ABx(o, a, bc)` ΓÇö pack with 18-bit combined B+C
- `CREATE_Ax(o, a)` ΓÇö pack with 26-bit field

### Constant/Register Encoding (RK Values)
- `BITRK` ΓÇö bit flag (1 << 8) marking constant vs. register
- `ISK(x)` ΓÇö test if x is constant (bit 8 set)
- `INDEXK(r)` ΓÇö extract constant table index from RK
- `RKASK(x)` ΓÇö encode constant index as RK value

### Bit Manipulation
- `MASK1(n, p)` ΓÇö create mask with n ones at position p
- `MASK0(n, p)` ΓÇö create mask with n zeros at position p
- `getOpMode(m)`, `getBMode(m)`, `getCMode(m)` ΓÇö query opcode metadata
- `testAMode(m)`, `testTMode(m)` ΓÇö test if opcode sets A / is test-class

## Control Flow Notes
This header defines the instruction encoding contract used throughout the Lua VM:
- **Compilation phase**: compiler emits instructions using CREATE_* and SETARG_* macros
- **VM execution**: `lvm.c` decodes instructions using GETARG_* and field tests to dispatch operations
- **Function prototypes**: stored as bytecode sequences; each instruction specifies one operation + operands

The instruction format is fixed (32-bit unsigned) and architecture-dependent aspects (byte order, integer width) are abstracted here.

## External Dependencies
- **Includes**: `llimits.h` (for `Instruction` type = `lu_int32`, `lu_byte`, size/alignment definitions)
- **Defined elsewhere**: 
  - `luaP_opmodes[]` ΓÇö typically in `lopcodes.c`
  - `luaP_opnames[]` ΓÇö typically in `lopcodes.c`
  - `LUAI_BITSINT`, `LUAI_DDEC`, `MAX_INT` ΓÇö from `llimits.h` / config

## Notes
- 38 opcodes cover: data movement (MOVE, LOADK), arithmetic (ADD, SUB, MUL, DIV, MOW, POW), logic (NOT, TEST, EQ, LT, LE), tables (GETTABLE, SETTABLE, NEWTABLE), functions (CALL, RETURN, CLOSURE), control flow (JMP, FORLOOP, TFORLOOP), and variadic args (VARARG).
- Signed fields (sBx, signed jumps) use excess-K encoding: value = unsigned_bits ΓêÆ MAXARG_sBx.
- RK encoding packs register indices and constant indices into 9-bit values; bit 8 distinguishes them.
- `LFIELDS_PER_FLUSH = 50` ΓÇö optimization hint for table constructor bytecode generation.
