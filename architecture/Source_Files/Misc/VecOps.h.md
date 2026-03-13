# Source_Files/Misc/VecOps.h

## File Purpose
A C++ template header providing foundational 3D vector operations (copy, add, subtract, scale, dot product, cross product). Designed for type-agnostic linear algebra enabling operations across different numeric types (float, fixed-point, etc.).

## Core Responsibilities
- Vector element-wise operations: copy, addition, subtraction
- In-place vector mutations: +=, -=
- Scalar multiplication (both creating new vector and mutating)
- Dot product (scalar projection)
- Cross product (3D vector product)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### VecCopy
- Signature: `template <class T0, class T1> inline void VecCopy(const T0* V0, T1* V1)`
- Purpose: Copy vector with implicit type conversion
- Inputs: V0 (source vector, array-like), T1 type target
- Outputs/Return: V1 (destination vector)
- Side effects: None (pure copy)
- Calls: None (implicit conversion operators)
- Notes: Assumes 3-element arrays; no bounds checking

### VecAdd, VecSub
- Signature: `template void VecAdd(const T0* V0, const T1* V1, T2* V2)` / VecSub (subtraction)
- Purpose: Element-wise addition/subtraction with output type conversion
- Inputs: V0, V1 (source vectors), T2 target type
- Outputs/Return: V2 (result vector)
- Side effects: None
- Calls: None
- Notes: Operates on 3 components; supports mixed numeric types

### VecAddTo, VecSubFrom
- Signature: `template void VecAddTo(T0* V0, const T1* V1)` / VecSubFrom (subtraction)
- Purpose: In-place vector addition/subtraction (+=, -=)
- Inputs: V0 (mutable vector), V1 (operand vector)
- Outputs/Return: None (mutation only)
- Side effects: Modifies V0 in-place
- Calls: None
- Notes: Simpler than copy-to-new-vector; used for accumulation

### VecScalarMult, VecScalarMultTo
- Signature: `template void VecScalarMult(const T0* V0, const TS& S, T1* V1)` / In-place variant
- Purpose: Scale vector by scalar (├ùS, ├ù=S)
- Inputs: V0 (vector), S (scalar multiplier)
- Outputs/Return: V1 (scaled result) or in-place mutation
- Side effects: None (or in-place mutation for MultTo)
- Calls: None
- Notes: Scalar type (TS) can differ from vector component type

### ScalarProd
- Signature: `template <class T> inline T ScalarProd(const T* V0, const T* V1)`
- Purpose: Compute dot product (V0 ┬╖ V1)
- Inputs: V0, V1 (vectors, same numeric type T)
- Outputs/Return: T (scalar result: sum of element-wise products)
- Side effects: None
- Calls: None
- Notes: Returns single numeric value; type must support multiplication/addition

### VectorProd
- Signature: `template void VectorProd(const T0* V0, const T1* V1, T2* V2)`
- Purpose: Compute cross product V2 = V0 ├ù V1
- Inputs: V0, V1 (3D vectors)
- Outputs/Return: V2 (perpendicular 3D vector)
- Side effects: None
- Calls: None
- Notes: Right-hand rule; result magnitude = |V0||V1|sin(╬╕)

## Control Flow Notes
No control flow; pure computational utility. Functions are called by geometry/physics modules handling world positions, velocities, rotations.

## External Dependencies
- Standard C++ operators (implicit type conversion, arithmetic)
- No explicit includes (assumes types passed support `+=`, `-=`, `*`, etc.)
