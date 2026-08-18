# Arrhenius for dummies

This is a guide for NAISS/SUPR users running jobs on **Arrhenius**, the new NAISS/EuroHPC supercomputer hosted at NSC in Linköping.


## DISCLAIMER:
🚨🚨 Necessary disclaimer, this guide was created by PhD students in mathematics with little knowledge on HPC systems and basic programming skills. 
With this guide, we mainly hope to achieve the following:
* Share our knowledge & experiences with Dardel and, hopefuly, save you valuable time that you can spend on what you love 🫂.
* Help other KTH researchers who are struggling to set up the environment to run their codes faster and in an optimal way.
* Generate constructive discussion that benefits everyone 💬.
* Any comment and suggestions are more than welcome, the ultimate goal is to help each other 🤝. 


## Create a project

1. Check available resources at https://supr.naiss.se/resource/. PhD students can typically only apply for NAISS Small Compute to start.
2. Go to https://supr.naiss.se/ and log in with your institutional account.
3. Click on the chosen compute resource → "Create New Proposal for NAISS Compute ...".
4. Fill in the required fields.
5. Once approved, request a login account for Arrhenius specifically through SUPR — hardware access and project allocation are separate steps.

## SSH Setup and Access

### Creating SSH keys

Same as any NAISS system — ed25519 or RSA accepted.

```bash
ssh-keygen -t ed25519
cat ~/.ssh/name_key.pub
```

Add the public key through the SUPR account page for your Arrhenius login.

### Connecting

```bash
ssh username@login.hpc.arrhenius.naiss.se
```

You'll be prompted for your password and a 2FA verification code on every connection. Because of this, prefer `rsync` over `scp` for anything that might be interrupted mid-transfer (see [File Transfer](#file-transfer) below) — resuming a dropped connection is much less painful than re-authenticating and restarting from zero.

### VSCode SSH configuration

```
Host arrhenius
    HostName login.hpc.arrhenius.naiss.se
    IdentityFile ~/.ssh/keyname
    IdentitiesOnly yes
    User username
```

## Storage layout

This is the single biggest thing to get right early — most early errors on a fresh Arrhenius account trace back to using the wrong path.

| Area | Path | Notes |
|---|---|---|
| Home | `/home/username` | Small, personal. Fine for scripts and env files, not for containers/datasets. |
| Project storage | `/nobackup/proj/disk/PROJECTNAME/personal/username/` | Where you should put containers, datasets, output, and pip caches. `PROJECTNAME` is your `naiss20XX-XX-XXXXX` allocation id. |
| Migrated Tetralith data | `/nobackup/proj/disk/PROJECTNAME/from-tetralith/` | If your project's data was auto-migrated from Tetralith, look here before assuming it's missing. |


### Checking space

```bash
df -h /nobackup/proj/disk/PROJECTNAME
du -h --max-depth=1 /nobackup/proj/disk/PROJECTNAME/personal/username | sort -hr
```

## Running Code on Arrhenius

Arrhenius has two architectures with **separate module trees and separate CPU/GPU login behavior**:

- **CPU partition**: AMD EPYC "Turin" (`epyc9005`), x86_64.
- **GPU partition**: NVIDIA Grace Hopper Superchips (GH200), **aarch64**. This matters a lot — anything you build for the GPU partition (containers, compiled code) must target `aarch64`, not `x86_64`.

The **login node is x86_64**. This means you cannot run or verify an aarch64 GPU container from the login node at all — `apptainer exec` on an aarch64 image from the login node fails with an architecture mismatch. All container verification has to happen inside a GPU allocation.

### Resource allocation

```bash
# Interactive CPU allocation
interactive -A naiss20XX-X-XXXX -p main --nodes=1 --time=02:00:00

# Interactive GPU allocation — must request GPUs explicitly, they are not implied by -p gpu
interactive -A naiss20XX-X-XXXX-gpu -p gpu -N 1 --gpus-per-node=1 -t 02:00:00
```

