# 1 Default Setup
## 1.1 Pre-requisites

You are assumed to have the following correctly built:
- `riscv64-unknown-linux-gnu-` toolchain
- bbl binary at `$OUT/bbl`
  - linux kernel at `$OUT/vmlinux`
- disk image at `$OUT/riscv_disk`
- In gem5: using `configs/example/riscv/fs_linux.py` or replace it with [`resources/fs_linux.py`](./resources/fs_linux.py) for a bit more flexibility

## 1.2 Running
Please customize the following command according to your likings (binary type, debug flags, cpu type etc.):
```bash
cd $G5
build/RISCV/gem5.opt --debug-flags=Clint -d $RISCV/logs -re configs/example/riscv/fs_linux.py --kernel=$OUT/bbl --caches --mem-size=256MB --mem-type=DDR4_2400_8x8 --cpu-type=AtomicSimpleCPU --disk-image=$OUT/riscv_disk
```

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

# 3 Other Setups
## 3.1 Custom Devicetree

## 3.2 Devicetree in Bootloader

## 3.3 No Disk Image

## 3.4 DioSix Hypervisor (To-Do)
