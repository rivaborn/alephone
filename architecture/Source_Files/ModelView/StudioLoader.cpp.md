# Source_Files/ModelView/StudioLoader.cpp

## File Purpose
Loads 3D Studio Max (.3ds) model files for the Aleph One game engine. Parses the hierarchical chunk-based file format, extracts vertex positions, texture coordinates, and polygon indices, and optionally converts 3D coordinates from right-handed to left-handed orientation.

## Core Responsibilities
- Parse 3DS file format structure (chunk headers, container chunks, leaf data chunks)
- Recursively traverse the Master ΓåÆ Editor ΓåÆ Object ΓåÆ Trimesh ΓåÆ geometry data hierarchy
- Buffer and deserialize chunk data with little-endian byte order
- Extract vertex positions (3 floats per vertex) from VERTICES chunks
- Extract texture coordinates (2 floats per vertex) from TXTR_COORDS chunks
- Extract polygon face indices from FACE_DATA chunks
- Convert coordinate systems and winding order (right-handed to left-handed)
- Provide error logging and validation throughout file reading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ChunkHeaderData | struct | Represents a 3DS chunk: 2-byte ID + 4-byte size |
| Model3D | struct (external) | Container for loaded geometry: Positions, VertIndices, TxtrCoords |
| OpenedFile | class (external) | File I/O abstraction with Read/SetPosition/GetPosition |
| FileSpecifier | class (external) | File path/handle wrapper for platform-independent file access |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Path | const char* | static | Points to current model file path for error messages |
| ChunkBuffer | vector\<uint8\> | static | Reusable buffer for reading chunk payloads |
| ModelPtr | Model3D* | static | Points to the Model3D being populated during load |

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main entry point; opens a 3DS file and populates a Model3D structure with geometry data.
- **Inputs:** FileSpecifier (file path), Model3D reference (output container)
- **Outputs/Return:** true if successful, false on any read/format error
- **Side effects:** Opens file, populates ModelΓåÆPositions, ModelΓåÆVertIndices, ModelΓåÆTxtrCoords; sets global Path and ModelPtr
- **Calls:** Spec.Open(), ReadChunkHeader(), ReadContainer(ReadMaster), logError(), logNote()
- **Notes:** Validates file has MASTER chunk ID; ensures both vertices and faces are present before returning success.