A common mistake: submitting to `-p gpu` **without** `--gpus-per-node` (or `--gres=gpu:N`) gives you a job on the GPU partition with four GPUs allocated. Always request GPUs explicitly.

### Sbatch template — CPU

```bash
#!/bin/bash -l
#SBATCH -A naiss20XX-X-XXXX
#SBATCH -J namejob
#SBATCH -p main
#SBATCH -t 02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH -e /nobackup/proj/disk/PROJECTNAME/personal/username/logs/name_%j.e
#SBATCH -o /nobackup/proj/disk/PROJECTNAME/personal/username/logs/name_%j.o
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=you@example.com

set -euo pipefail
mkdir -p /nobackup/proj/disk/PROJECTNAME/personal/username/logs

# No modules needed for a plain Python/conda environment — see below
source ~/miniforge3/etc/profile.d/conda.sh
conda activate myenv

cd /nobackup/proj/disk/PROJECTNAME/personal/username/myproject
python3 -u main.py
```


### CPU environment: Miniforge + pip

There's no central PyTorch/scientific-stack module on the CPU or GPU trees as of writing — check what's installed for your case first:

```bash
module avail 2>&1 | sed -n '/CPU partition/,/^---/p'   # x86_64 tree
module avail 2>&1 | sed -n '/GPU partition/,/^---/p'   # aarch64 tree
```

If nothing suitable is there, build your own environment with Miniforge (available as a module on both trees):

```bash
module load Miniforge/recommendation      # exact module name varies; check `module avail Miniforge`

export CONDA_ENVS_PATH=/nobackup/proj/disk/PROJECTNAME/personal/username/envs
export PIP_CACHE_DIR=/nobackup/proj/disk/PROJECTNAME/personal/username/pip_cache

conda create -n myenv python=3.12 -y
conda activate myenv
pip install numpy scipy pandas ...
```

Do the `conda create`/`pip install` step on a **login node** (or a CPU interactive session — login nodes have internet access, compute nodes may not), and keep the environment off your home directory. Conda environments are tens of thousands of small files; on a shared Lustre filesystem that's slow for you and everyone else. Once built, `conda activate myenv` works the same in a batch job as interactively.

### Running dynamically (interactive) on CPU

```bash
interactive -A naiss20XX-X-XXXX -p main --nodes=1 --time=02:00:00
conda activate myenv
python3 main.py
```

## GPU workflow: containers

There is no equivalent of Dardel's `/pdc/software/resources/sing_hub/` container library on Arrhenius. You pull and manage your own containers with **Apptainer**, which is preinstalled system-wide — **no module load needed**, `apptainer`/`singularity` are directly in `/usr/bin`.

### 1. Set up paths (do this every session — see the pitfall below)

```bash
export BASE=/nobackup/proj/disk/PROJECTNAME/personal/username
export APPTAINER_CACHEDIR=$BASE/apptainer_cache
export APPTAINER_TMPDIR=$BASE/apptainer_tmp
mkdir -p "$APPTAINER_CACHEDIR" "$APPTAINER_TMPDIR" "$BASE/containers"
```

Both `CACHEDIR` and `TMPDIR` matter — without them Apptainer defaults to `/tmp` on whatever node you're on, which is small and will kill a multi-GB pull partway through.

**Put these exports in a small file you `source` every time you get a new session or allocation:**

```bash
cat > ~/myenv.sh <<'EOF'
export BASE=/nobackup/proj/disk/PROJECTNAME/personal/username
export PROJECT=$BASE/myproject
export SIF=$BASE/containers/pytorch_ngc.sif
export APPTAINER_CACHEDIR=$BASE/apptainer_cache
export APPTAINER_TMPDIR=$BASE/apptainer_tmp
EOF
```

Then `source ~/myenv.sh` as the **first thing** in any new shell — every `interactive` allocation and every reconnection after 2FA starts a fresh shell with none of these set. Forgetting this step causes cryptic errors like `apptainer image is not in an allowed configured path` (which really just means the `$SIF` variable was empty) or `cat: /file.txt: No such file or directory` (a path variable silently resolving to nothing).

