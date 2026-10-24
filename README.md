## DVC Practice — Project Flow with Explanations

---

### 1. Create a Git Repository and Clone It Locally

Create a repository on GitHub and clone it onto your laptop. This provides a local working directory called `DVC`, where you can write code, generate data, and track changes using Git.

```bash
git clone <repository-url>
cd DVC
```

---

### 2. Create `mycode.py` and Generate the CSV File

Write your Python code in `mycode.py` and execute it. Your code creates a `data/` directory and saves `sample_data.csv` inside it. Initially, the CSV contains Alice, Bob, Charlie, Daniel, and Edith because your code adds two new rows to the original three in the later stages.

```bash
python mycode.py
```

---

### 3. Run Git Add, Commit, and Push Before Initializing DVC

Stage, commit, and push your initial code and data.

```bash
git add .
git commit -m "Add initial code and data"
git push origin main
```

> **Why do this first?**  
> At this stage, Git tracks both `mycode.py` and the generated CSV file. We do this to understand the transition from Git-based data tracking to DVC-based data tracking. Later, we will stop tracking the actual dataset with Git.

---

### 4. Install DVC and Initialize It

Install DVC using `pip` and initialize it inside your repository root.

```bash
pip install dvc
dvc init
```

* **What `dvc init` does:** Initializes DVC in your repository.
* **What is created:**
  * `.dvc/`: Internal configuration and metadata directory.
  * `.dvcignore`: Specifies files and directories DVC should ignore.
* DVC is now ready to manage datasets separately from Git, while Git continues tracking your code and DVC metadata.

---

### 5. Create an `S3/` Directory

Create a local directory to act as your remote storage.

```bash
mkdir S3
```

> **Note on Naming:**  
> In this project, `S3/` is a local directory used to simulate remote storage. Despite its name, it is not an actual AWS S3 bucket. You will use it to understand how DVC stores data outside Git's normal version history.

---

### 6. Configure the DVC Remote

Configure the newly created `S3/` directory as DVC's default remote target.

```bash
dvc remote add -d myremote S3
```

* **Command breakdown:**
  * `-d`: Sets `myremote` as the default remote.
  * `myremote`: The alias for your remote storage endpoint.
  * `S3`: The target directory path.
* **Separation of Concerns:**
  * **GitHub:** Stores your code, configuration, and lightweight `.dvc` metadata.
  * **DVC Remote (`S3/`):** Stores the actual data files and binaries.

---

### 7. Run `dvc add data/` for the First Time

Attempt to add the data directory to DVC.

```bash
dvc add data/
```

Initially, Git is already tracking `data/`. DVC detects this collision and warns you to stop tracking the dataset directly with Git. Run the commands suggested by DVC:

```bash
git rm -r --cached data
git commit -m "Stop tracking data"
```

> **Why `--cached`?**  
> The `--cached` option removes `data/` from Git's index/staging tree without deleting the actual files from your disk. This is essential because we want Git to track only the dataset's metadata through DVC rather than maintaining full copies of every data revision.

---

### 8. Run `dvc add data/` Again

Now that Git has released tracking of `data/`, hand it over to DVC:

```bash
dvc add data/
```

* **What happens:** DVC creates `data.dvc` and automatically appends `data/` to `.gitignore` to prevent Git from tracking the raw files again.
* **What `data.dvc` is:** A small metadata text file containing the dataset path and an MD5 hash identifying its exact contents.
* Git tracks `data.dvc`, while DVC manages the actual dataset. The MD5 hash allows DVC to determine whether the dataset has changed and identify which version to retrieve.

Stage and commit the newly created metadata files to Git:

```bash
git add .gitignore data.dvc
git commit -m "Track data with DVC - V1"
git push origin main
```

---

### 9. Run `dvc commit` and `dvc push`

Synchronize your data with your DVC remote storage:

```bash
dvc commit
dvc push
```

* **`dvc commit`:** Records the current state of data for an existing DVC-tracked output, typically after a change is detected. (In this initial workflow, `dvc add` has already cached the data, so `dvc commit` is generally optional here).
* **`dvc push`:** Uploads the required data objects from `.dvc/cache` to your configured remote storage (`S3/`).

---

### 10. Make Changes to `mycode.py` to Create Version 2

Modify your Python script to append another row to the dataset, then run it:

```bash
python mycode.py
dvc status
```

* Running the script regenerates `data/sample_data.csv`.
* `dvc status` compares the current hash of the dataset against what is recorded in `data.dvc` and detects that the contents no longer match.

---

### 11. Record and Upload Version 2

Capture the updated dataset, upload it to the remote, and record the new pointer in Git:

```bash
dvc push
dvc commit
git add .
git commit -m "Track data - V2"
git push origin main
```

* `dvc push` uploads the new data payload to `S3/`.

---

### 12. Check Git and DVC Status

Verify that both tracking systems are clean and synchronized:

```bash
git status
dvc status
```

* **`git status`:** Checks for changes to Git-tracked files (code, `.dvc` files, `.gitignore`).
* **`dvc status`:** Checks whether the raw data on disk matches the checksum in `data.dvc`.
* After committing and pushing, both commands should report clean working trees.

---

### 13. Repeat Steps 10–12 to Create Version 3

Modify `mycode.py` once more, execute it, check the status, update metadata, push the data, and commit the pointer:

```bash
python mycode.py
dvc status
dvc commit
dvc push
git add .
git commit -m "Track data - V3"
git push origin main
```

You now have three distinct dataset versions, each represented by a specific Git commit and a corresponding DVC hash:

```text
Git Commit (Main Timeline)       data.dvc Hash       Underlying Dataset
───────────────────────────       ─────────────       ──────────────────
e5752a0 ("Track data - V3")  ───> [Hash V3]      ───> 5 rows in sample_data.csv
fdadbd0 ("Track data - V2")  ───> [Hash V2]      ───> 4 rows in sample_data.csv
24e3308 ("Track data - V1")  ───> [Hash V1]      ───> 3 rows in sample_data.csv
```

Git controls **which** data version you want; DVC retrieves the **actual** data.

---

## 14. Restore an Older Version of the Data for Viewing

Suppose your Git history looks like this:

```text
e5752a0  Third version of data
fdadbd0  Second version of data
24e3308  First version of data
```

To view Version 2 without altering the `main` branch or creating an extra commit:

```bash
git status
git checkout fdadbd0
dvc pull
```

* **`dvc pull`:** Reads the newly restored `data.dvc` hash and retrieves the corresponding dataset version from remote storage into your working directory.
* Open `data/sample_data.csv` to confirm the older state of the dataset.

### Returning to Version 3:

```bash
git checkout main
dvc pull
```

This returns HEAD to the latest commit on `main` and pulls the latest dataset.

---

### Conceptual Summary

### System Layout

```text
DVC/
├── .dvc/
│   └── cache/          # Stores local content-addressable data objects
├── data/
│   └── sample_data.csv # Active working copy you see and edit
├── data.dvc            # Small pointer file tracking the data version hash
└── S3/                 # Your configured DVC remote storage
```

### Command Flow

1. **`dvc add data/`** — Computes the dataset hash, copies the data into the local `.dvc/cache/`, and updates `data.dvc`.
2. **`dvc push`** — Copies cached data objects from `.dvc/cache/` to the remote storage (`S3/`).
3. **`dvc pull`** — Fetches any missing objects from the remote storage into the cache and restores the workspace files.