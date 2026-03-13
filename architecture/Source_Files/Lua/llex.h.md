# Source_Files/Lua/llex.h

## File Purpose
Defines the lexical analyzer interface for Lua. Declares token types (reserved words and operators), the LexState structure that maintains lexer state during tokenization, and the public API for lexical analysis and token manipulation.

## Core Responsibilities
- Define token type constants for all Lua reserved words and operators (RESERVED enum)
- Maintain lexer state including current/lookahead tokens, input position, and line tracking (LexState)
- Initialize and configure lexer state for input streams
- Advance token stream and provide lookahead capability
- Intern strings to ensure single-copy representation
- Report syntax errors with source location information
- Convert token codes to human-readable strings for diagnostics

## Key Types / Data Structures
| Name | Kind (struct/enum/typedef) | Purpose |
|------|---------------------------|---------|
| RESERVED | enum | Token type constants (keywords like TK_AND, TK_IF; operators like TK_EQ, TK_CONCAT; special tokens TK_NUMBER, TK_NAME, TK_STRING, TK_EOS) |
| SemInfo | union | Semantic value of a token: lua_Number (r) or TString* (ts) |
| Token | struct | A lexical token: type (int token) + semantic info (SemInfo seminfo) |
| LexState | struct | Lexer state machine: current char, line counters, current/lookahead tokens, input stream (ZIO), token buffer (Mbuffer), source name, environment var name, locale decimal point |

## Global / File-Static State
None.

## Key Functions / Methods

### luaX_init
- Signature: `void luaX_init (lua_State *L);`
- Purpose: Initialize the lexer subsystem (one-time setup)
- Inputs: L (Lua state)
- Outputs/Return: void
- Side effects: Sets up lexer globals in the Lua state
- Calls: Not inferable from this file
- Notes: Called during Lua state initialization

### luaX_setinput
- Signature: `void luaX_setinput (lua_State *L, LexState *ls, ZIO *z, TString *source, int firstchar);`
- Purpose: Prepare a lexer state for reading from an input stream
- Inputs: L (Lua state), ls (lexer state), z (input stream), source (source file name for diagnostics), firstchar (initial character code)
- Outputs/Return: void
- Side effects: Initializes LexState fields (current, linenumber, t, lookahead, z, buff, source, decpoint)
- Calls: Not inferable from this file
- Notes: Called before tokenizing each source chunk

### luaX_newstring
- Signature: `TString *luaX_newstring (LexState *ls, const char *str, size_t l);`
- Purpose: Intern a string; returns a canonical TString* for deduplication
- Inputs: ls (lexer state), str (string bytes), l (length in bytes)
- Outputs/Return: TString* (pointer to interned string)
- Side effects: May allocate memory; updates string pool in Lua state
- Calls: Not inferable from this file
- Notes: Ensures identical strings share single memory location

### luaX_next
- Signature: `void luaX_next (LexState *ls);`
- Purpose: Advance to the next token
- Inputs: ls (lexer state)
- Outputs/Return: void
- Side effects: Moves ls->lookahead to ls->t, reads next lookahead token, updates ls->lastline
- Calls: Not inferable from this file
- Notes: Typically called by parser to consume a token

### luaX_lookahead
- Signature: `int luaX_lookahead (LexState *ls);`
- Purpose: Peek at the next token type without consuming it
- Inputs: ls (lexer state)
- Outputs/Return: int (RESERVED token code from ls->lookahead)
- Side effects: None
- Calls: Not inferable from this file
- Notes: Parser uses this to guide lookahead-dependent decisions

### luaX_syntaxerror
- Signature: `l_noret luaX_syntaxerror (LexState *ls, const char *s);`
- Purpose: Report a syntax error and abort compilation
- Inputs: ls (lexer state for source location), s (error message)
- Outputs/Return: l_noret (does not return; longjmp or exception)
- Side effects: Unwinds execution, halts parsing
- Calls: Not inferable from this file
- Notes: Non-returning function; typically called from parser

### luaX_token2str
- Signature: `const char *luaX_token2str (LexState *ls, int token);`
- Purpose: Convert a token code to a human-readable string
- Inputs: ls (lexer state), token (RESERVED token code)
- Outputs/Return: const char* (token name like "and", "==", "NAME")
- Side effects: None
- Calls: Not inferable from this file
- Notes: Used in error messages and debugging output

## Control Flow Notes
This header defines the lexer stage of compilation. Usage pattern:
1. **Initialization:** `luaX_init()` (once per Lua state)
2. **Per-source:** `luaX_setinput()` prepares LexState for a new input stream
3. **Token loop:** Parser repeatedly calls `luaX_lookahead()` to inspect, then `luaX_next()` to consume
4. **Error handling:** `luaX_syntaxerror()` aborts if malformed input detected
5. **String interning:** `luaX_newstring()` called when identifiers/string literals are encountered

Lexer is the first stage of the compile pipeline: Source ΓåÆ Lexer (llex.h/llex.c) ΓåÆ Tokens ΓåÆ Parser ΓåÆ AST ΓåÆ Codegen.

## External Dependencies
- **`lobject.h`**: TString (interned strings), lua_Number (numeric values), type system macros
- **`lzio.h`**: ZIO (buffered input stream), Mbuffer (growable token buffer)
- **`lua.h`** (implicit): lua_State, LUAI_FUNC (visibility macro), lua_Reader callback type
