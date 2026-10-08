# VASP settings (TMX4@G project)

Values marked TODO were not in the manuscript. Fill them in from the project's own input files before first use;
until then, ask the user rather than guessing.

## Shared settings (from the manuscript methods)

| Setting | Value |
| --- | --- |
| Code | VASP 5.4.4, PAW, GGA-PBE |
| Spin | ISPIN = 2 (spin-polarised) |
| ENCUT | 500 eV |
| k-points | 4 × 4 × 1 (see KPOINTS note below) |
| EDIFF (relaxation) | 1E-5 |
| EDIFF (static) | 1E-6 |
| EDIFFG | -0.01 (forces < 0.01 eV/Å) |
| Pre/post-processing | VASPKIT (d-band centre, PDOS) |
| Vacuum | 15 Å (user-stated; not yet written in the manuscript) |
| Supercell size | TODO |
| vdW correction | DFT-D3 (IVDW = 11, zero damping) |
| Dipole correction (LDIPOL/IDIPOL) | TODO |
| Initial MAGMOM | not set in template (VASP default 1 μB per atom); set explicitly if spin states matter |
| POTCAR | generated with VASPKIT (its recommended PAW_PBE set) |
| Solvation (VASPsol) | TODO: used or not |

## Relaxation INCAR (project template)

Use this as is for `*`, `H*` and `2H*` relaxations. Only change SYSTEM, MAGMOM (if needed) and NPAR (machine-dependent).

```
# Collinear magnetic calculation
ISPIN      = 2
LASPH      = .TRUE.
GGA_COMPAT = .FALSE.
VOSKOWN    = 1
LMAXMIX    = 4        # d elements; use 6 for f elements
AMIX       = 0.2
BMIX       = 0.0001
AMIX_MAG   = 0.4
BMIX_MAG   = 0.0001

# DFT-D3 correction
IVDW   = 11           # DFT-D3, zero damping

# Global parameters
ISTART = 0            # start from scratch
LREAL  = Auto
ENCUT  = 500
PREC   = Accurate
LWAVE  = .TRUE.
LCHARG = .FALSE.      # set .TRUE. when Bader charges are needed
ADDGRID = .TRUE.
NPAR   = 8            # depends on node count

# Electronic relaxation
ISMEAR = 0
SIGMA  = 0.05
NELM   = 90
NELMIN = 6
EDIFF  = 1E-05        # 1E-06 for static single-point runs

# Ionic relaxation
NSW    = 500
IBRION = 2
ISIF   = 2
EDIFFG = -1E-02

LORBIT = 12
```

Static single-point (for ε_d, PDOS, Bader): same file with NSW = 0, IBRION = -1, EDIFF = 1E-06,
LCHARG = .TRUE., plus LAECHG = .TRUE. for Bader.

## KPOINTS (project template)

```
K-Spacing Value to Generate K-Mesh: 0.030
0
Monkhorst-Pack
   4   4   1
0.0  0.0  0.0
```

Note: the manuscript methods say "Γ-centred Monkhorst–Pack 4 × 4 × 1", but this file uses a plain
Monkhorst-Pack grid, which for an even mesh is not Γ-centred. Confirm with the user which one the production
runs used before generating new jobs, and keep new jobs identical to the existing dataset.

## Frequency INCAR (project template)

Fix all slab atoms with selective dynamics (F F F); leave only the H atoms free (T T T).
Use `scripts/make_freq_poscar.py`. This is the only job type with fixed atoms; relaxations fix nothing.

```
# Collinear magnetic calculation
ISPIN  = 2
LPLANE = .TRUE.
LSCALU = .FALSE.
NSIM   = 4

# DFT-D3 correction
IVDW   = 11

# Global parameters
ISTART = 0
LREAL  = Auto
ENCUT  = 500
PREC   = Normal
LWAVE  = .FALSE.
LCHARG = .FALSE.
ADDGRID = .TRUE.
ALGO   = Fast

# Electronic relaxation
ISMEAR = 0
SIGMA  = 0.05
NELM   = 90
NELMIN = 6
EDIFF  = 1E-06        # tighter for frequencies
EDIFFG = -1E-02

# Frequency calculation
IBRION = 5            # finite differences
NFREE  = 2            # central differences
POTIM  = 0.015        # displacement in Å
NSW    = 1

LORBIT = 12
```

ZPE and TS from the real modes via VASPKIT (TODO: record the exact VASPKIT option used).

## Reference energies

| Quantity | Value (eV) | Note |
| --- | --- | --- |
| G_H₂ | -6.8901 | From the project spreadsheet (H2+surface − surface). TODO confirm whether this includes ZPE and TS. |
| E_TM (bulk, per atom) | TODO | One value per metal |

## AIMD (shortlisted candidates)

TODO: ensemble, temperature, time step, total time, thermostat. Pass criterion: TM remains anchored and
coordination is retained.

## HPC

Note: Before use, replace `{PROJECT_ID}` with your actual HPC allocation/project ID, `{STORAGE_PROJECT_ID}` with your storage project ID, and `{STORAGE_PATHS}` with the storage paths required by your HPC system.

| System | Scheduler | Project | Notes |
| --- | --- | --- | --- |
| NCI Gadi | PBS Pro | {PROJECT_ID} (storage {STORAGE_PROJECT_ID} + {PROJECT_ID}) | template below |
| Pawsey Setonix | SLURM | {PROJECT_ID} / {STORAGE_PROJECT_ID} | TODO: job script template, modules |
| QUT Aquarius | TODO | | |

### Gadi job script (project template)

Fill every `{PLACEHOLDER}` before submitting; ask the user for any value not given for the job.
`{NCPUS}` must be a multiple of the cores per node of `{QUEUE}`, and NPAR a divisor of `{NCPUS}`.

```bash
#!/bin/bash
#PBS -P {PROJECT_ID}
#PBS -N {JOBNAME}
#PBS -q {QUEUE}
#PBS -l ncpus={NCPUS}
#PBS -l walltime={WALLTIME}
#PBS -l mem={MEM}
#PBS -l software=vasp
#PBS -l wd
#PBS -j oe
#PBS -l storage={STORAGE_PATHS}

module purge
module load vasp/5.4.4

mpirun -np {NCPUS} vasp_std > log
```
