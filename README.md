# gem5 RISC-V FS Linux Setup Guide

## 1. Shell Environment
First, add these lines to your `~/.<shell>rc` file:
```bash
export RISCV=<riscv_working_dir>
export PATH=$RISCV/bin:$PATH
export G5=<gem5_repo_path>
export SRC=$RISCV/src
export OUT=$RISCV/out
```

Then, create the directories:
```bash
mkdir -p $RISCV/bin
mkdir -p $RISCV/out
mkdir -p $RISCV/src
```

Clone this repo for pre-built binaries and images and devicetree (or you can build them from scratch).
```bash
git clone https://github.com/ppeetteerrs/gem5-RISC-V-FS-Linux.git $OUT
```

## 2. RISC-V Libraries

### 2.1 RISC-V GNU Toolchain
```bash
sudo apt-get install -y autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
cd $SRC
git clone https://github.com/riscv/riscv-gnu-toolchain toolchain
cd toolchain
./configure --prefix=$RISCV
make linux -j$(nproc)
```

### 2.2 QEMU
```bash
sudo apt-get install -y git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
cd $SRC
git clone https://git.qemu.org/git/qemu.git qemu
cd qemu
./configure --target-list=riscv64-softmmu --prefix=$RISCV
make -j$(nproc)
make install
```

## 3. gem5
```bash
# Ubuntu 18.04
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python-dev python-six python libboost-all-dev pkg-config

# Ubuntu 20.04
sudo apt install -y build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python3-six python-is-python3 libboost-all-dev pkg-config

git clone https://gem5.googlesource.com/public/gem5 $G5
cd $G5

# prepend PATH to use default Python3 (if anaconda is installed)
export PATH=/usr/bin:$PATH && scons build/RISCV/gem5.opt -j$(nproc)
```

## 4. Full System Linux

### 4.1 Disk Image (UCanLinux)
Personally, I manually edit the kernel.config file to set `CONFIG_DEBUG_KERNEL=y`. I am not sure if that has any effect on the RISC-V Linux Kernel but just in case. 

We are using the pre-built disk image here. Please check the repo if you intend to build it from scratch. (There is no such need though, as we can edit the disk content by mounting it)
```bash
cd $SRC
git clone https://github.com/UCanLinux/riscv64-sample.git ucanlinux
cp ucanlinux/riscv_disk $OUT
```

### 4.2 Linux Kernel v5.10
```bash
cd $SRC
git clone https://github.com/torvalds/linux.git linux
cd linux
git checkout v5.10
cp $SRC/ucanlinux/kernel.config .config
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- olddefconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- all -j$(nproc)
cp vmlinux $OUT
```


### 4.3 Berkeley Bootloader
```bash
sudo apt-get install -y device-tree-compiler
cd $SRC
git clone https://github.com/riscv/riscv-pk.git pk
mkdir -p pk/build 
cd pk/build
../configure --host=riscv64-unknown-linux-gnu --with-payload=$OUT/vmlinux --prefix=$RISCV --with-dts=$OUT/device.dts
CFLAGS="-g" make -j$(nproc)
make install
cp bbl $OUT
```

## 5. Things to Note
### 5.1 Running QEMU FS Linux
```bash
# Single core
qemu-system-riscv64 -nographic -machine virt -bios none -kernel $OUT/bbl -append 'root=/dev/vda ro console=ttyS0' -drive file=$OUT/riscv_disk,format=raw,id=hd0 -device virtio-blk-device,drive=hd0

# Multicore
qemu-system-riscv64 -nographic -smp 2 -machine virt -bios none -kernel $OUT/bbl2 -append 'root=/dev/vda ro console=ttyS0' -drive file=$OUT/riscv_disk,format=raw,id=hd0 -device virtio-blk-device,drive=hd0
```

### 5.2 Running gem5 FS Linux
```bash
# Single core
cd $G5
build/RISCV/gem5.opt --debug-flags= -d logs -re configs/example/fs.py --kernel=$OUT/bbl --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --disk-image=$OUT/riscv_disk

# Multicore
cd $G5
build/RISCV/gem5.opt --debug-flags= -d logs -re configs/example/fs.py --kernel=$OUT/bbl2 --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --disk-image=$OUT/riscv_disk --num-cpus=2

# Single core with remote GDB
build/RISCV/gem5.opt --debug-flags=RiscvClint -d logs -re configs/example/fs.py --kernel=$OUT/bbl --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --param 'system.cpu[0].wait_for_remote_gdb = True' --disk-image=$OUT/riscv_disk
```

### 5.3 gem5 FS Python Configuration

This is the RISC-V linux setup in `FSConfig.py`. The rest of the setup are shared with other ISAs in `fs.py`.

Note that all addresses and interrupt source IDs must match that of `device.dts`. Currently, RISC-V Full System does not support generating the DTB automatically from Python.

```python
def makeLinuxRiscvSystem(mem_mode, mdesc=None, cmdline=None):
    self = System()
    if not mdesc:
        mdesc = SysConfig()
    self.mem_mode = mem_mode
    self.mem_ranges = [AddrRange(start=0x80000000, size=mdesc.mem())]

    self.workload = RiscvBareMetal()

    self.iobus = IOXBar()
    self.membus = MemBus()

    self.system_port = self.membus.slave

    self.intrctrl = IntrControl()

    # HiFive platform
    self.platform = HiFive()

    # CLNT
    self.platform.clint = Clint()
    # CLINT RTC Signal Frequency (will be decoupled in future release)
    self.platform.clint.frequency = Frequency("100MHz") 
    self.platform.clint.pio = self.membus.master

    # PLIC
    self.platform.plic = Plic()
    self.platform.clint.pio_addr = 0x2000000
    self.platform.plic.pio_addr = 0xc000000
    # 11 sources because UART is pin 0xa
    self.platform.plic.n_src = 11
    self.platform.plic.pio = self.membus.master

    # UART
    self.uart = Uart8250(pio_addr=0x10000000)
    self.terminal = Terminal()
    self.platform.uart_int_id = 0xa
    self.uart.pio = self.iobus.master

    # VirtIOMMIO
    image = CowDiskImage(child=RawDiskImage(read_only=True), read_only=False)
    image.child.image_file = mdesc.disks()[0]
    self.platform.disk = MmioVirtIO(
        vio=VirtIOBlock(image=image),
        interrupt_id=0x8,
        pio_size = 4096
    )
    self.platform.disk.pio_addr = 0x10008000
    self.platform.disk.pio = self.iobus.master

    # PMA
    self.pma = PMA()
    self.pma.uncacheable = [
        AddrRange(0x10000000, 0x10000008),
        AddrRange(0x10008000, 0x10009000),
        AddrRange(0xc000000, 0xc210000),
        AddrRange(0x2000000, 0x2010000)
    ]

    self.bridge = Bridge(delay='50ns')
    self.bridge.master = self.iobus.slave
    self.bridge.slave = self.membus.master
    self.bridge.ranges = [
        AddrRange(0x10000000, 0x10000080),
        AddrRange(0x10008000, 0x10009000)
    ]

    return self
```

### 5.4 Future Improvements
- Decoupling RTC from CLINT
- DTB generation from Python (maybe)
