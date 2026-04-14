[![Basic Model Interface](https://img.shields.io/badge/CSDMS-Basic%20Model%20Interface-green.svg)](https://bmi.readthedocs.io/)
[![Build/Test](https://github.com/csdms/bmi-example-fortran/workflows/Build/Test/badge.svg)](https://github.com/csdms/bmi-example-fortran/actions?query=workflow%3ABuild%2FTest)

# bmi-example-fortran

An example of implementing the
[Fortran bindings](https://github.com/csdms/bmi-fortran)
for the CSDMS [Basic Model Interface](https://bmi.csdms.io) (BMI).

This Fork was created by Deltares to create Shared Libraries for both Windows and Linux that are compatible with Java.

## Overview

This is an example of implementing a BMI
for a simple model that  solves the diffusion equation
on a uniform rectangular plate
with Dirichlet boundary conditions.
Tests and examples of using the BMI are provided.
The model is written in Fortran 90.
The BMI is written in Fortran 2003.

## Deltares FEWS / Java interop

This fork adds a C/Java interoperability layer on top of the standard BMI
implementation, following the [NOAA-OWP NextGen `iso_c_fortran_bmi`](https://github.com/NOAA-OWP/ngen/tree/development/extern/iso_c_fortran_bmi)
pattern. This allows the model to be called from Java via
[JNA (Java Native Access)](https://github.com/java-native-access/jna)
inside [Deltares FEWS](https://www.deltares.nl/en/software-and-data/products/delft-fews).

### Architecture

```
Java (FEWS)
    │  JNA – interop/FortranModelJnaLibrary.java
    ▼
libbmi_heat.so / .dll
    ├── register_bmi.f90          model-specific factory function
    ├── iso_c_bmif_2_0.f90        generic C-interop layer (all 50+ BMI functions)
    │       uses ↓
    ├── bmif_2_0_iso.f90          BMI abstract type with ISO C integer kinds
    │       extends ↓
    ├── bmi.f90                   CSDMS BMI v2.0 abstract spec
    │
    └── bmi_heat.f90              concrete heat-model BMI implementation
            uses ↓
        heat.f90                  2D heat equation physics
```

### Opaque handle pattern

`register_bmi(void** handle)` allocates a `bmi_heat` instance, wraps it in a
thin Fortran `box` type, and returns `c_loc(box)` as an opaque `void*`. Every
other BMI function takes this handle, recovers the Fortran object with
`c_f_pointer`, and dispatches polymorphically.

`finalize(void** handle)` **both** runs the BMI finalize method **and**
deallocates the model. There is no separate `bmi_destroy()` — do not use
the handle after calling `finalize`.

**ABI note:** every function except `register_bmi` receives the handle as
`type(c_ptr), intent(in)` **without** the Fortran `VALUE` attribute.
Without `VALUE`, Fortran `bind(C)` passes by reference, so the C ABI is
`void**` throughout. In JNA this maps to `PointerByReference` — do **not**
unwrap with `.getValue()` before calling BMI methods.

### FEWS build (GitHub Actions)

| Platform | Container / runner | Compiler | Output |
|----------|--------------------|----------|--------|
| Linux | AlmaLinux 8 Docker (`docker/Dockerfile`) | Intel ifx 2025.2 | `libbmi_heat.so` (statically linked Intel runtime) |
| Windows | `windows-latest` | Intel ifx 2025.2 | `libbmi_heat.dll` + 4 Intel runtime DLLs |

### Java usage (JNA)

```java
import bmi.model.FortranModelJnaLibrary;
import bmi.model.FortranString;
import com.sun.jna.Native;
import com.sun.jna.ptr.IntByReference;
import com.sun.jna.ptr.PointerByReference;

// Load the shared library
FortranModelJnaLibrary lib =
    Native.load("bmi_heat", FortranModelJnaLibrary.class);

// Allocate the model
PointerByReference handleRef = new PointerByReference();
lib.register_bmi(handleRef);
// All methods take handleRef directly — do NOT call handleRef.getValue()

// Initialize
lib.initialize(handleRef, new FortranString("/path/to/config.cfg").toBytes());

// Find grid size for a variable
IntByReference gridRef = new IntByReference();
lib.get_var_grid(handleRef, new FortranString("plate_surface__temperature").toBytes(), gridRef);
IntByReference sizeRef = new IntByReference();
lib.get_grid_size(handleRef, gridRef, sizeRef);

// Run one step and retrieve results
lib.update(handleRef);
float[] dest = new float[sizeRef.getValue()];
lib.get_value_float(handleRef, new FortranString("plate_surface__temperature").toBytes(), dest);

// Finalize (also frees memory — do not use handleRef after this)
lib.finalize(handleRef);
```

### Interop files

| File | Purpose |
|------|---------|
| `interop/bmi.h` | C header with exact exported symbol signatures and ABI notes (DIFF 1–8) |
| `interop/FortranModelJnaLibrary.java` | JNA interface (`bmi.model` package) |
| `interop/FortranString.java` | Helper: converts Java String ↔ Fortran fixed-size 2048-byte buffer |
| `interop/bmi_from_spec.h` | Reference header derived from the abstract Fortran spec (not the actual ABI) |

See `interop/bmi.h` for important ABI differences from the abstract spec,
including known stub implementations and a bug in `get_grid_edge_nodes`.

### Known limitations

- `get_value_ptr_*` — not implemented; always returns `BMI_FAILURE`.
- `get/set_value_at_indices_*` — not implemented; always returns `BMI_FAILURE`.
- `get_grid_edge_nodes` — contains a logic bug in `iso_c_bmif_2_0.f90`
  (line 893) that may cause incorrect array sizing. See `interop/bmi.h` (BUG 1).

### Credits

- NOAA-OWP `iso_c_fortran_bmi` interop layer: Nels Frazier, NOAA OWP (Apache 2.0)
