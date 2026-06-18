# Reverse Engineering Writeups

A collection of my reverse-engineering and binary-analysis work — from CTF-style CrackMe challenges to a full hardware-level security audit of iOS Activation Lock. Each writeup documents the methodology, tooling, and findings, including the cases where a system's defenses held and *why* they held.

These are documentation of my own analysis, carried out on devices and binaries I own or that are published as legal practice challenges, for learning and demonstration.

---

## Contents

### iOS Security Architecture & Reverse Engineering Audit
**[`iOS_Reverse_Engineering_Report.md`](./iOS_Reverse_Engineering_Report.md)**

A deep-dive audit of the iOS Activation Lock security model on an iPhone 6 (iOS 12.5.8, A8 chip). Starting from a `checkm8` BootROM exploit, I systematically attempted every local methodology to bypass activation — runtime process termination, Mach-O binary patching, USB tunneling, pairing-record injection, and certificate spoofing — and documented exactly which security barrier defeated each one.

The conclusion is the interesting part: the audit empirically demonstrates that Activation Lock is **not** a software lock but a trust policy enforced by a hardware-bound root-of-trust. Even with a BootROM compromise on the main CPU, the Secure Enclave Processor (SEP) manages the activation state independently, so every user-land modification triggers an integrity mismatch and a recovery loop. The report maps the full BootROM-to-AMFI chain of trust to explain why.

Topics covered: `checkm8` / DFU exploitation, AMFI & `CS_ENFORCE` code-signing enforcement, the Secure Enclave and Effaceable Storage, the activation handshake, and a structured cyber-kill-chain analysis of each attempted vector.

Tools: Ghidra, `ldid`, `tcpdump`, `ideviceinfo`, `iproxy`, `cycript`/`Frida`, GDB.

### CrackMe 02-64 — Solution Writeup
**[`Crackme02-64x.md`](./Crackme02-64x.md)**

A walkthrough of a 64-bit ELF CrackMe: reconnaissance with `checksec`/`strings`/`objdump`, static analysis in Ghidra to recover the password-checking algorithm, derivation of the valid input, and dynamic verification in GDB.

Tools: `file`, `checksec`, `strings`, `objdump`, Ghidra, GDB.

---

## Methodology

The writeups follow a consistent reverse-engineering workflow:

- **Reconnaissance** — fingerprint the target (binary protections, device state, available symbols).
- **Static analysis** — disassemble and read the logic in Ghidra to understand the control flow before running anything.
- **Dynamic analysis** — verify findings at runtime with a debugger or live device.
- **Conclusion** — document what worked, what didn't, and the underlying security mechanism that explains the result.

---

## Tools & Techniques

`Ghidra` · `GDB` · `objdump` · `checksec` · `strings` · `file` · `ldid` · `tcpdump` · `Frida` / `cycript` · `checkm8` · `ideviceinfo` · `iproxy` · ARMv8 / x86-64 assembly · Mach-O & ELF binary formats

---

## Notes

More CrackMe writeups are in progress and will be added over time. All analysis here is performed for educational and defensive-research purposes on owned or publicly-provided practice targets; no proprietary software is redistributed.
