# Environment, Project Scaffolding and Data Versioning

## Goal

The goal for this stage of the project was to set up the foundations for the project. This included an isolated Python environment, a clean package layout, and version controlled access to the raw Open Food Facts dataset without committing large files to git.

## Tools Used

| Tool                       | Purpose                                             |
| -------------------------- | --------------------------------------------------- |
| `venv`                     | Create an isolated Python environment               |
| `pyproject.toml`           | Project metadata and declares dependencies          |
| Git and GitHub             | Source control for the code                         |
| DVC (Data Verison Control) | Versioning for large data files, decoupled from Git |
| Local filesystem remote    | DVC remote storage backend (see notes below)        |

## What Was Done

### 1. Repo Setup

Created an empty GitHub repository and cloned it locally.

### 2. Python environment

Isolated virtual environment using venv. Dependencies are declared in `pyproject.toml`.

### 3. Folder structure

Adopted an src/ package layout to avoid accidental local-import bugs and to make the package installable.

### 4. .gitignore

Excludes the virtual environment, Python cache files, notebook checkpoints, MLFlow run artifacts, secrets, the contents of `data/raw/` and `data/processed`.

Two details that came up while doing setup:

- Empty folders aren't traclked by git, so .gitkeep placeholder files were added inside data folders to preserve the structure on a fresh clone.

- DVC's pointer files (`*.dvc`) need to be tracked by git even thoiugh they live inside otherwise ignored data folders. The blanket `data/raw/*` exclusion rule blocked these until explicit exceptions were added.

### 5. Data versioning with DVC

`dvc init` created a .dvc/ directory (small config/metadata which is tracked by git) which manages large files separately from git's own history

### 6. Got the raw dataset

Downloaded the Open Food Facts product database (Parquet format, ~4.4GB) directly from the offciial HuggingFace mirror.

`curl -L -o data/raw/food.parquet \`
`"https://huggingface.co/datasets/openfoodfacts/product-database/resolve/main/food.parquet?download=true"`

### 7. Tracked the dataset with DVC

`dvc add data/raw/food.parquet` computes a content hash for the file and stores it in a small pointer file. The real file is added to a local .gitignore automatically so git never touches it directly. The .dvc pointer
file is what gets versioned going forward.

`git add data/raw/food.parquet.dvc .gitignore`

### 8. Remote storage

Original plan was a Google Drive remote. This was chosen to keep infrastructure cost at zero and required registering a personal OAuth client in Google Cloud Console (Google blocks DVC's own shared/default OAuth client, since it hasn't gone through Google's app verification process), which involved:

- Creating a Google Cloud project and enabling the Drive API
- Configuring an OAuth consent screen (External user type, Testing publish status, own account added as a test user)
- Creating a Desktop-app OAuth client and wiring the client ID/secret into DVC via `dvc remote modify storage --local`
- Declaring the specific OAuth scopes DVC requests (`drive` and `drive.appdata`)
  on the consent screen. These don't appear in the default scope picker and had
  to be added manually by URL

Even though all of the above were configured correctly, authentication kept failing with `Error 403: access_denied`. The detailed error pointed to Google's brand verification requirements for restricted scopes (a stricter tier than "sensitive" scopes), which seem to need additional consent-screen configuration (e.g. a privacy policy URL) beyond what's normally needed for a personal Testing-mode app. Thought this was disproportionate setup for a single-user personal project.

Rather than continue down the verification path, switched the default DVC remote to a local filesystem path:

`dvc remote add -d storage-local C:/Users/charl/dvc-backup`
`dvc push`

This does satisfy DVC's technical requirements. Data is content-addressed,
versioned, and fetchable via dvc pull by anyone with access to that path. Howeevr, it is not an off-machine backup, since it lives on the same physical disk as the working copy. This is follow-up work (see below), not treated as equivalent to a real cloud backup.