### LoadModel_Studio_RightHand
- **Signature:** `bool LoadModel_Studio_RightHand(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Loads a 3DS model and converts it from right-handed (Z-up, Y-back) to left-handed (Z-up, X-forward) orientation.
- **Inputs:** FileSpecifier, Model3D reference
- **Outputs/Return:** true if load and conversion succeeded
- **Side effects:** Modifies ModelΓåÆPositions (swaps/negates X,Y); reverses ModelΓåÆVertIndices winding order; flips ModelΓåÆTxtrCoords Y
- **Calls:** LoadModel_Studio(), logTrace()
- **Notes:** Applies to Wings 3D and Blender-generated 3DS models; preserves Z-up orientation; restores clockwise face winding.

### ReadChunkHeader
- **Signature:** `bool ReadChunkHeader(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Reads a 6-byte chunk header (ID + Size).
- **Inputs:** OpenedFile, reference to ChunkHeaderData struct to fill
- **Outputs/Return:** true if read succeeded
- **Side effects:** Advances file position by 6 bytes; logs error on I/O failure
- **Calls:** OFile.Read(), StreamToValue(), logError()

### LoadChunk
- **Signature:** `bool LoadChunk(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Reads chunk payload into global ChunkBuffer for later processing.
- **Inputs:** OpenedFile, ChunkHeaderData (already read)
- **Outputs/Return:** true if read succeeded
- **Side effects:** Resizes ChunkBuffer, populates it from file; advances file position
- **Calls:** SetChunkBufferSize(), OFile.Read(), logTrace(), logError()

### ReadContainer
- **Signature:** `bool ReadContainer(OpenedFile& OFile, ChunkHeaderData& ChunkHeader, bool (*ContainerCallback)(OpenedFile&,int32))`
- **Purpose:** Generic dispatcher for container chunks; calls a callback to read sub-chunks until ParentChunkEnd is reached.
- **Inputs:** OpenedFile, ChunkHeaderData, function pointer to handle child chunks
- **Outputs/Return:** true if callback processed chunk range successfully
- **Side effects:** Computes ParentChunkEnd; delegates to callback
- **Calls:** ContainerCallback()
- **Notes:** Callback signature receives OpenedFile and boundary position; responsible for iteration logic.

### ReadMaster, ReadEditor, ReadObject, ReadTrimesh, ReadFaceData
- **Purpose:** Hierarchical chunk readers; each loops through its container's children until ParentChunkEnd, dispatching to next level or skipping unknown chunks.
- **Side effects:** ReadTrimesh asserts ModelPtr is non-null; ReadFaceData populates ModelPtrΓåÆVertIndices directly
- **Notes:** All validate that file position does not overrun ParentChunkEnd; log errors if it does.

### LoadVertices
- **Signature:** `static void LoadVertices()`
- **Purpose:** Extracts vertex count from ChunkBuffer, deserializes 3D positions into Model3D.
- **Inputs:** ChunkBuffer (global), ModelPtr (global)
- **Outputs/Return:** Populates ModelPtrΓåÆPositions
- **Side effects:** Resizes ModelPtrΓåÆPositions to 3*NumVertices floats
- **Calls:** LoadFloats()

### LoadTextureCoordinates
- **Signature:** `static void LoadTextureCoordinates()`
- **Purpose:** Extracts texture coordinate count, deserializes (U,V) pairs into Model3D.
- **Inputs:** ChunkBuffer, ModelPtr
- **Outputs/Return:** Populates ModelPtrΓåÆTxtrCoords
- **Side effects:** Resizes ModelPtrΓåÆTxtrCoords to 2*NumCoords floats
- **Calls:** LoadFloats()

### LoadFloats
- **Signature:** `void LoadFloats(int NVals, uint8 *Stream, GLfloat *Floats)`
- **Purpose:** Deserializes 4-byte little-endian IEEE 754 floats from a byte stream into GLfloat array.
- **Inputs:** Count of floats, source byte stream, destination GLfloat array
- **Outputs/Return:** Populates Floats array in-place
- **Side effects:** Advances Stream pointer; copies bytes directly (assumes GLfloat is 4 bytes)
- **Calls:** StreamToValue() (reads uint32), memcpy-like byte copy
- **Notes:** Asserts sizeof(GLfloat) == 4; portable across platforms with IEEE 754 floats.

## Control Flow Notes
**Initialization & Load:**
1. `LoadModel_Studio` opens file and reads MASTER chunk header.
2. `ReadContainer(ReadMaster)` begins recursive descent through file hierarchy.

**Chunk Processing Tree:**
- MASTER (file root) ΓåÆ calls ReadMaster
  - EDITOR ΓåÆ calls ReadEditor (skips non-EDITOR chunks)
    - OBJECT ΓåÆ calls ReadObject (skips non-OBJECT chunks)
      - TRIMESH ΓåÆ calls ReadTrimesh (processes geometry)
        - VERTICES ΓåÆ `LoadChunk()` + `LoadVertices()`
        - TXTR_COORDS ΓåÆ `LoadChunk()` + `LoadTextureCoordinates()`
        - FACE_DATA ΓåÆ calls ReadFaceData (deserializes inline)

**Coordinate Conversion (optional):**
- `LoadModel_Studio_RightHand` post-processes Model3D after load:
  - Swaps and negates X,Y (preserves Z-up orientation)
  - Reverses polygon winding order (clockwise)
  - Flips texture Y coordinates

## External Dependencies
- **Logging.h:** logError, logTrace, logNote macros
- **Packing.h:** StreamToValue, StreamToList template functions for little-endian binary deserialization
- **StudioLoader.h:** Public function declarations
- **Model3D.h:** Model3D struct with Positions, VertIndices, TxtrCoords vectors and accessor methods (PosBase, VIBase, TCBase)
- **FileHandler.h:** OpenedFile class (Read, SetPosition, GetPosition); FileSpecifier class (Open, GetPath)
- **cseries.h:** Platform types (uint8, uint16, uint32, int32, GLfloat); vector, string, assertions
