# Source_Files/Sound/Decoder.cpp

## File Purpose
Implements factory methods for creating audio decoders. Attempts to instantiate a libsndfile-based decoder and returns it on successful file open, or null on failure.

## Core Responsibilities
- Factory method to create StreamDecoder instances
- Factory method to create Decoder instances
- Attempt to open audio files using libsndfile
- Return decoder on success, null pointer on failure

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamDecoder::Get
- Signature: `static std::unique_ptr<StreamDecoder> StreamDecoder::Get(FileSpecifier& File)`
- Purpose: Factory method to create and initialize a StreamDecoder for a given audio file.
- Inputs: `File` (FileSpecifier reference) ΓÇö the audio file to decode
- Outputs/Return: `std::unique_ptr<StreamDecoder>` ΓÇö a managed decoder instance on success, or null (0) if the file cannot be opened
- Side effects: Creates and attempts to open the audio file; allocates resources via libsndfile
- Calls: `std::make_unique<SndfileDecoder>()`, `SndfileDecoder::Open(FileSpecifier&)`
- Notes: Returns ownership via unique_ptr; caller receives automatic resource cleanup on move or destruction

### Decoder::Get
- Signature: `static Decoder* Decoder::Get(FileSpecifier& File)`
- Purpose: Factory method to create and initialize a Decoder instance for a given audio file.
- Inputs: `File` (FileSpecifier reference) ΓÇö the audio file to decode
- Outputs/Return: Raw `Decoder*` pointer on success, or null (0) if the file cannot be opened
- Side effects: Creates and attempts to open the audio file; allocates resources via libsndfile; caller owns pointer
- Calls: `std::make_unique<SndfileDecoder>()`, `SndfileDecoder::Open(FileSpecifier&)`, `unique_ptr::release()`
- Notes: Uses `release()` to transfer ownership from unique_ptr to raw pointer; caller responsible for deletion; different ownership model than StreamDecoder::Get

## Control Flow Notes
This file provides entry points for audio file loading during game initialization or runtime sound loading. Both factory methods follow a "try-and-return" pattern: instantiate SndfileDecoder, attempt to open the file, and return the decoder (on success) or null (on failure). Callers use returned decoders to stream audio data during playback.

## External Dependencies
- `Decoder.h` ΓÇö base class definitions (StreamDecoder, Decoder)
- `SndfileDecoder.h` ΓÇö concrete libsndfile-based decoder implementation
- `<memory>` ΓÇö std::unique_ptr, std::make_unique
- `FileSpecifier` ΓÇö defined elsewhere (likely FileHandler.h), represents a file path/reference
