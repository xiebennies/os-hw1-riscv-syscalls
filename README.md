# OS HW1: Custom Linux Kernel & New System Calls (RISC-V)

Cross-compiled Linux v6.1 for RISC-V 64-bit and added two system calls,
tested under QEMU.

- `0001-Add-revstr-system-call-for-RISC-V.patch`: `sys_revstr(char __user *str, size_t n)`
- `0002-Add-tempbuf-system-call-for-RISC-V.patch`: `sys_tempbuf(int mode, void __user *data, size_t size)`
- `config/kernel.config`: kernel config used

## Environment
Docker (Ubuntu 24.04) + riscv64-linux-gnu-gcc 13.3.0, QEMU 8.2.2

## Apply
    git clone --depth 1 --branch v6.1 https://github.com/torvalds/linux
    cd linux && git am /path/to/*.patch