### 2. Pull an NGC container (from the login node)

```bash
cd $BASE/containers
apptainer pull --arch arm64 pytorch_ngc.sif docker://nvcr.io/nvidia/pytorch:25.06-py3
```

**`--arch arm64` is not optional.** The login node is x86_64; without this flag, `apptainer pull` silently grabs the x86_64 variant of a multi-arch image, which will fail with an architecture mismatch the moment you try to run it on a GH200 node. Always verify after pulling:

```bash
apptainer exec "$SIF" python -c "import platform; print(platform.machine())"
```

This check must be run **inside a GPU allocation**, since the login node can't execute an aarch64 binary at all — you'll get `the image's architecture (arm64) could not run on the host's (amd64)` if you try from the login node, which is actually a *good* sign (it confirms the image really is aarch64).

Check the [NGC PyTorch container catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch/tags) for the current recommended tag.

Image size is typically 10–15 GB; the pull takes 15–25 minutes. NGC's warnings about `rootless{...} EPERM on setxattr` during the pull are harmless — ignore them.

### 3. Install extra Python packages without breaking the container

The NGC image ships numpy, scipy, torch (built for aarch64+CUDA), and a fairly complete scientific stack — but not everything (matplotlib, torch_geometric, h5py, hydra-core, cellpose, etc. may all be missing depending on your workflow). Install missing packages into a separate directory bound in via `PYTHONPATH`, never into the container itself (the `.sif` is read-only anyway).

```bash
apptainer exec -B "$BASE:$BASE" "$SIF" \
  pip install --target=$BASE/pylibs --no-deps <package>
```

**Always use `--no-deps`.** This is the single most important rule in this whole workflow. Without it, pip's dependency resolver can decide to "fix" a supposed conflict by installing a *different* numpy, scipy, or torch into `pylibs` — and because `PYTHONPATH` (your `pylibs` directory) takes precedence over the container's own site-packages, that silently shadows the container's carefully matched, GH200-tuned build. The result is confusing downstream failures (`Failed to initialize NumPy`, ABI mismatches, CUDA no longer available) that look nothing like "wrong numpy version."

With `--no-deps`, install one package at a time, run the import, see what's still missing, repeat:

```bash
apptainer exec --nv -B "$BASE:$BASE" "$SIF" \
  bash -lc "export PYTHONPATH=$BASE/pylibs:\$PYTHONPATH; python -c 'import mypackage'"
```

If you genuinely need pip to resolve a real dependency tree (e.g. geospatial packages like `geopandas`/`rasterio`), first pin the container's own numpy/scipy/torch versions so pip can't touch them, then drop `--no-deps`:

```bash
apptainer exec "$SIF" python -c \
  "import numpy, scipy; print(f'numpy=={numpy.__version__}'); print(f'scipy=={scipy.__version__}')" \
  > $BASE/constraints.txt

apptainer exec -B "$BASE:$BASE" "$SIF" \
  pip install --target=$BASE/pylibs -c $BASE/constraints.txt <package>
```

Keep the constraints file to numpy/scipy only unless you know exactly what you're doing — pinning torch by its NGC pre-release version string (e.g. `2.8.0a0+...`) tends to make the resolver *worse*, not better, since pre-release identifiers don't satisfy ordinary `>=` requirements under PEP 440.

**If a package needs compiled CUDA extensions** (`torch_scatter`, `torch_sparse`, and similar PyG add-ons are the classic case): there are usually no aarch64 wheels for these on the normal package index. Before spending 20–40 minutes on a from-source build (`--no-build-isolation`, with `TORCH_CUDA_ARCH_LIST=9.0` for Hopper), check whether the calling code can use the pure-PyTorch fallback that modern `torch_geometric` ships instead (`torch.scatter_reduce`, `torch_geometric.utils.scatter`, etc.) — often a one-line rewrite avoids the build entirely.

