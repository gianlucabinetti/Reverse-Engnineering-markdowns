# Crackme02.64 - Solution Writeup

## Reconnaissance
- ELF 64-bit, PIE enabled, NX enabled, no stack canary
- 32 symbols available (not stripped)

## Static Analysis
- Found hardcoded string "password1" at 0x2005
- Algorithm: Subtracts 1 from each expected character before comparing

## Solution
- Expected: `password1`
- Algorithm: Each input character = (expected character - 1)
- Password: `o`rrvnqc0`

## Verification (GDB)

- (gdb) break main
- (gdb) run 'orrvnqc0' (gdb) continue Yes, orrvnqc0 is correct!


## Tools Used
- `file`, `checksec`, `strings`, `objdump` - Reconnaissance
- `Ghidra` - Pseudocode analysis
- `GDB` - Dynamic verification
