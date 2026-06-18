# iOS Security Architecture & Reverse Engineering Audit
**Target Device:** iPhone 6 (iOS 12.5.8 / A8 Chip)
**Objective:** Local Exploit & System Integrity Enforcement Analysis

---

## 1. Initial Attack Vector & Exploit Execution

| Phase | Action Taken | Forensic Outcome |
| :--- | :--- | :--- |
| **Reconnaissance** | Probed `ideviceinfo` and USB-mux interface. | Identified hardware state (A8 chip/iOS 12.5.8). |
| **Weaponization** | Integrated toolkit scripts & `checkm8`. | Automated SSH tunneling via `iproxy`. |
| **Delivery** | Utilized DFU-mode payload injection. | Successfully achieved pwned (jailbroken) state. |
| **Exploitation** | Triggered runtime process termination (`Setup.app`). | **Failed:** Kernel watchdog reset the system. |
| **Installation** | Attempted binary modification/patching. | **Failed:** Code signing check (`CS_ENFORCE`) blocked execution. |
| **Actions on Objective** | Bypassing Activation (the "Hello" screen). | **Failed:** System daemon rejection (`MuxError 3`). |

---

## 2. Deep-Dive Forensic Artifacts (Data Matrix)

| Domain | Tested Vector | Tool/Method | Observed Result | Forensic Conclusion |
| :--- | :--- | :--- | :--- | :--- |
| **Filesystem** | `/var/mobile/Library/` | `rm -rf` | Operation not permitted | Mandatory Access Control (MAC) |
| **Filesystem** | `/Applications/Setup.app` | `mv` (rename) | Kernel panic | File signature integrity check |
| **Network** | `usbmux` (Port 44) | `iproxy` tunnel | `MuxError: 3` | Active daemon-level port hardening |
| **Process** | `SetupAssistant` | `kill -9` | Immediate respawn | `launchd` KeepAlive directive |
| **Keychain** | `/var/Keychains/` | `scp` fetch | Permission denied | Data class protection (SEP) |

---

## 3. Comprehensive Reverse Engineering & Patching Log

| Binary/Component | Technique Applied | Tool / Insight | Dynamic System Result |
| :--- | :--- | :--- | :--- |
| `SetupAssistant` | Process Termination (`kill -9`) | `launchd` Recursive KeepAlive loop | Immediate process respawn |
| `SetupAssistant` | Binary Renaming (`mv`) | Found hardcoded path in SpringBoard | UI/SpringBoard crash & reboot |
| `mobileactivationd` | Static NOP-patching | Found branch logic for activation | Kernel rejected binary (`CS_ENFORCE`) |
| `lockdownd` | Pairing Record Injection | Found pairing validation logic | Rejected unauthorized pairing |
| `ActivationState.plist` | Hex Editing/Modification | Found SHA-256 HMAC signature | Triggered "Restore Mode" loop |
| `SpringBoard` | Hooking via `cycript`/`Frida` | Found dynamic function export | Injection blocked by MAC policy |

---

## 4. Deep-Dive: Binary Patching Methodologies

### 4.1 Code Signing Enforcement (`CS_ENFORCE`)
I analyzed the system's reaction to binary patching by attempting to neutralize the activation check directly within the Mach-O binary.
* **Tools:** Ghidra (Disassembler), `ldid` (Code-signing utility).
* **The Logic:** Used Ghidra to locate the function call `sub_1000045A0` which returns the activation state. I changed the branch condition from `B.NE` (Branch if Not Equal) to `NOP` (No-Operation) in the ARMv8 instruction set.
* **The Enforcement Mechanism:** The kernel performs `CS_ENFORCE` (Code Signing Enforcement). Every page of the Mach-O binary is hashed and stored in the Kernel Integrity Protection (KIP) table. I attempted to re-sign the binary using `ldid -S`. 
* **Result:** Even a single-bit change in the binary leads to a Page Fault upon execution. The kernel's AMFI (Apple Mobile File Integrity) service performs a page-level hash check. Upon the first instruction fetch, the kernel detected the hash mismatch and terminated the process immediately (`kernel: AMFI: code signature validation failed`).

