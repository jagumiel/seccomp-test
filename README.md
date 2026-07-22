# Seccomp-BPF Learning Lab

> A progressive C laboratory for understanding Linux system-call filtering with seccomp and classic BPF.

[![Language: C](https://img.shields.io/badge/language-C-00599C?style=flat-square&logo=c&logoColor=white)](https://www.c-language.org/)
[![Platform: Linux](https://img.shields.io/badge/platform-Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](https://www.kernel.org/)
[![Security: seccomp--BPF](https://img.shields.io/badge/security-seccomp--BPF-4C1?style=flat-square)](https://docs.kernel.org/userspace-api/seccomp_filter.html)
[![Status: Educational](https://img.shields.io/badge/status-educational-blue?style=flat-square)](#project-status)

This repository demonstrates how a Linux program can progressively reduce the kernel attack surface available to it. The same small `fork()` example is developed through four stages: first without confinement, then with a syscall allowlist, next with a diagnostic `SIGSYS` reporter, and finally with a deliberately expanded policy.

The project uses the low-level kernel interface directly: a classic BPF program examines each system call, validates the calling architecture and returns an allow, trap or terminate action. It is intended as a learning lab for Linux security, systems programming and application hardening.

> [!IMPORTANT]
> seccomp filtering is not a complete sandbox. It restricts the system calls visible to a process, but it does not independently control files, identities, information flow or every other system resource. Combine it with other isolation and hardening mechanisms for real security boundaries.

## What this project demonstrates

- Installing a seccomp filter with `prctl()`.
- Setting `PR_SET_NO_NEW_PRIVS` before enabling an unprivileged filter.
- Validating the syscall ABI before checking syscall numbers.
- Building an allowlist with classic BPF instructions.
- Observing the difference between `SECCOMP_RET_KILL` and `SECCOMP_RET_TRAP`.
- Handling `SIGSYS` during policy development to identify a blocked syscall.
- Showing that a filter installed before `fork()` also constrains the child.
- Iteratively refining a policy from observed application behaviour.

## Learning path

Each directory is a complete snapshot of the example at one point in the learning process. Work through them in order; the differences between consecutive steps are the core of the exercise.

| Step | Directory | Change introduced | Expected lesson |
| --- | --- | --- | --- |
| 1 | [`paso1/`](paso1/) | Baseline program using `fork()`, `sleep()` and `wait()` | Observe the program before applying any syscall restrictions. |
| 2 | [`paso2/`](paso2/) | Minimal seccomp allowlist with a terminate-by-default action | The process is stopped as soon as it requests a syscall not present in the policy. |
| 3 | [`paso3/`](paso3/) | Development-only `SIGSYS` reporter and trap action | Identify the first blocked syscall; in the reference environment it is `clone()`. |
| 4 | [`paso4/`](paso4/) | Expanded allowlist including the calls needed to create and run the child | The child can complete, while the parent is still stopped when it reaches a non-allowed wait operation. |

The progression can be summarized as:

```text
baseline program
      -> deny by default
      -> report the missing syscall
      -> review and deliberately extend the allowlist
```

This is the main security idea behind the laboratory: an observed syscall should not be allowed automatically. It should first be understood and then either permitted because the application genuinely needs it or removed by changing the program design.

## How the filter works

The filter is installed before the call to `fork()`:

```c
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

Its BPF program follows four decisions:

1. Read and validate the architecture stored in `struct seccomp_data`.
2. Read the system-call number.
3. Return `SECCOMP_RET_ALLOW` for calls explicitly included in the allowlist.
4. Apply the default action to every other call.

Steps 2 and 4 terminate on a disallowed syscall. Steps 3 and 4 include the diagnostic reporter, which changes the default action to `SECCOMP_RET_TRAP`, handles `SIGSYS` and prints the triggering syscall before exiting.

> [!WARNING]
> The reporter is intentionally a development aid. Its own header emits a compiler warning stating that it must not be used in production.

## Repository structure

```text
.
├── paso1/                         # Unfiltered fork example
├── paso2/                         # Initial seccomp allowlist
├── paso3/                         # Syscall reporting during policy development
├── paso4/                         # Extended allowlist and partial execution
├── Instructions.txt               # Original short build instructions
├── SSC-Desarrollo_y_Demostracion-JA_Gumiel.pdf
│                                    # Full project report in Spanish
└── README.md
```

The complete rationale, original environment, source changes and execution evidence are documented in:

**[SCC - Seccomp: Cómo evitar las llamadas al sistema](SSC-Desarrollo_y_Demostracion-JA_Gumiel.pdf)** (Spanish, PDF)

## Requirements

- Linux with seccomp filter support.
- An x86 or x86-64 development environment; see [Compatibility](#compatibility).
- GCC or another compatible C compiler.
- GNU Make.
- GNU Autoconf and Autoheader.

On Debian or Ubuntu:

```bash
sudo apt update
sudo apt install build-essential autoconf
```

You can inspect whether the current process exposes seccomp state with:

```bash
grep -E '^(NoNewPrivs|Seccomp|Seccomp_filters):' /proc/self/status
```

Kernel configuration files, when available, can also be checked for `CONFIG_SECCOMP` and `CONFIG_SECCOMP_FILTER`.

## Build and run

Choose a step and build it inside its directory. For example:

```bash
cd paso1
autoconf
autoheader
sh ./configure
make
./procFork
```

Repeat the same procedure in `paso2`, `paso3` or `paso4`.

### Expected observations

#### Step 1 - baseline

Both processes complete normally:

```text
Hijo en ejecucion. Espere.
Hijo terminado, saliendo..
Proceso padre a la espera...
Padre terminado...
```

#### Step 2 - blocked syscall

The minimal allowlist does not permit process creation. The kernel stops the program, commonly reported by the shell as:

```text
Bad system call
```

#### Step 3 - policy discovery

The trap-based reporter identifies the first missing call in the original environment:

```text
Looks like you also need syscall: clone(...)
```

The exact syscall number and even the syscall name may differ across architectures, C libraries and kernel versions.

#### Step 4 - deliberately expanded policy

In the original 32-bit Debian 9 environment, the child is allowed to run and finish. The parent then reaches a wait-related operation that is not in the allowlist and is stopped by the filter. This demonstrates that allowing one blocked syscall often reveals the next dependency and that policy construction is iterative.

## Compatibility

The original experiment was performed on **32-bit Debian 9** in a virtual machine, and the executables currently stored in the repository are i386 binaries. The included helper header explicitly targets the i386 and x86-64 ABIs, but the syscall policy in Step 4 retains 32-bit-specific names such as `fstat64`.

Consequently:

- Steps 1-3 still demonstrate the intended progression on a typical x86-64 Linux system.
- Step 4 does not compile unchanged on modern x86-64 systems where `__NR_fstat64` is not defined.
- Newer C libraries may use different calls, such as `clone3`, and vDSO behaviour can also alter what is observed.
- Syscall numbers and calling conventions are architecture-specific; a policy copied from one ABI must not be assumed safe or correct on another.

For a faithful reproduction of the report, use a compatible 32-bit Debian environment. Porting the final step to current x86-64 Linux is intentionally left as future work rather than silently broadening the policy.

## Security notes

- Prefer an **allowlist**. A denylist can miss new syscalls or alternative ways to perform the same operation.
- Always validate the architecture before interpreting a syscall number.
- Install the narrowest policy that supports the intended application behaviour.
- Treat the syscall reporter as a diagnostic tool, not a production enforcement mechanism.
- Do not assume that an unknown program is safe to execute merely because a small seccomp example has been added around it.
- Test every policy against the exact architecture, C library, kernel and application build that will run in production.

## Project status

This repository preserves an educational seccomp-BPF experiment and its original evidence. It is not currently a portable sandboxing library or a production-ready policy generator.

The code was reviewed on a current x86-64 Linux environment with these results:

| Step | Build result | Runtime result |
| --- | --- | --- |
| 1 | Successful | Parent and child complete. |
| 2 | Successful | Process terminates with `SIGSYS` as intended. |
| 3 | Successful | Reporter identifies `clone` as the blocked syscall. |
| 4 | Requires porting | Compilation stops at the 32-bit-specific `fstat64` policy entry. |

These results are evidence of the original architecture dependency, not a CI guarantee. No automated workflow is currently configured.

## Source provenance and licensing

The `seccomp-bpf.h` and syscall-reporter sources identify Will Drewry, Kees Cook and The Chromium OS Authors in their file headers and state that they are governed by a BSD-style license. The original learning material is referenced below.

The repository does not currently declare a repository-wide license. Until licensing and third-party notices are added explicitly, review the individual source headers before reusing the code.

## References

- [Linux kernel documentation: Seccomp BPF](https://docs.kernel.org/userspace-api/seccomp_filter.html)
- [Linux manual page: `seccomp(2)`](https://man7.org/linux/man-pages/man2/seccomp.2.html)
- [Kees Cook: Using simple seccomp filters](https://outflux.net/teach-seccomp/)
- [GNU Autoconf manual](https://www.gnu.org/software/autoconf/manual/autoconf.html)

## Author

Developed and documented by **Jose Ángel Gumiel**.

- [GitHub](https://github.com/jagumiel)
- [Technical blog](https://jagumiel.xyz/blog/)
