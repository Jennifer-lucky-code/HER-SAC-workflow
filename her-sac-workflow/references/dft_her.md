# DFT and HER analysis steps

Read this for loop steps 4–8 (structure generation, VASP jobs, checks, free energies, mechanism, stability).

## 1. Model and naming

- Model: a transition-metal (TM) atom coordinated by four ligand atoms X ∈ {C, N, O, P, S} in a graphene supercell.
- Name each configuration `TM-X1X2X3X4@G` (e.g. `Ir-CCNS@G`), using the canonical (rotation-reduced) ligand order.
- Symmetry reduction: map every ligand arrangement to its canonical rotationally invariant form before running
  DFT, so equivalent structures are calculated once. 28 TMs × 5⁴ arrangements = 17,500 nominal → 4,900 unique.
- Keep one folder per configuration: `<name>/{slab, H1_metal, H1_ligand, H2_*, freq_*}`.

## 2–4. VASP calculations

Settings used in the TMX4@G project are in `references/vasp_settings.md`. Read it before writing any INCAR.

- **Bare catalyst (`*`)**: relax, spin-polarised.
- **First H (`H*`)**: try H on top of the metal and on each inequivalent ligand site; keep the lowest-energy
  converged structure and record its site.
- **Second H (`2H*`)**: start from the best `H*` and always compute BOTH Tafel configurations:
  - **2H*-C**: second H on an adjacent C atom (metal–C cooperative recombination).
  - **2H*-M ("horns")**: both H on the same metal atom, pointing up like two horns (dihydride).
  Each gives its own Volmer–Tafel route (VT-C, VT-M); keep both, do not keep only the lower-energy one.
- After relaxation, check the H geometry (section 6): an H–H distance below ~0.9 Å means H₂ formed or a
  dihydrogen complex; label it, do not silently treat it as 2H*.

## 5. Frequencies (ZPE and TS)

Fixed atoms are used ONLY here. Structure relaxations (`*`, `H*`, `2H*`) fix no atoms.
Make the frequency POSCAR from the relaxed CONTCAR with `python scripts/make_freq_poscar.py CONTCAR`
(all substrate atoms F F F, all H atoms T T T).


- Run a frequency job for every adsorbed state, with only the H atoms free and the slab fixed.
- G = E_DFT + E_ZPE − TS, T = 298.15 K (ZPE/TS via VASPKIT 501, E_DFT = OSZICAR `E0`; commands in
  `references/vasp_settings.md`). The bare slab uses G_* = E_*.
- G_H₂ for gas-phase H₂: use the project reference value in `references/vasp_settings.md`; never mix references
  from different setups.
- Imaginary modes on H: report them; the structure may not be a minimum.

## 6. Checks before any result enters the dataset

A job passes only if all hold:

- Electronic SCF converged in every ionic step, and the final ionic step reached EDIFFG (forces < 0.01 eV/Å).
  The job log must end with `reached required accuracy - stopping structural energy minimisation`
  (last or second-to-last line).
- No error messages in the output or scheduler log; the job finished normally.
- H stayed bound: metal–H or ligand–H distance within a sensible bonding range; no H drifted into vacuum.
- Final magnetic moment recorded; note large changes between `*`, `H*` and `2H*`.
- Structure integrity: TM still four-coordinated, no large out-of-plane distortion. Flag symmetry breaking
  (Jahn–Teller-like distortion) in the log rather than discarding the job.

Failed jobs are fixed automatically (no need to ask the user), found by the 5-hourly `scripts/check_jobs.py` run:

1. Back up before changing anything: `log` → `log-N`, `INCAR` → `INCAR-N` (N = retry number).
2. Apply the one fix the script names for the error (its table covers walltime, memory, POTCAR, MAGMOM,
   ZBRENT, BRIONS, LREAL, symmetry, eigensolver, BRMIX, NBANDS, crashes). For a relaxation that had made
   progress, `cp CONTCAR POSCAR` first so the restart keeps it.
3. Never change anything that shifts the energy reference to make a job run: ENCUT, PREC, IVDW, ISPIN,
   POTCAR set, k-spacing, cell. If the only fix would need that, flag the job instead.
4. Resubmit, write the new JOBID, add the retry to `loop_state.json`.
5. After 3 failed retries of the same job, stop and flag it for the user with the errors seen.

Log every action (what failed, what was changed, new job ID) in `workflow_log.md`, and keep the changed
setting in the dataset row's notes so it is clear which jobs deviate from the template INCAR.

## 7. Free energies, mechanism and overpotential

Run `python scripts/her_analysis.py energies.csv --plot` (input columns are documented at the top of the script).
It computes, with the reference state * + 2H⁺ + 2e⁻ = 0 at U = 0 V:

- ΔG_H* = G_H* − G_* − ½G_H₂ (first Volmer)
- ΔG_2H* = G_2H* − G_H* − ½G_H₂ (second Volmer, a step value, not the cumulative 2H* level)
- ΔG_Tafel = −(ΔG_H* + ΔG_2H*) (chemical step, no electron transfer)
- ΔG_Heyrovsky = −ΔG_H*

Mechanism: only uphill (positive) steps cost energy.

- η_VH = largest positive step of {ΔG_H*, −ΔG_H*} = |ΔG_H*|
- η_VT = largest positive step of {ΔG_H*, ΔG_2H*, ΔG_Tafel}, computed separately for VT-C and VT-M
- Preferred pathway = the smallest of η_VH, η_VT-C, η_VT-M; report its value as the limiting potential.
- If one 2H* configuration is missing, say so next to the result.
- If the largest VT step is the Tafel step, say so: it is chemical and cannot be lowered by applied potential.

Keep the project's screening descriptor separate and clearly named: η_HER = max(|ΔG_H*|, |ΔG_2H*|) (Eq. 4 of
the TMX4@G paper, using the more stable 2H* configuration). Never mix screening-descriptor values and mechanism values in one column or one figure axis.

Worked check (Cr-PSSS@G, from the project data): ΔG_H* = −0.352; 2H* levels −0.397 (C) and −0.401 (M)
→ η_VH = 0.352, η_VT-C = 0.397, η_VT-M = 0.401 → Volmer–Heyrovsky, 0.35 V. Use this to test any new script version.

## 8. Stability

- E_f = E_TMX4@G − E_X4@G − E_TM (E_TM per atom of the bulk metal). E_f < 0 → thermodynamically stable.
- For shortlisted candidates only: AIMD (see `references/vasp_settings.md`) and check the TM stays anchored.

## 9. Dataset row (DFT part)

Append one row per configuration to the dataset with: name, TM, ligands, E_f, ΔG_H*, H* site, ΔG_2H* (C and M),
η_VH, η_VT-C, η_VT-M, preferred pathway, η_HER (screening), magnetic moment, d-band centre (VASPKIT), job paths, date.
Never overwrite existing rows; if a value changes, add a new row with a note and keep the old one.