**Watch for package-name traps.** Not every import name matches its PyPI package name, and some PyPI names are already taken by unrelated old packages. For example `pip install hydra` installs a defunct MurmurHash library, not the Hydra config framework you probably want (`hydra-core`). If an install "succeeds" but the import still fails with a strange error (not `ModuleNotFoundError`), double check you installed the right package.

**Clear stale bytecode after fixing a version conflict.** If you reinstall a package to fix a version mismatch and still get the *old* error, check for cached `.pyc` files under `pylibs/__pycache__` — Python may be running stale bytecode from the previous, broken install:

```bash
find $BASE/pylibs -name "__pycache__" -exec rm -rf {} +
```

### 4. `torch.load` and `weights_only`

PyTorch 2.6+ changed `torch.load`'s default from `weights_only=False` to `weights_only=True`. If your code (or a library like `torch_geometric`) loads cached `.pt` files containing custom Python objects (not just tensors), you'll hit:

```
_pickle.UnpicklingError: Weights only load failed. ... Unsupported global: GLOBAL torch_geometric.data.data.DataEdgeAttr ...
```

If the file is your own processed data (not something downloaded from an untrusted source), the fix is simply:

```python
torch.load(path, weights_only=False)
```

Grep your whole project for every `torch.load(` call before you run a long job — this tends to hit once per dataset file, and it's much faster to fix them all at once than one crash at a time:

```bash
grep -rn "torch.load(" /path/to/project --include="*.py"
```

### 5. Verifying before a long batch job

Get an interactive GPU session and run the **actual** script end to end before submitting a many-hour batch job — import errors, missing packages, and path issues show up in seconds; only real training progress tells you the job is actually going to work.

```bash
interactive -A naiss20XX-X-XXXX-gpu -p gpu -N 1 --gpus-per-node=1 -t 02:00:00
source ~/myenv.sh

apptainer exec --nv -B "$BASE:$BASE" -B "$PROJECT:$PROJECT" "$SIF" \
  bash -lc "export PYTHONPATH=$BASE/pylibs:\$PYTHONPATH; cd $PROJECT && python -u main.py"
```

`--nv` is mandatory for GPU access inside the container — without it, `torch.cuda.is_available()` silently returns `False` even on a GPU node. Check GPU utilization from a second SSH session to the same node while it runs:

```bash
nvidia-smi
```

Let it run long enough to see real training/processing output, not just successful imports, before trusting it to a long unattended job.

### Sbatch template — GPU

```bash
#!/bin/bash -l
#SBATCH -A naiss20XX-X-XXXX-gpu
#SBATCH -J namejob
#SBATCH -p gpu
#SBATCH -t 12:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --gpus-per-node=1
#SBATCH -e /nobackup/proj/disk/PROJECTNAME/personal/username/logs/name_%j.e
#SBATCH -o /nobackup/proj/disk/PROJECTNAME/personal/username/logs/name_%j.o
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=you@example.com

set -euo pipefail

BASE=/nobackup/proj/disk/PROJECTNAME/personal/username
PROJECT=$BASE/myproject
SIF=$BASE/containers/pytorch_ngc.sif
OUTPUT=$BASE/output/myrun

mkdir -p "$OUTPUT/tmp" "$BASE/logs"

echo "started : $(date)"
echo "node    : $(hostname) $(uname -m)"

apptainer exec --nv \
  -B "${BASE}:${BASE}" \
  "${SIF}" \
  bash -lc "
    unset PYTHONUSERBASE; export PYTHONNOUSERSITE=1
    export PYTHONPATH=${BASE}/pylibs:\$PYTHONPATH
    export TMPDIR=${OUTPUT}/tmp
    export OUTPUT_DIR_OVERRIDE=${OUTPUT}
    cd ${PROJECT}
    python3 -u main.py
  "

echo "finished: $(date)"
```

