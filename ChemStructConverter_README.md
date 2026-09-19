# ChemStructConverter 1.1.0

A Windows-friendly batch structure converter for computational chemistry files.

## Main workflow

1. Launch `ChemStructConverter.exe`.
2. Drag any mixture of supported files or folders into the input list.
3. Select one target format.
4. Click `Convert`.
5. Every input file is detected and converted independently.

Files with the same output name are never overwritten. The program automatically creates names such as `_2`, `_3`, and so on.

## Output formats

- Plain `.xyz`
- Gaussian `.gjf`
- `.cif`
- VASP POSCAR with Direct coordinates
- VASP POSCAR with Cartesian coordinates

## Computational chemistry inputs and outputs

- XYZ, including multi-frame XYZ trajectories; the last frame is used
- Gaussian input (`.gjf`, `.com`)
- Gaussian output (`.log`, `.out` when identified by content)
- CIF
- VASP POSCAR / CONTCAR
- VASP OUTCAR
- VASP XDATCAR
- VASP `vasprun.xml`
- ORCA input with inline `* xyz` or `%coords` Cartesian coordinates
- ORCA output
- NWChem input and output
- CP2K input with inline `&COORD` geometry
- CP2K output
- Quantum ESPRESSO input and output
- SIESTA FDF input with inline coordinates
- SIESTA text output using the last coordinate block
- SIESTA `.XV`

For output files and trajectory-like formats, only the last available structure is converted.

## Additional structure formats

- PDB (`.pdb`)
- PQR (`.pqr`)
- MDL MOL (`.mol`)
- Tripos MOL2 (`.mol2`)
- SDF (`.sdf`)
- XCrySDen XSF (`.xsf`)
- Gaussian CUBE (`.cube`, `.cub`)
- DFTB+ GEN (`.gen`)
- AtomEye CFG (`.cfg`)
- GROMACS GRO (`.gro`)
- GROMOS96 (`.g96`)
- ASE trajectory (`.traj`)
- SHELX RES (`.res`)
- Materials Studio XSD (`.xsd`)
- Materials Studio XTD (`.xtd`)
- WIEN2k STRUCT (`.struct`)
- VASP-style structure files with `.vasp`

## Drag and drop

The Windows build uses `tkinterdnd2` to support native file and folder drag-and-drop.

You can drop:

- one file
- many files with mixed formats
- one folder
- many folders
- files and folders at the same time

Dropped folders are scanned recursively by default. Disable `Recursive folder scan` if needed.

## CIF partial and mixed occupancy

CIF files are read with fractional occupancy information enabled.

When a crystallographic site has partial or mixed occupancy, the converter automatically converts it to an integer-occupancy site before writing the result:

- a site containing one partially occupied species keeps that species and is treated as occupancy 1
- a site containing several species keeps the species with the highest occupancy
- if several species have the same highest occupancy, the representative species selected by the CIF reader is retained
- fractional occupancy metadata is removed after ordering

The conversion log reports how many CIF sites were integerized.

This is a deterministic single-cell ordering rule. It does not create a supercell or attempt to reproduce the exact bulk composition of a disordered material. Use a dedicated SQS or configurational-ordering workflow when composition-preserving disorder models are required.

## CP2K KIND labels

CP2K atom-kind labels do not need to be plain element symbols.

Examples that are supported:

```text
Mn_A
Mn_B
Fe_1
Fe_2
O_surface
C_A
```

If `&KIND` contains an `ELEMENT` line, that element definition has priority. Otherwise the element is inferred from the kind-label prefix. This avoids interpreting `C_A` as calcium while still interpreting `Mn_A` as manganese.

Example:

```text
&KIND Mn_A
  ELEMENT Mn
&END KIND

&KIND Mn_B
  ELEMENT Mn
&END KIND

&COORD
  Mn_A 0.0 0.0 0.0
  Mn_B 1.0 1.0 1.0
&END COORD
```

Both atoms are written as `Mn` in the converted structure.

## Periodicity rules

- Periodic input -> XYZ or GJF: cell and periodic flags are removed.
- Isolated input -> CIF or POSCAR: the structure is centered in a cubic cell.
- The default isolated-system cell is `30 x 30 x 30 Angstrom`.
- If the molecule does not fit safely in the requested cell, the cube is enlarged automatically.
- Existing periodic cells are preserved for CIF/POSCAR output.
- PDB `CRYST1`, MOL2 `CRYSIN`, XSF cells, and other readable periodic cell information are preserved when available.

## GUI usage from source

Install dependencies:

```text
python -m pip install -r requirements.txt
```

Run:

```text
python ChemStructConverter.py
```

You can also use `Add Files` and `Add Folder` if drag-and-drop is not available.

If the output directory field is empty, each input directory receives a `converted` subfolder.

## Command-line usage

```text
python ChemStructConverter.py FILE1 FILE2 --to xyz
python ChemStructConverter.py CALC_FOLDER --to cif --box 30
python ChemStructConverter.py result.log --to poscar-direct --outdir converted
```

Allowed values for `--to`:

```text
xyz
gjf
cif
poscar-direct
poscar-cartesian
```

## Build a single Windows EXE

Use a 64-bit CPython installation on Windows. Python 3.11, 3.12, or 3.13 is recommended.

From PowerShell or Command Prompt in this directory:

```text
py build_windows.py
```

The build script creates a clean virtual environment, installs the required packages, includes the Tk drag-and-drop binaries through the bundled PyInstaller hook, and builds:

```text
dist\ChemStructConverter.exe
```

The target computer does not need Python, ASE, or tkinterdnd2 installed.

## Important parser limits

- ORCA `* xyzfile` inputs are not self-contained and are rejected unless converted to inline coordinates first.
- CP2K inputs that obtain coordinates exclusively through external includes or topology files are not self-contained and are rejected.
- SIESTA FDF inputs that obtain the coordinate block exclusively through `%include` are not self-contained and are rejected.
- Highly customized or nonstandard output formatting may require a parser update.
- Conversion preserves geometry and periodic cell information where applicable; it does not attempt to translate calculation settings, basis sets, pseudopotentials, constraints, velocities, or electronic-structure parameters.
