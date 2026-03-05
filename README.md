# BlackParrot in Docker 
## Create project folder on Desktop (Mac Terminal)
Open Terminal on your Mac (not inside Docker) and run:
```bash
mkdir -p ~/Desktop/bp-work
```
You should now see a folder called ```bp-work``` on your Desktop.
## Create the Ubuntu container
Still in Mac Terminal, run:
```bash
docker run -it \
  --name bp-uvm \
  -v ~/Desktop/bp-work:/workspace \
  ubuntu:22.04
```
### What this does:
- Creates a container named ```bp-uvm```
- Connects Desktop folder to``` /workspace```
- Starts Ubuntu
- Drops you inside container shell
  
You should now see something like:
```bash
root@xxxx:/#
```
That means you're inside Ubuntu.
## Verify that Desktop mapping works
Inside the container, run:

```bash
ls /workspace
```

It should be empty.

Now create a test file:

```bash 
touch /workspace/test.txt
```

Now go look at your Desktop 
— you should see ```test.txt``` inside ```bp-work```.
- If yes → mapping works perfectly.
## Install required packages (inside container)
Now inside container:
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
##  Install latest version of Verilator
Still inside container:
```bash 
cd /workspace
git clone https://github.com/verilator/verilator
cd verilator
git checkout master
autoconf
./configure
make -j2
make install
```
Always use ``` -j2 ```(important if it is an 8GB Mac).
Verify:
```bash
verilator --version
```
If version prints → good.
## Clone BlackParrot
```bash
cd /workspace
git clone https://github.com/black-parrot/black-parrot.git
cd black-parrot
git submodule update --init --recursive
```
Now your Desktop folder will contain:
```
bp-work/
   verilator/
   black-parrot/
```

# Testing 
## Checkout the compatible version
(Example known stable version)
```bash 
git checkout v5.004
```
## Build
```bash 
autoconf
./configure
make -j2
make install
```
Important:
Use  ```-j2``` (safe for 8GB RAM machines).
## Verify
```bash 
verilator --version
```
## Cloning BlackParrot
```bash 
cd /workspace
git clone https://github.com/black-parrot/black-parrot.git
cd black-parrot
git submodule update --init --recursive
```
This downloads:
```
bp_be
bp_fe
basejump_stl
other dependencies
```
## Building BlackParrot (Verilator Flow)
Move to:
```bash 
cd /workspace/black-parrot/bp_top/verilator
```
Available targets:
```bash 
make help
```
then you will see 
```
help
lint.verilator
build.verilator
sim.verilator
wave.verilator
```
## First Build Attempt Issue
Using latest Verilator caused:
```
%Error: Invalid option: --binary
```
**Reason**:

Newer Verilator removed ```--binary```  option used in BlackParrot Makefiles.

**Solution**:

To resolve this issue, Verilator was downgraded to version **5.008**, which is compatible with the current BlackParrot build flow.

## Enabling Trace Support
By default:
```
TRACE ?= 0
```
Tracing is disabled.
Without trace:

No ```.vcd```

No ```.fst```

```wave.verilator``` fails

## Correct Build With Tracing Enabled
### Clean
```bash 
make clean
```
If needed:
```bash 
rm -rf obj_dir results
```
### Build with Trace
```bash 
make build.verilator TRACE=1 -j2
```
This enables:```--trace-fst``` internally.

### Run Simulation
```bash 
make sim.verilator TRACE=1
```
Successful output:
```bash
[BSG-PASS]
Verilog $finish
Trace confirmation:
BSG-INFO: TOP.testbench.tracer flushing trace: dump.fst
```

### For viewing the waveform 
#### Find the path of the ```.fst``` file
```bash
find results -name "*.fst"
```
#### Exit the container
```bash
exit
```
Now you are in the host terminal 
### Copying the the ```.fst ``` file into the host 
```bash
docker cp bp-uvm:/workspace/black-parrot/bp_top/verilator/results/bp_tethered.e_bp_default_cfg/demo/dump.fst ~/Desktop/
```
the ```.fst``` file  is stored in desktop 

### Run on GTKWave

Initially check if  GTKWave is installed

```bash
gtkwave --version
```
if not installed 
```bash
brew install gtkwave
```

Run this command to see the waveform
```bash
gtkwave ~/Desktop/dump.fst
```

# Additional
## How to exit container later
```bash
exit
```

## Later you re-enter the container  with:
```bash 
docker start bp-uvm
docker exec -it bp-uvm bash
```

or
```bash
docker start -ai bp-uvm
```
