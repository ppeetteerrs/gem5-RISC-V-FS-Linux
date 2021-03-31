# gem5 RISC-V Full System Linux Guide

[Setup and Installation (here)](.)

[Running RISC-V FS Linux]("Full System Linux Guide.md")

[Building and Running PARSEC 3.0](PARSEC.md)

## 1. Recommended Environment Setup

### 1.1 Folder Structure
```text
📦~ (or any working directory)
 ┣ 📂gem5
 ┗ 📂riscv
   ┣ 📂bin: RISC-V tool binaries (e.g. GNU-toolchain, QEMU etc.)
   ┣ 📂logs: gem5 simulation logs
   ┣ 📂out: build outputs (kernel / bootloader / devicetree / disk image)
   ┗ 📂src
     ┣ 📂linux: Linux kernel repo
     ┣ 📂pk: RISC-V proxy kernel (bbl bootloader)
     ┣ 📂qemu: QEMU emulator
     ┣ 📂toolchain: RISC-V GNU toolchain
     ┗ 📂ucanlinux: UCanLinux disk image
 ┗ 📂benchmarks
   ┗ 📂parsec: PARSEC 3.0 benchmark for RISC-V
```

### 1.2 Bash Environment
Add these environment variables for easier navigation:
```bash
# Add to ~/.bashrc, replace paths if desired
export RISCV="~/riscv"
export PATH=$RISCV/bin:$PATH
export G5="~/gem5"
export SRC=$RISCV/src
export OUT=$RISCV/out
```

Setup by running:
```bash
mkdir -p $RISCV/bin
mkdir -p $RISCV/out
mkdir -p $RISCV/src
```

## 2 Tools and Libraries

### 2.1 RISC-V GNU Toolchain

The RISC-V GNU Toolchain is used for cross-compiling and debugging RISC-V code.

```bash
sudo apt-get install -y autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
git clone https://github.com/riscv/riscv-gnu-toolchain $SRC/toolchain
cd $SRC/toolchain
./configure --prefix=$RISCV --enable-multilib
make linux -j$(nproc)
# You will need this to compile things like PARSEC and riscv-tests
make newlib -j$(nproc)
```

> If you encounter problems building the RISC-V toolchain (e.g. failure to download riscv-newlib due to proxy), replace `url = git://sourceware.org/git/newlib-cygwin.git` in `.gitmodules` with `url = ../riscv-newlib`

### 2.2 QEMU

QEMU can be used to quickly verify that binaries / disk images / kernels are valid before running / debugging in gem5.

```bash
sudo apt-get install -y git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
git clone https://git.qemu.org/git/qemu.git $SRC/qemu
cd $SRC/qemu
./configure --target-list=riscv64-softmmu --prefix=$RISCV
make -j$(nproc)
make install
```

## 3 gem5 Installation

### 3.1 Dependencies
**Ubuntu 18.04**:
```bash
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python-dev python-six python libboost-all-dev pkg-config python3 python3-dev python3-six
```

**Ubuntu 20.04**:
```bash
sudo apt install -y build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python3-six python-is-python3 libboost-all-dev pkg-config
```

### 3.2 Building gem5 Binaries

Refer to the [official documentation](https://www.gem5.org/documentation/general_docs/building) for more advanced options.

The syntax for building gem5 binaries is **LIKELY TO CHANGE** soon, please refer to the official website (and raise an issue here!) if any problems are encountered.

```bash
git clone https://gem5.googlesource.com/public/gem5 $G5

# Uncomment if you want to contribute to gem5
# cd $G5 && git checkout develop

# Uncomment if you have anaconda3 installed (or non-deault python interpreter)
# export PATH=/usr/bin:$PATH

cd G5 && scons build/RISCV/gem5.opt -j$(nproc)
```

## 4. Linux System
You can download the prebuilt binaries from the [resources](/resources) folder. They should work out of the box. In case you want to build them yourself, follow the instructions below:

### 4.1 Disk Image (UCanLinux)
- Personally, I manually edit the `kernel.config` file to set `CONFIG_DEBUG_KERNEL=y`. I am not sure if that has any effect on the Linux kernel but just in case.
- [UCanLinux](https://github.com/UCanLinux/riscv64-sample) repo has a pre-built disk image. The [disk image]() in this repo contains the PARSEC 3.0 `blackscholes` benchmark, see [PARSEC.md](PARSEC.md) for more info.

```bash
git clone https://github.com/UCanLinux/riscv64-sample.git $SRC/ucanlinux
cp $SRC/ucanlinux/riscv_disk $OUT
```

### 4.2 Linux Kernel v5.10
Feel free to use newer versions of the kernel, but v5.10 has been tested to work.

```bash
git clone https://github.com/torvalds/linux.git $SRC/linux
cd $SRC/linux
git checkout v5.10
cp $SRC/ucanlinux/kernel.config .config
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- olddefconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- all -j$(nproc)
cp vmlinux $OUT
```

### 4.3 Berkeley Bootloader (bbl)
- The Berkeley Bootloader is used to load the Linux kernel, see the [SiFive blog](https://www.sifive.com/blog/all-aboard-part-6-booting-a-risc-v-linux-kernel) for its inner workings.
- bbl expects the hart uid and devicetree address to be stored in registers `a0` and `a1` respectively when loaded. The devicetree generation and register value settings are handled by gem5, see [FULL_SYSTEM.md](FULL_SYSTEM.md) for more info.
```bash
sudo apt-get install -y device-tree-compiler
git clone https://github.com/riscv/riscv-pk.git $SRC/pk
mkdir -p $SRC/pk/build 
cd $SRC/pk/build
../configure --host=riscv64-unknown-linux-gnu --with-payload=$OUT/vmlinux --prefix=$RISCV
CFLAGS="-g" make -j$(nproc)
make install
cp bbl $OUT
```
## 5. Useful Examples
### 5.1 Running QEMU FS Linux
Here are the commands for running QEMU full system for verification:
```bash
# Single core
 

# Multicore
qemu-system-riscv64 -nographic -smp 2 -machine virt -bios none -kernel $OUT/bbl2 -append 'root=/dev/vda ro console=ttyS0' -drive file=$OUT/riscv_disk,format=raw,id=hd0 -device virtio-blk-device,drive=hd0

# Dump DTB for inspection
qemu-system-riscv64 -nographic -machine virt -machine dumpdtb=dtb.dtb
```
Append `-s -S` to the commands to wait for remote GDB connection, then run:

```bash
riscv64-unknown-linux-gnu-gdb $OUT/bbl
# In GDB shell
add-symbol-file ~/riscv/out/vmlinux
target remote :1234
```

### 5.2 Parsing and Dumping Devicetree
```bash
# DTB to DTS
dtc -I dtb -O dts xxx.dtb > xxx.dts

# DTS to DTB
dtc -I dts -O dtb xxx.dts > xxx.dtb
```
- Use `fdtdump` to check dtb header info
- Use `hexcurse` or other hex editors to edit dtb header info (e.g. some libraries might have bugs that require manually changing the `last_comp_version` to 2)

### 5.3 Helper Scripts
There are two scripts in [scripts](./scripts) that can be placed into `$RISCV/bin`. They are examples (templates) of helper scripts to avoid repeatedly typing long commands and settings. Remember to do your `chmod +x`.

- `bg5`: build gem5 binary
- `rg5`: run gem5 RISC-V full system