---
name: her-sac-workflow
description: Closed-loop, active-learning HER screening workflow for single-atom catalysts (SACs) such as TMX4@G / M-N4/C. Covers the full loop - asking the user to run their private screening program, parsing the candidate log they provide (metal Z, ligands, predicted ΔG ± σ), POSCAR generation by element substitution, VASP input assembly, HPC submission and job tracking, failure handling, result processing (H*/2H* free energies, Volmer–Heyrovsky vs Volmer–Tafel mechanism, overpotential, stability, feature extraction) and dataset update before retraining. Use this skill whenever the user asks to run, continue, check or analyse an active-learning cycle, select or submit catalyst candidates, set up or parse HER calculations, compute ΔG_H*, ΔG_2H*, overpotentials or free-energy diagrams, or update an HER/ML dataset, even if they do not say "skill" or "workflow".
---

# Active-learning HER workflow for single-atom catalysts

Reusable version of the TMX4@G workflow.

Note: Before use, fill in your actual HPC project IDs and storage paths as described in `references/vasp_settings.md`.

**Ground rule:** the agent orchestrates; it never invents numbers. Structures come from scripts, energies from
VASP, descriptors and free energies from scripted post-processing, predictions from the user's screening program.
If a value is missing, report it instead of estimating it.

## The loop

```
 DFT-labelled dataset ──▶ (1) user runs screening ──▶ (2) parse log ──▶ (3) user picks candidates
        ▲                                                                           │
        │                                                                           ▼
 (8) update dataset ◀── (7) result processing ◀── (6) track jobs ◀── (5) HPC ◀── (4) POSCAR + inputs
```

Keep a state file `loop_state.json` in the project root and update it after every step:
cycle number, dataset size, candidates picked, each job's path, scheduler ID and status
(`queued / running / done / failed / flagged`), retry count, and whether its results were added.
Always read it first when resuming, so a cycle is never repeated or skipped.

## Steps

**1. Ask the user to run the screening program.** The agent does not run it. When a new cycle starts
(dataset updated with the last results), remind the user to run their screening program to pick new
structures, then stop and wait. Set `loop_state.json` to `waiting_for_log`. Continue only when the user
provides the log file.

**2. Parse the log the user provided.** `python scripts/parse_candidates.py <log>` → `candidates.csv`
(name such as `Ru-PONS@G`, metal Z, ligand atomic numbers, ΔG ± σ). Report flagged blocks.

**3. Pick candidates.** Show the table and ask the user which ones to run this cycle. Skip anything already
calculated.

**4. Build inputs.** `python scripts/build_poscar.py candidates.csv --template assets/POSCAR_template --sites assets/sites.json --only <names>`
(element substitution on the project template, see `references/screening.md`), then assemble INCAR, KPOINTS and POTCAR from `references/vasp_settings.md`. Job set per candidate: bare `*`,
`H*` (metal and ligand sites), `2H*` in both Tafel configurations, frequency jobs. Relaxations fix no atoms;
only frequency (ZPE/TS) jobs fix the substrate (`scripts/make_freq_poscar.py`). Details in `references/dft_her.md`.

**5. Submit.** Submit to the HPC scheduler (Gadi: PBS; Setonix: SLURM). Ask the user before the first
submission of a session. Record every job ID in `loop_state.json`.

**6. Track and handle failures.** Poll job status. For a failed job, read only the relevant part of the output
(the tail of OUTCAR/OSZICAR and the scheduler error file, not whole files), diagnose, change one setting,
resubmit. After 3 failed retries, mark it `flagged` and tell the user. Log every action in `workflow_log.md`.

**7. Process results.** For each finished candidate, run the checks in `references/dft_her.md` §6, then:
- free energies, VH/VT mechanism and limiting potential: `python scripts/her_analysis.py energies.csv --plot`
- formation energy E_f (and AIMD only for shortlisted candidates)
- feature extraction for the ML dataset: ε_d (VASPKIT), Bader charge transfer, d_M−X, d_M−H, plus the
  screening target η_HER (Eq. 4) kept separate from the mechanism values.

**8. Update the dataset and loop.** Append one row per validated candidate to the dataset the screening program
reads (never overwrite old rows), then return to step 1. Stop when the stopping criterion in `references/screening.md` is met,
and report the cycle summary.

## Reference files

- `references/screening.md`: the user-run screening step, log format, POSCAR building, loop control.
- `references/dft_her.md`: structures, VASP jobs, checks, free energies, VH/VT mechanism, stability.
- `references/vasp_settings.md`: INCAR/KPOINTS settings, reference energies, HPC details.
- `scripts/parse_candidates.py`: screening log → `candidates.csv`.
- `scripts/build_poscar.py`: candidates → POSCARs from the template and site map.
- `scripts/make_freq_poscar.py`: relaxed CONTCAR → frequency POSCAR (substrate fixed, H free).
- `assets/POSCAR_template`, `assets/sites.json`: TMX4@G template structure and metal/ligand atom indices.
- `scripts/her_analysis.py`: ΔG_H*, ΔG_2H* (C and metal "horns" configurations), η_VH, η_VT, pathway, diagrams.

## Reporting to the user

At the end of each cycle give a short table: candidate, predicted ΔG ± σ, DFT
limiting potential, preferred pathway, E_f, flags.
List failed, flagged or missing items separately. Round to 2 decimals only for display.
