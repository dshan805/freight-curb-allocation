# freight-curb-allocation

CET 513 group project (Autumn 2026): a network optimization model to minimize
the time commercial delivery vehicles spend double-parking in Downtown Seattle
by improving the allocation and use of existing loading zones.

## Folder structure

```
freight-curb-allocation/
├── data/            # raw and processed data (not on Git, see .gitignore)
├── notebooks/       # exploration and analysis (e.g. 01-eda.ipynb)
├── lit_review/      # literature review materials
├── src/             # model and reusable code
├── outputs/         # figures and results (not on Git)
└── docs/            # proposal, report, references
```

## Setup

1. Clone the repo.
2. Create any missing folders (PowerShell):
   ```powershell
   mkdir data, notebooks, src, outputs, docs
   ```
3. Download the data from the shared Google Drive folder into `data/`.
4. Create and activate a virtual environment (PowerShell, from the project folder):
   ```powershell
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```
   If activation is blocked by the execution policy, run this once:
   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
   ```
5. Install dependencies:
   ```powershell
   pip install -r requirements.txt
   pip install ipykernel
   ```
6. To use the notebooks in VS Code, open a notebook, click **Select Kernel**,
   and choose the `.venv` environment.

Activation only lasts for the current terminal session, so repeat step 4's
activation command in each new terminal (VS Code may do this automatically
once `.venv` is selected as the Python interpreter).
