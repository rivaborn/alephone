# Source_Files/ModelView/WavefrontLoader.cpp

## File Purpose
Loads and parses Wavefront OBJ 3D model files for the Aleph One engine. Converts OBJ geometry (vertices, texture coordinates, normals) into the engine's internal `Model3D` representation. Includes optional coordinate-system conversion from OBJ's right-handed to Aleph One's left-handed convention.

## Core Responsibilities
- Parse Wavefront OBJ file format, handling line continuations (backslash escapes)
- Extract and buffer vertex positions, texture coordinates, and surface normals
- Parse face definitions and convert vertex index sets (with per-component indices: position/texture/normal)
- Deduplicate vertex sets to build a unique vertex list and index remapping
- Tessellate arbitrary polygons into triangles via fan decomposition
- Validate index ranges and presence of required data (positions are mandatory)
- Convert geometry and normals from right-handed (OBJ convention) to left-handed (engine convention)
- Provide detailed error, warning, and trace logging throughout parsing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `IndexedVertListCompare` | struct | STL comparator for sorting vertex index sets by position, then texture, then normal |
| `Present_Position`, `Present_TxtrCoord`, `Present_Normal` | enum | Bitflags indicating which vertex components are present in a given vertex set |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Path` | const char* | static | Pointer to model file path; used in error messages |
| `InputLine` | vector\<char\> | static | Reusable buffer for reading and parsing file lines with continuation support |

## Key Functions / Methods

### LoadModel_Wavefront
- **Signature:** `bool LoadModel_Wavefront(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main entry point; reads OBJ file, parses geometry, populates Model3D object.
- **Inputs:** `FileSpecifier` (file reference), `Model3D` (output container, cleared on entry)
- **Outputs/Return:** `true` if parsing succeeded and model has valid polygons; `false` otherwise
- **Side effects:** Populates `Model.Positions`, `Model.TxtrCoords`, `Model.Normals`, `Model.VertIndices`; logs errors/warnings; modifies static `Path` and `InputLine`
- **Calls:** `CompareToKeyword`, `GetVertIndxSet`, `sort` (STL), `Model3D::Clear`, logging functions
- **Notes:** Handles Wavefront conventions (1-based indexing, negative indices reference end of current list). Converts polygons to triangles. Validates all indices before returning.

### LoadModel_Wavefront_RightHand
- **Signature:** `bool LoadModel_Wavefront_RightHand(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Wrapper that loads OBJ via `LoadModel_Wavefront`, then transforms coordinates and texture UVs to match Aleph One's left-handed system.
- **Inputs:** `FileSpecifier`, `Model3D` (output)
- **Outputs/Return:** `true` if load and conversion succeed; `false` if load fails
- **Side effects:** Swaps/negates position and normal coordinates; reverses triangle winding order; inverts V texture coordinate
- **Calls:** `LoadModel_Wavefront`
- **Notes:** Assumes OBJ models use Blender/Wings3D orientation (Y up, front faces +Z). Reverses triangle winding because coordinate swap changes handedness.

### CompareToKeyword
- **Signature:** `char *CompareToKeyword(const char *Keyword)`
- **Purpose:** Matches a keyword at the start of the current input line, skipping leading whitespace.
- **Inputs:** Keyword string (e.g., "v", "f", "vt")
- **Outputs/Return:** Pointer to rest of line after keyword + whitespace, or `NULL` if no match
- **Calls:** `strlen`

### GetVertIndxSet
- **Signature:** `char *GetVertIndxSet(char *Buffer, short& Presence, short& PosIndx, short& TCIndx, short& NormIndx)`
- **Purpose:** Parse a single vertex reference from a face definition (e.g., "123/456/789").
- **Inputs:** Buffer (rest of face line), references for output values
- **Outputs/Return:** Pointer past the vertex set; fills `Presence` flags and three index values
- **Calls:** `GetVertIndx` (three times)

### GetVertIndx
- **Signature:** `char *GetVertIndx(char *Buffer, bool& WasFound, short& Val, bool& HitEnd)`
- **Purpose:** Parse a single numeric index, handling "/" and whitespace delimiters.
- **Inputs:** Buffer (portion of vertex reference), output references
- **Outputs/Return:** Pointer past index; sets `WasFound`, `Val`, `HitEnd` flags
- **Calls:** `sscanf`
- **Notes:** Reads up to "/" or whitespace; returns `HitEnd=true` if end of vertex set reached.

## Control Flow Notes
**Initialization:** Clear model; open file; allocate intermediate lists (Positions, TxtrCoords, Normals, PolygonSizes, VertIndxSets).

**Reading Phase:** Line-by-line parsing loop with continuation support (backslash joins lines). Each non-comment, non-blank line is parsed for keywords.

**Parsing Phase:** Keywords route to handlersΓÇö`v`/`vt`/`vn` append to geometry buffers; `f` parses indices and builds intermediate polygon data.

**Post-Processing Phase:** Deduplicate vertex sets via STL sort; map old indices to unique indices; expand unique vertices into Model3D containers; tessellate polygons into triangles via fan decomposition.

**Validation & Return:** Check that all indices are in range, positions are present, and at least one valid polygon exists.

## External Dependencies
- **Framework:** `Model3D`, `FileSpecifier` (engine types)
- **Logging:** `logNote`, `logError`, `logWarning`, `logTrace` macros
- **STL:** `vector`, `sort`, `algorithm`
- **Standard C/C++:** `ctype.h`, `stdlib.h`, `string.h`, `cstring`, `<algorithm>`
