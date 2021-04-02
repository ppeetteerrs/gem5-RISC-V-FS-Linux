This section is a tutorial on running benchmarks on gem5 RISC-V FS Linux. We will use PARSEC 3.0 as an example.

# 1 Build m5 Library
```bash
cd $G5/util/m5
scons riscv.CROSS_COMPILE=riscv64-unknown-linux-gnu- build/riscv/out/m5
```

# 2 Add Linker Flags
Add linker flag in compilation script, e.g. PARSEC's `config/gcc-hooks.bldconf`:
```bash
LDFLAGS="${LDFLAGS} -static -L${G5}/util/m5/build/riscv/out -lm5"
```

Note that scripts must be **linked statically**.

# 3 Compile Banchmark
See my [PARSEC Fork](https://github.com/ppeetteerrs/gem5-RISC-V-PARSEC) on how to compile PARSEC 3.0 for gem5 RISC-V.

# 4 Create Disk Image
Create new 500Mb disk:
```bash
cd $OUT
dd if=/dev/zero of=riscv_parsec_disk bs=1M count=500
mkfs.ext2 -L riscv-rootfs riscv_parsec_disk
```

Copy over system files:
```bash
sudo mkdir -p /mnt/old_disk
sudo mkdir -p /mnt/new_disk
sudo mount riscv_disk /mnt/old_disk
sudo mount riscv_parsec_disk /mnt/new_disk
sudo cp -a /mnt/old_disk/* /mnt/new_disk

# Copy benchmark binaries to /mnt/new_disk also

sudo umount /mnt/old_disk
sudo umount /mnt/new_disk
```

Now you can boot using `$OUT/riscv_parsec_disk`