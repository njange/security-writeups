# Risky Chain — CTF Writeup

## Challenge Information

- **Name:** Risky Chain
- **Category:** Blockchain, RISC-V
- **Difficulty:** Medium
- **Points:** 100
- **Description:** "Metal to the future!"

---

## Solution

### Overview

This challenge involves a custom blockchain implementation where each block contains executable RISC-V code. The goal is to mine a valid block with RISC-V assembly that calls a special system call (ECALL 1337) to retrieve the flag.

When connecting to the service, the server presents a genesis block (Block 0) and prompts us to submit a new block that includes:

- A valid nonce (proof-of-work)
- RISC-V assembly code that gets executed by the chain's VM when the block is accepted

The key insight is that ECALL 1337 prints the flag, but the assembler intentionally blocks the usual ECALL argument register x17 (a7).

### Step 1: Understanding the Challenge

Upon manual interaction with the service we see two inputs required:

1. Nonce (given as hex)
2. RISC-V assembly contract (multi-line, terminated by a blank line)

The ECALL instruction triggers a handler that will print the flag if it receives syscall number 1337. However, the assembler/environment blocks direct use of `x17` (a7), presumably to prevent trivial ECALL usage.

### Step 2: Finding a Valid Nonce

We attempted a few common nonces used in CTFs and PoW bypasses (simple brute force and pattern testing). The nonce that consistently passed the proof-of-work check was:

```
Nonce (as hex): deadbeef
```

This turned out to be a valid and accepted nonce for mining a block.

### Step 3: Bypassing the x17 Restriction

Although `x17` (a7) is blocked, other argument registers were allowed. We discovered that `x10` (a0) was permitted. The ECALL handler interestingly checks `x10` for the syscall number (in addition to — or instead of — `x17`). That allowed us to place the syscall number 1337 into `x10` and then execute `ecall`.

The minimal RISC-V assembly used was:

```asm
addi x10, zero, 1337
ecall
```

This sequence:

- Loads immediate 1337 into register `x10` (a0)
- Executes `ecall`, which the VM's handler interprets and prints the flag when it sees the 1337 syscall number in `x10`

### Step 4: Getting the Flag

The complete interaction looked like this when submitting the block:

```
Nonce (as hex): deadbeef
Enter your RISC-V assembly contract (end with a blank line):
addi x10 zero 1337
ecall
```

The service responded with the flag:

```
infobahn{Th3_futur3_15_m3t4l1c_4nd_RISC-V}
```

---

## Technical Details

- Proof-of-work bypass: The nonce `0xdeadbeef` consistently satisfied the PoW check for this challenge. It was found via quick brute-force testing of common patterns.

- Register workaround: The assembler/environment blocks `x17` (a7) to prevent naive ECALL usage. However, the ECALL handler appears to examine multiple registers (notably `x10`/a0) for the syscall number. By placing `1337` into `x10` and issuing `ecall`, we trigger the flag-printing path.

- RISC-V execution: The custom blockchain executes the submitted RISC-V assembly in a VM when a valid block is mined. This made the challenge a blend of blockchain mining and low-level syscall exploitation.

### Why This Works

- The PoW check was satisfied by a known nonce pattern (`deadbeef`).
- The ECALL handler's implementation detail (checking `x10`) allowed us to bypass the explicit `x17` restriction imposed by the assembler.

### Key Learning Points

- Blockchain CTFs can combine PoW puzzles with arbitrary code execution via embedded contracts.
- Familiarity with the target CPU architecture (RISC-V) and calling conventions helps find alternative register paths.
- Security controls that block a single register can be insufficient if other registers or execution paths are not tightly controlled.

## Tools Used

- Netcat (or similar) for interactive service connection
- Python for brute-forcing/testing nonces
- Basic RISC-V assembly knowledge (for writing the minimal contract)

## Conclusion

The challenge title "Metal to the future!" hints at low-level RISC-V programming. The intended solution is to find a valid nonce and craft a tiny RISC-V assembly payload that uses an allowed register (`x10`) to pass the syscall number `1337` to the ECALL handler. With those two pieces — nonce `0xdeadbeef` and the assembly snippet — the service prints the flag:

```
infobahn{Th3_futur3_15_m3t4l1c_4nd_RISC-V}
```

This challenge demonstrates how execution of user-supplied code in a blockchain environment can be abused when sandboxing is incomplete or system call handlers trust unexpected registers.

---

### Notes / Next Steps

- If you plan to include this writeup in `CTFs/infobahnCTF2025/README.md`, you can link to this file from there.
- For a defensive writeup: recommend the challenge author sanitize and strictly control how syscalls are invoked in the VM, and ensure PoW validation is robust (or vary accepted nonce space) to avoid trivial patterns.

---

*Writeup created by an automated assistant. File: `Risky_Chain.md`*
