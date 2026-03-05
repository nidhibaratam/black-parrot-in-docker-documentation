# Verilator Docker Setup (macOS)

This document describes a reproducible Docker-based setup for building and running BlackParrot using Verilator on macOS.

The flow pins Verilator to v5.008 to maintain compatibility with the current BlackParrot Makefiles.

---

## Tested Environment

- Host: macOS (Apple Silicon M1)
- Docker: Default architecture (linux/arm64)
- Container: Ubuntu 22.04
- Verilator: v5.008

---

## Host Requirements

A host directory is mounted into the container to persist build artifacts.

```bash
mkdir -p ~/bp-work
```
---

## Create Docker Container
```bash
docker run -it \
  --name bp-verilator \
  -v ~/bp-work:/workspace \
  ubuntu:22.04
```
This launches an interactive Ubuntu 22.04 container.

### Verify Volume Mapping (Important)
Inside the container:
```bash 
ls /workspace
touch /workspace/test.txt
```
Check ```~/bp-work``` on macOS.
If ```test.txt```  exists, the volume mapping is correct.

---

## Install Dependencies

Inside the container:
```bash
apt update
apt install -y \
  git build-essential \
  autoconf automake libtool \
  python3 python3-pip \
  cmake ninja-build \
  bison flex \
  libgmp-dev libmpc-dev libmpfr-dev \
  zlib1g-dev \
  device-tree-compiler \
  curl
```
---

## Install Verilator (v5.008)
```bash
cd /workspace
git clone https://github.com/verilator/verilator
cd verilator
git checkout v5.008
autoconf
./configure
make -j2
make install
```
```-j2``` is recommended for machines with 8GB RAM to avoid out-of-memory errors during compilation.

Verify installation:
```bash
verilator --version
```

Expected:
```
Verilator 5.008
```
Newer Verilator releases remove the --binary option used by the current BlackParrot Verilator Makefiles. Verilator v5.008 is verified to be compatible with the current BlackParrot Verilator flow.

If a newer Verilator version is used, the build fails with:
```
%Error: Invalid option: --binary
```

In that case, reinstall Verilator and ensure that version v5.008 is checked out before building.

---

## Clone BlackParrot
```bash
cd /workspace
git clone https://github.com/black-parrot/black-parrot.git
cd black-parrot
git submodule update --init --recursive
```
Workspace layout should look like:
```
bp-work/
  verilator/
  black-parrot/
```
  
---

## Running Verilator Simulation

Navigate to the Verilator flow:
```bash
cd /workspace/black-parrot/bp_top/verilator
```

Available targets:
```bash
make help
```

Common targets:
```
- help 
- lint.verilator
- build.verilator
- sim.verilator
- wave.verilator
```

---

## Build and Simulate

Clean previous artifacts:
```bash
make clean
rm -rf obj_dir results
```

Build with waveform support:

```bash
make build.verilator TRACE=1 -j2
```

- TRACE=1 enables --trace-fst in Verilator and generates waveform dump files.
- Do not increase parallelism beyond available RAM (e.g., avoid -j8 on 8GB systems).

Run simulation:
```bash
make sim.verilator TRACE=1
```

---

## Expected Output

A successful run includes:
```
Hello World!
[CORE FSH] PASS
[BSG-PASS]
Verilog $finish
```

Presence of [BSG-PASS] indicates successful simulation.


If ```TRACE=1``` is enabled, an FST waveform file is generated under:
```bash
results/<test>/<config>/
```
Inside the container:
```bash
find results -name "*.fst"
```
Example output:
```
results/bp_tethered.e_bp_default_cfg/demo/dump.fst
```
---

## Viewing Waveforms (Host)

Copy the FST file from the container:
```bash
docker cp \
bp-verilator:/workspace/black-parrot/bp_top/verilator/results/bp_tethered.e_bp_default_cfg/demo/dump.fst \
~/bp-work/
```

Open with GTKWave:
```bash
brew install gtkwave # If gtkwave is not installed
gtkwave ~/bp-work/dump.fst
```
---

## Re-entering the Container
```bash
docker start bp-verilator
docker exec -it bp-verilator bash
```

---

## Notes

- TRACE is disabled by default (TRACE ?= 0).
- Use make clean if CONFIG, FLAGS, or PARAMS are changed.
- The mounted host directory preserves build artifacts across container restarts.
- The flow has been validated with Verilator v5.008.


---

Validated against BlackParrot `master` as of March 2026.