Note there's only **one** bind mount (`$BASE`) — as long as your project code, data, and pip-installed libraries all live somewhere under `/nobackup/proj/disk/PROJECTNAME/...`, one bind covers everything and avoids path confusion. Avoid keeping your working code under `/home` — `/home` mounts have occasionally not resolved reliably inside a bound container path; project storage is more predictable.

### Running dynamically (interactive) on GPU

```bash
interactive -A naiss20XX-X-XXXX-gpu -p gpu -N 1 --gpus-per-node=1 -t 02:00:00
source ~/myenv.sh

apptainer shell --nv -B "$BASE:$BASE" "$SIF"
Apptainer> export PYTHONPATH=$BASE/pylibs:$PYTHONPATH
Apptainer> cd $PROJECT
Apptainer> python main.py
```

`apptainer shell` (rather than `exec`) is convenient for iterating quickly — it drops you inside the container so you can retry the script without re-typing the whole `apptainer exec ... bash -lc "..."` incantation each time.

### Keeping containers as single `.sif` files

Don't use `apptainer build --sandbox` on project storage for day-to-day work — a sandbox explodes one `.sif` file into hundreds of thousands of tiny files, which is slow on Lustre and hard on the shared filesystem generally. Sandboxes are fine for short debugging sessions on node-local scratch (`/tmp` on a compute node) if you're actively fixing a build recipe, but the artifact you keep around should always be a single `.sif`.

If you're building a container from a `.def` recipe (rather than just pulling a prebuilt NGC image), build it on the **same architecture partition you intend to run it on** — a GPU-partition container should be built inside a GPU allocation, a CPU-partition one on a CPU node.

## Checking job status

```bash
squeue -u $USER
scontrol show job <job_id>
sacct --jobs=<job_id>
scancel <job_id>
```

## File Transfer

Use `rsync` rather than `scp` where possible — it resumes cleanly if the connection drops (which matters more here than on most systems, given 2FA on every reconnect).

```bash
# Local to Arrhenius
rsync -avP /local/path/ username@login.hpc.arrhenius.naiss.se:/nobackup/proj/disk/PROJECTNAME/personal/username/path/

# Arrhenius to local
rsync -avP username@login.hpc.arrhenius.naiss.se:/nobackup/proj/disk/PROJECTNAME/personal/username/path/ /local/path/
```

If moving data that already exists on another NAISS system (Dardel, Tetralith), check whether a project-storage migration already happened before copying anything through your laptop — routing large transfers between two NAISS centres through a local machine is much slower than a direct centre-to-centre transfer, and duplicate copies waste your storage quota.

## Quick troubleshooting reference

| Symptom | Likely cause |
|---|---|
| `apptainer image is not in an allowed configured path` | A path variable (`$SIF`, `$BASE`) is empty — new shell, forgot to `source ~/myenv.sh` |
| `the image's architecture (arm64) could not run on the host's (amd64)` | Running an aarch64 container from the x86_64 login node — needs a GPU allocation |
| `ModuleNotFoundError` for something you just pip installed | Installed with full dependency resolution instead of `--no-deps`, and something got shadowed/broken — check `PYTHONPATH` ordering and whether numpy/scipy/torch got silently upgraded |
| Same error after reinstalling a package | Stale `__pycache__` — clear it |
| `torch.cuda.is_available()` returns `False` inside a container on a GPU node | Missing `--nv` flag on `apptainer exec`/`shell` |
| `No such file or directory` for a file you can clearly `ls` | Check `realpath`/nested-directory issues (e.g. `rsync` creating `foo/foo/` instead of flattening), or a stale/unset path variable |
| `_pickle.UnpicklingError: Weights only load failed` | PyTorch 2.6+ default change — add `weights_only=False` to the relevant `torch.load()` call if the file is your own trusted data |
| `ResolutionImpossible` from pip | Drop back to `--no-deps` and add missing dependencies by hand, or check for a pinned pre-release version string confusing the resolver |
