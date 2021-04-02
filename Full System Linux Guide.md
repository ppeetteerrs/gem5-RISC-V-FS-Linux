# 1 Default Setup
## 1.1 Pre-requisites

You are assumed to have the following correctly built:
- `riscv64-unknown-linux-gnu-` toolchain
- bbl binary at `$OUT/bbl`
  - linux kernel at `$OUT/vmlinux`
- disk image at `$OUT/riscv_disk`
- In gem5: using `configs/example/riscv/fs_linux.py` or replace it with [`setup/fs_linux.py`](./setup/fs_linux.py) for a bit more flexibility (will be merged by next release)

## 1.2 Using pre-built binaries
Please customize the following commands according to your likings (binary type, debug flags, cpu type etc.):

**DTB automatically generated**
```bash
cd $G5
build/RISCV/gem5.opt --debug-flags=Clint -d $RISCV/logs -re configs/example/riscv/fs_linux.py --kernel=$OUT/bbl --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --disk-image=$OUT/riscv_disk -n 1
```

**DTB compiled into BBL**

Precompiled BBL and devicetree for single / double core are given under the `prebuilt/single_cpu` and `prebuilt/dual_cpu` folders. Add the `--bare-metal` flag when DTB generation is not needed.
```bash
cd $G5
build/RISCV/gem5.opt --debug-flags=Clint -d $RISCV/logs -re configs/example/riscv/fs_linux.py --kernel=$OUT/bbl_single --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --disk-image=$OUT/riscv_disk --bare-metal
```

See Section 3 for more alternatives.

## 1.3 Debugging
To debug gem5 code, append `gdb --args` to the above command.

To debug the RISC-V code, append `--param 'system.cpu[0].wait_for_remote_gdb = True'`. Use the following command to access the remote GDB:
```bash
riscv64-unknown-linux-gnu-gdb $OUT/bbl
# In GDB shell
add-symbol-file ~/riscv/out/vmlinux
target remote :7000 # CPU 1 will be at 7001 etc.
```

# 2 Python Configuration

This section goes through the Python configurations which are relevant to RISC-V full system.

## 2.1 HiFive Platform
The HiFive platform is constructed based on SiFive's HiFive board series. By default, it contains the interrupt controllers CLINT and PLIC at addresses `0x2000000` and `0xc000000` respectively. It also has a default UART terminal attached at `0x10000000`.
```python
system.platform = HiFive()
```

## 2.2 CLINT
CLINT is responsible for timer interrupts (through setting MMIO `mtimecmp`) and software interrupts. The `mtime` register in CLINT is incremented by an external signal (interrupt pin). By default, it is connected to a dummy `RiscvRTC` object which generates a fixed-frequency clock signal but provides no RTC MMIO interface for setting kernel wall clock time.
```python
system.platform.rtc = RiscvRTC(frequency=Frequency("10MHz"))
system.platform.clint.int_pin = system.platform.rtc.int_pin
```

See Section 4 for known issues regard CLINT's devicetree. It has **high impace** on the simulation speed.

## 2.3 Disk Image
If you use [`setup/fs_linux.py`](./setup/fs_linux.py), a disk image can be optionally passed in. The disk image is wrapped as a VirtIOMMIO device. Multiple disks can be added if desired, following the format below:

```python
image = CowDiskImage(child=RawDiskImage(read_only=True), read_only=False)
image.child.image_file = mdesc.disks()[0]
system.platform.disk = MmioVirtIO(
    vio=VirtIOBlock(image=image),
    interrupt_id=0x8,
    pio_size=4096,
    pio_addr=0x10008000
)
```

## 2.4 PLIC and Buses
Connect the off-chip devices to the system's `iobus`, connect on-chip devices to `membus`:
```python
system.bridge.ranges = system.platform._off_chip_ranges()
system.platform.attachOnChipIO(system.membus)
system.platform.attachOffChipIO(system.iobus)
```

