# OS HW1: Custom Linux Kernel & New System Calls (RISC-V)

Cross-compiled Linux v6.1 for RISC-V 64-bit, added two system calls (`revstr`, `tempbuf`), and tested them under QEMU.

## Environment
| Item | Value |
|---|---|
| Host | MacBook Pro M4 |
| Container OS | Ubuntu 24.04 (Docker) |
| Kernel | Linux v6.1 |
| Target | RISC-V 64-bit |
| Cross compiler | riscv64-linux-gnu-gcc 13.3.0 |
| Emulator | QEMU 8.2.2 (`virt` machine) |

Build happens in the container; the syscalls are exercised in QEMU with a custom initramfs.

## Contents
- `0001-Add-revstr-system-call-for-RISC-V.patch`: `sys_revstr` (#451)
- `0002-Add-tempbuf-system-call-for-RISC-V.patch`: `sys_tempbuf` (#452)
- `config/kernel.config`: kernel config used

## Integration
Per syscall: define the entry (`SYSCALL_DEFINE2` / `SYSCALL_DEFINE3`), register the number in `include/uapi/asm-generic/unistd.h` (and bump `__NR_syscalls` to 453), declare it in `include/linux/syscalls.h`, and add the object to `kernel/Makefile`. Both are built into the kernel (no Kconfig option).

## `revstr(char __user *str, size_t n)` (#451)
Reverses the first `n` bytes (excluding the NUL) of a user string in place.
1. Allocate `n + 1` bytes (checks `n + 1` for overflow), `copy_from_user()` the first `n` bytes, NUL-terminate.
2. Log the original string, swap symmetric characters (`n / 2` swaps), log the reversed string.
3. `copy_to_user()` the first `n` bytes back, free the buffer, return 0.

Errors: `-ENOMEM` on allocation failure, `-EFAULT` on copy failure (buffer freed first to avoid a leak).

## `tempbuf(int mode, void __user *data, size_t size)` (#452)
Keeps a kernel-side list of strings that persists across syscalls (until removed or reboot).

| Mode | Value | `data` / `size` |
|---|---|---|
| PRINT | 0 | output buffer / capacity |
| ADD | 1 | string to add / length |
| REMOVE | 2 | string to search / length |

- Each node holds a string pointer, its length, and a `list_head`; string and node are allocated separately.
- **ADD**: allocate and copy first, then take the mutex and `list_add_tail()` (keeps insertion order).
- **REMOVE**: copy the key, lock, walk the list comparing length then `memcmp()`; unlink and free the first match, `-ENOENT` if none.
- **PRINT**: under the mutex, two passes (compute total length, then copy entries separated by one space) into a kernel buffer, unlock, then copy to user space. At most `size - 1` characters are written; returns the number of characters copied (excluding NUL). An empty list writes an empty string and returns 0.
- All three operations share one mutex.

Errors: `-EFAULT` (NULL pointer, zero capacity, copy failure), `-ENOENT`, `-ENOMEM`, `-EINVAL` (invalid mode), `-EOVERFLOW` (length computation overflow).

## Testing
Test programs are cross-compiled with `-static` and packed into an initramfs (`cpio -H newc | gzip`); QEMU boots the custom kernel with it. Each tempbuf test starts from a fresh boot so leftover list entries cannot affect results.

Checked with `assert()`:
- `revstr`: empty string, single char, odd/even lengths, strings with spaces
- `tempbuf`: ADD A, B, A then REMOVE A leaves only the first A removed (`"B A"`); PRINT return length; empty-list PRINT; 4-byte and 1-byte buffers (truncation, NUL, length); out-of-bounds guard bytes unchanged; NULL / zero capacity returns `EFAULT`; removing a missing entry returns `ENOENT`

Not covered: unmapped non-NULL user pointers, concurrent use from multiple processes, and allocation-failure paths were not exercised at runtime.

## Patch verification
Each syscall was committed separately and exported with `git format-patch`. Applying both patches to a clean v6.1 checkout gave no errors and no diff against the finished tree. The kernel was not fully rebuilt in that verification copy.

## Lessons learned
- Putting the source on the macOS shared folder broke the build: filenames that differ only in case collapse into one file. Moving the source into a Docker volume mounted at `/work` fixed it.
- Do allocation and `copy_from_user()` before taking the lock where possible, and copy to user space after releasing it, so the mutex is held only while touching the list.

## Apply
    git clone --depth 1 --branch v6.1 https://github.com/torvalds/linux
    cd linux && git am /path/to/*.patch
