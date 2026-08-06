# FBM Two-Stage Independent Fission-Yield Training Manual

This manual describes the environment setup, download and compilation of
Radford M. Neal's Flexible Bayesian Modelling (FBM) software, and the two-stage
training commands used for the independent fission-yield model.



## Contents

- [1. Environment setup](#1-environment-setup)
- [2. Download FBM](#2-download-fbm)
- [3. Compile FBM](#3-compile-fbm)
- [4. Training-data layout](#4-training-data-layout)
- [5. Stage-one training commands](#5-stage-one-training-commands)
- [6. Stage-two training commands](#6-stage-two-training-commands)
- [7. FBM command reference](#7-fbm-command-reference)
- [8. Example mass-yield data](#8-example-mass-yield-data)

## 1. Environment setup

FBM is intended for Unix-like systems. Use Linux or Windows Subsystem for Linux
2 (WSL 2). A CPU-only build can also be used on macOS.

### 1.1 WSL 2 on Windows

Open PowerShell as Administrator:

```powershell
wsl --install
wsl --update
```

Restart Windows if requested, open Ubuntu, and install the build tools:

```bash
sudo apt update
sudo apt install -y build-essential git python3
```

For GPU sampling, install an NVIDIA driver and CUDA Toolkit combination that
supports CUDA inside WSL or Linux. Verify the installation with:

```bash
nvidia-smi
nvcc --version
```

The training commands do not require a particular Slurm account, partition,
module system, or cluster directory.

## 2. Download FBM

This workflow is intended for an FBM release from approximately 2022-2023. The
stable tag `fbm.2022-04-21` is a suitable reference, while nearby revisions are
also expected to work.

```bash
git clone https://gitlab.com/radfordneal/fbm.git fbm-paper
cd fbm-paper
git checkout fbm.2022-04-21
```

To use another nearby revision, omit the `git checkout` command and record the
output of `git describe --tags --always` with the run.

## 3. Compile FBM

Run one of the following commands from the FBM repository root.

### 3.1 CPU build

```bash
./make-all dbl
```

### 3.2 CPU and GPU build

```bash
./make-all dbl gpu
```

Add FBM to the current shell's `PATH` and perform a basic check:

```bash
export FBM_ROOT="$PWD"
export PATH="$FBM_ROOT/bin-fbm:$PATH"

command -v using
command -v net-spec
command -v net-mc

test -x "$FBM_ROOT/bin-dbl/net-mc" && echo "FBM CPU build found"
test -e "$FBM_ROOT/bin-dbl-gpu/net-mc" && echo "FBM GPU build found"
```

An optional smoke test creates and displays a minimal network specification:

```bash
FBM_TEST_DIR="$(mktemp -d)"
using dbl net-spec "$FBM_TEST_DIR/test.net" 1 2 1 / \
  ih=0.1 bh=0.1 ho=0.1 bo=1
using dbl net-spec "$FBM_TEST_DIR/test.net"
```

If the second command prints the 1-2-1 network specification without an error,
the CPU installation is usable. The training commands below use `using dbl` for
CPU execution and `using dbl-gpu` for GPU execution.

## 4. Training-data layout

The recorded `data-spec` commands use three inputs and one target. Prepared
training files must be headerless numeric files with at least four columns:

| Column | Value used by FBM | 
|---:|---|
| 1 | scaled proton number |
| 2 | scaled mass number |
| 3 | scaled incident-neutron energy |
| 4 | scaled independent-yield target |
| 5, scared rror | Can be included by modifying FBM |

In our case, we modified the fbm to include the error.Without taking into account the uncertainties of GEF model, BNN can lead to narrow distributions of priors for the second stage.
Run the commands from a directory containing
`gef_u235_independent_yields.csv`, `jendl_u235_training_yields.csv`, and
`jendl_u235_test_yields.csv`, or replace these names with the corresponding
file names. The range of the training data and test data can be subsituted by the true range of the dataset. Both dataset must use the same scaling.
## 5. Stage-one training commands

The original stage-one network shape shoule be the same as the stage-one network shape. We use 3-22-22-1 in this work. The network log is named `stage1_gef.net` consistently.

```bash
net-spec stage1_gef.net 3 22 22 1 / \
  ih=0.05:0.5 \
  bh=0.01:0.5 \
  hh=0.05:0.25 \
  bh1=0.05:0.5 \
  ho=x0.05:0.5 \
  bo=10

model-spec stage1_gef.net real 0.05:0.5

data-spec stage1_gef.net 3 1 / \
  gef_u235_independent_yields.csv@1:9333 . \
  gef_u235_independent_yields.csv@9334: .

net-gen stage1_gef.net fix 0.5

mc-spec stage1_gef.net \
  repeat 2000 sample-noise heatbath hybrid 100:10 0.5

using dbl net-mc stage1_gef.net 1

mc-spec stage1_gef.net \
  sample-sigmas heatbath hybrid 2000:10 0.2

using dbl-gpu net-mc stage1_gef.net 100000
```

For a CPU-only run, replace the last command with:

```bash
using dbl net-mc stage1_gef.net 100000
```

## 6. Stage-two training commands

The stage-two log is named `stage2_independent.net`. It uses the same
3-22-22-1 topology as stage one.
```bash
net-spec stage2_independent.net 3 22 22 1 / \
  ih=0.05:0.5 \
  bh=0.01:0.5 \
  hh=0.05:0.25 \
  bh1=0.05:0.5 \
  ho=x0.05:0.5 \
  bo=10

model-spec stage2_independent.net real 0.05:0.5

data-spec stage2_independent.net 3 1 / \
  jendl_u235_training_yields.csv@1:4135 . \
  jendl_u235_test_yields.csv@11982: .

log-append \
  stage1_gef.net 99000:100000 \
  stage2_independent.net 0

mc-spec stage2_independent.net \
  repeat 2000 sample-noise heatbath hybrid 100:10 0.5

using dbl net-mc stage2_independent.net 1001

mc-spec stage2_independent.net \
  sample-sigmas heatbath hybrid 2000:10 0.2

using dbl-gpu net-mc stage2_independent.net 5000
```

For a CPU-only run, replace the last command with:

```bash
using dbl net-mc stage2_independent.net 5000
```

For the detailed meaning of all commands and parameters, see `doc/index.html`
in Neal's FBM distribution or the
[official FBM repository](https://gitlab.com/radfordneal/fbm).

## 7. Example  data

Two mass-yield examples for thermal-neutron-induced fission of U-235 are
also available

| Source | File | Mass range | Rows | Sum of mass yields |
|---|---|---:|---:|---:|
| JENDL | [`jendl_u235_thermal_mass_yield.csv`](open_source_release/examples/data/jendl_u235_thermal_mass_yield.csv) | 66-172 | 107 | 200.000000751% |
| GEF | [`gef_u235_thermal_mass_yield.csv`](open_source_release/examples/data/gef_u235_thermal_mass_yield.csv) | 64-169 | 106 | 199.999570302% |


These data can be employed to test the whether the workflow works in your computer