Route off-chip `PlicIntDevice` (see `gem5/src/dev/riscv/PlicDevice.py`) have their interrupts through PLIC (use `Platform::postPciInt(int line)` to raise interrupt to PLIC). `attachPlic()` register their `interrupt_id` to PLIC and configure the number of interrupt sources in the devicetree:
```python
system.platform.attachPlic()
```

## 2.5 PMAChecker
The PMAChecker adds flags to translated virtual addresses (e.g. uncacheability). Configure the uncacheable memory ranges (or any custom-flag ranges by extending `PMAChecker` class) **after** CPUs have been added to the system:

```python
uncacheable_range = [
    *system.platform._on_chip_ranges(),
    *system.platform._off_chip_ranges()
]

for cpu in system.cpu:
    cpu.mmu.pma_checker = PMAChecker(uncacheable=uncacheable_range)
```

## 2.6 DTB Generation
Call `generateDtb(system)` to generate dtb and embed it into the workload binary (this might be merged into `HiFive` class in the future).

# 3 Command Line Options and Custom Setup

## 3.1 Custom Devicetree
Supply `--dtb-filename=<path_to_dtb_file>` to avoid automatic DTB generation. Configuration in dtb should match the system setup in Python. It might be easier to automatically generate a DTB from Python then make the necessary changes (e.g. new devices / change settings) to get a custom dtb.

## 3.2 Devicetree in Bootloader
If devicetree is already embedded in bootloader binary (bootloader should handling setting the register `a1` to point to devicetree), use `--bare-metal`. For example, this can be done by building bbl with the `--with-dts` option.

## 3.3 No Disk Image
If you use [`setup/fs_linux.py`](./setup/fs_linux.py), disk image is optional.

## 3.4 DioSix Hypervisor (To-Do)
Currently gem5 full system does not support H-mode. But machine-mode hypervisors like Diosix can be booted. However, some bugfixes might be involved.

## 3.5 Checkpointing

Checkpointing and restoration is supported for RISC-V full system (although it doesn't take long to boot from O3CPU). 
### 3.5.1 Taking Checkpoints
You can take checkpoint use the `--take-checkpoint=...,...` option. 

Or you can write C scripts that uses m5 functions (e.g. `m5_checkpoint`) to checkpoint in the linux terminal.
- Build m5 library: `scons riscv.CROSS_COMPILE=riscv64-unknown-linux-gnu- build/riscv/out/m5` in the directory `gem5/util/m5`
- Compile and link using flags: `-static -L${G5}/util/m5/build/riscv/out -lm5`
- Copy binary into riscv disk (see [Benchmark Guide.md](Benchmark%20Guide.md))

### 3.5.2 Restore from Checkpoint
Use the flag `-r <N>` where `<N>` starting from 1 is the index of the checkpoint folder inside your `logs` folder.




Checkpoint restore


## 4 Known Issues
### 4.1 Wrong CPU Timebase
Currently, the DTB generation functionality produces a constant `10MHz` timebase for the CPUs. If the RTC signal frequency is different from `10MHz`, it is necessary to manually edit `gem5/src/dev/riscv/HiFive.py` (see below) and **rebuild gem5**. 

In most cases, however, it is desirable to leave the timebase at `10MHz` with the RTC signal set at `100MHz` to **improve simulation speeds**.

```python
# Under class HiFive
def generateDeviceTree(self, state):
    cpus_node = FdtNode("cpus")
    cpus_node.append(FdtPropertyWords("timebase-frequency", [10000000]))
    yield cpus_node
```

### 4.2 Address Translation Mode
Currently gem5 RISC-V only supports `SV39` address translation, but the generated devicetree specifies `SV48`. Edit `gem5/src/dev/riscv/HiFive.py` (see below) and **rebuild gem5** to correct that.

```python
# Under class HiFive
def annotateCpuDeviceNode(self, cpu, state):
    cpu.append(FdtPropertyStrings('mmu-type', 'riscv,sv48'))
```