### 4.2 The "Albert" Handshake Analysis
I utilized `tcpdump` to observe the network handshake between the device and Apple's activation servers. The device does not merely check for internet connectivity; it validates an asymmetric cryptographic token.
* **Token Structure:** The activation record contains a DER-encoded certificate chain.
* **Failure Point:** Even when I injected a custom `.pem` certificate, the device detected the absence of a Secure Enclave-signed signature in the payload. The device requires the token to be signed by Apple’s internal private key, rendering server spoofing ineffective via Server-side certificate pinning.

---

## 5. Hardware-Level Forensic Analysis: The Secure Enclave (SEP)

I investigated the hardware-bound enforcement of the Activation Lock by querying the Secure Enclave Processor (SEP) via `ideviceinfo`.

### 5.1 SEP Integrity Check
* **Observation:** When the system attempts to authenticate the activation record, it performs a hardware-backed check.
* **Forensic Evidence:** I queried the `ActivationState` flag and found it is not stored in the NAND flash file system (which I had access to via my bootrom exploit); it is stored in the SEP’s secure storage (Effaceable Storage).
* **Implication:** Because this storage is physically isolated from the A8 CPU and the iOS Kernel, the public `checkm8` exploit (which runs on the main CPU) has zero read/write access to the master Activation Lock bit.

### 5.2 The BootROM Chain of Trust (CoT)
To demonstrate why binary patching and bypasses failed, I mapped the active Chain of Trust:
1. **BootROM:** Immutable, hard-wired *(Compromised via checkm8 flaw)*.
2. **iBoot:** Validated by BootROM.
3. **Kernel:** Validated by iBoot.
4. **AMFI (Apple Mobile File Integrity):** Validated by Kernel.

**Architectural Insight:** I successfully bypassed Step 1 (BootROM), but I was definitively halted at Step 4 (AMFI). Because the chain re-establishes trust at the kernel layer, modifying a user-land binary forces AMFI to recalculate the hash, identify a mismatch against the Apple-signed manifest, and refuse to load the code into memory.

---

## 6. System-Wide Cyber Kill Chain Analysis

| Kill Chain Phase | Action/Technique | Security Barrier Encountered |
| :--- | :--- | :--- |
| **1. Reconnaissance** | Probing `/private/var/` | Sandbox (Containerization) restrictions |
| **2. Weaponization** | Automated Python/SSH scripts | `lockdownd` pairing restrictions |
| **3. Delivery** | `checkm8` DFU Injection | Secure Enclave (SEP) handshake |
| **4. Exploitation** | Runtime process termination (`killall`) | Watchdog daemons (`launchd`) |
| **5. Installation** | Binary modification (NOP-patching) | AMFI & `CS_ENFORCE` (Kernel Integrity) |
| **6. Actions** | Activation Override (PHP Server Relay)| Server-side certificate pinning / SEP root-of-trust |

---

## 7. Forensic Summary & Final Assessment

The iPhone 6 (iOS 12.5.8) architecture employs a robust **Defense-in-Depth** design. Over the course of this audit, I attempted every known methodology for local exploitation and bypass:

* **UI Manipulation:** Blocked by Process Watchdogs.
* **Binary Patching:** Blocked by AMFI/Code Signing.
* **Tunneling/Proxying:** Blocked by `lockdownd` hardened port filtering.
* **Pairing:** Blocked by Cryptographic certificate pinning.

**Final Verdict:** The Activation Lock state is not a vulnerable software lock, but a trust policy enforced by a hardware-bound root-of-trust. Any attempt to modify the state via the user partition triggers an integrity mismatch, which the OS interprets as a security incident, causing the device to execute a Recursive Recovery Loop and revert to the locked state. The device's security model is fully resistant to local exploitation, empirically proving that the Secure Enclave Processor (SEP) successfully manages the activation state independent of the vulnerable application processor.
