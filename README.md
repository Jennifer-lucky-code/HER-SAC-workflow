# HER-SAC-workflow

An active-learning workflow for hydrogen evolution reaction (HER) screening of single-atom catalysts (SACs), developed by Jennifer-lucky-code.

## Workflow

- Candidate-log parsing
- POSCAR generation and VASP input preparation
- HPC job tracking
- HER free-energy analysis
- Dataset updates

See [her-sac-workflow](her-sac-workflow/) for the workflow files.

Before use, fill in your actual HPC project IDs and storage paths in `her-sac-workflow/references/vasp_settings.md`.

Requirements: Python 3 (standard library); `matplotlib` only for `her_analysis.py --plot`
(`pip install -r her-sac-workflow/requirements.txt`). Run the regression tests with
`cd her-sac-workflow && python -m unittest discover -s tests`.
