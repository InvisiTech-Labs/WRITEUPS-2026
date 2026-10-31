# Vm inception

## Challenge overview

The challenge provides a single 64-bit Linux ELF executable named `crackme`. It accepts one command-line argument:

```bash
./crackme 'YOUR_GUESS_HERE'
```

A correct 34-byte argument prints `Correct! Well done.` and exits with status `0`. An incorrect argument prints `Incorrect.` and exits with status `1`.

The hints point toward three distractions and the actual solution:

- A readable string embedded in the binary is a decoy.
- There is an anti-debugging check involving `ptrace`.
- The real validation is performed by a small custom bytecode virtual machine (VM) inside `main`.

The objective is to recover the bytecode's per-character transformation and reverse it.

## 1. Basic file inspection

Identify the executable and inspect its printable strings:

```bash
file crackme
strings -a -t x crackme
```

The file is a stripped 64-bit ELF PIE executable. Among the printable strings is:

```text
flag{not_the_flag_you_are_looking_for}
```

This is not the answer. The string is referenced by an unusual `argc` branch (`argc == 0xf423f`) rather than the normal execution path used when the program is run with one argument. The ordinary invocation proceeds into the VM validation.

The binary also imports `ptrace`. This is an anti-debugging measure and explains why debugging the process can affect its behaviour. It is not the actual flag check; running the program normally from a shell is sufficient for final verification.

## 2. Locate the VM and its bytecode

Disassemble the executable:

```bash
objdump -d -Mintel crackme
```

The main routine contains an interpreter loop. It reads one byte from a bytecode region, uses that byte as an index into a jump table, and dispatches to the corresponding instruction handler.

The important addresses are:

| Address | Purpose |
|---|---|
| `0x1100` | Main routine / VM interpreter |
| `0x2060` | Relative jump table for VM opcodes |
| `0x20e0` | VM bytecode |
| `0x20a0` | Embedded decoy string |

The bytecode can be inspected with:

```bash
objdump -s -j .rodata crackme
```

The interpreter limits the bytecode program counter to `0x19d` and dispatches opcode values through the table at `0x2060`.

## 3. Reconstruct the VM instruction set

Following the jump-table entries into their handlers reveals the operations used by the VM. The relevant behaviour is:

| Opcode | Operation | Behaviour |
|---|---|---|
| `0x00` | Stop | Ends VM execution and checks whether any comparison failed |
| `0x01` | Push immediate | Pushes the following byte onto the VM stack |
| `0x02` | Load input | Pushes the input character at the following byte's index |
| `0x03` | XOR | XORs the top two stack values and replaces them with the result |
| `0x04` | ADD | Adds the top two stack values modulo 256 |
| `0x05` | SUB | Subtracts the top stack value from the value below it modulo 256 |
| `0x06` | Rotate left | Rotates the top stack byte left by the following immediate count |
| `0x07` | Compare | Compares the top stack byte with the following immediate byte; records a failure on mismatch |
| `0x08` | Jump | Loads a little-endian jump destination from the next two bytes |
| `0x09` | Conditional jump | Jumps when a previous comparison has failed; otherwise continues past the two-byte destination |
| `0x0a` | Advance | Advances the bytecode program counter by one |

The handlers do not all advance the program counter by the same amount. Some consume an immediate operand, while arithmetic and logic handlers advance by one byte. This is why it is important to follow the actual handler code instead of assuming a conventional fixed-width instruction format.

In the successful execution path for this challenge, the repeated per-character sequence uses opcodes `0x02`, `0x01`, `0x03`, `0x01`, `0x04`, `0x06`, and `0x07`. The `0x0a` instructions act as single-byte advances between some groups. The jump and conditional-jump handlers exist in the interpreter but are not needed to recover the successful path's character transformations.

## 4. Understand one character's validation

The repeated bytecode sequence performs the following operations for each input position `i`:

1. Load `input[i]` onto the stack.
2. Push an XOR constant, `X[i]`.
3. XOR the input byte with that constant.
4. Push an addition constant, `A[i]`.
5. Add the constant modulo 256.
6. Rotate the result left by `R[i]` bits.
7. Compare the result against the expected byte, `E[i]`.

The forward transformation is therefore:

```text
T[i] = ROL8(((input[i] XOR X[i]) + A[i]) mod 256, R[i])
```

The comparison succeeds when:

```text
T[i] = E[i]
```

Each of the 34 input bytes is checked independently using its own constants.

## 5. Reverse the transformation

The operations are reversible, so the original character can be recovered from the expected byte.

The forward operation is:

```text
T = ROL8((input XOR X) + A, R)
```

Undo the rotate first:

```text
U = ROR8(E, R)
```

Undo the addition:

```text
V = (U - A) mod 256
```

Undo the XOR:

```text
input = V XOR X
```

The complete inverse formula is:

```text
input[i] = (((ROR8(E[i], R[i]) - A[i]) mod 256) XOR X[i])
```

`ROL8` and `ROR8` are 8-bit circular rotations. All additions and subtractions are performed modulo 256.

## 6. Recover the 34 characters

The bytecode stores the constants directly in the repeated instruction sequence. For each character, the values are:

- `X`: XOR immediate
- `A`: addition immediate
- `R`: rotate-left count
- `E`: expected comparison byte

Applying the inverse formula to each group recovers the following input:

```text
flag{inm_the_vm_theres_another_vm}
```

The unusual substring `inm` is intentional: the recovered value is taken from the binary's actual per-byte checks rather than corrected based on what the flag might be expected to say.

## 7. Reproduce the recovery with Python

The following script reads the bytecode from the supplied ELF, walks the repeated instruction sequence, reverses each character transformation, and prints the recovered argument.

Save it as `solve.py` in the same directory as `crackme`:

```python
#!/usr/bin/env python3
from pathlib import Path

BINARY = Path("crackme")
BYTECODE_FILE_OFFSET = 0x20E0
BYTECODE_LENGTH = 0x19E


def ror8(value: int, count: int) -> int:
    """Rotate an 8-bit value right."""
    count %= 8
    if count == 0:
        return value & 0xFF
    return ((value >> count) | (value << (8 - count))) & 0xFF


def main() -> None:
    data = BINARY.read_bytes()
    code = data[BYTECODE_FILE_OFFSET:BYTECODE_FILE_OFFSET + BYTECODE_LENGTH]

    if len(code) != BYTECODE_LENGTH:
        raise RuntimeError("Could not read the complete VM bytecode")

    pc = 0
    recovered = bytearray()

    def expect_opcode(opcode: int) -> None:
        nonlocal pc
        actual = code[pc]
        if actual != opcode:
            raise RuntimeError(
                f"Unexpected opcode 0x{actual:02x} at bytecode offset 0x{pc:x}; "
                f"expected 0x{opcode:02x}"
            )
        pc += 1

    while pc < len(code) and code[pc] != 0x00:
        # LOAD input[index]
        expect_opcode(0x02)
        input_index = code[pc]
        pc += 1

        # PUSH xor_constant
        expect_opcode(0x01)
        xor_constant = code[pc]
        pc += 1

        # XOR
        expect_opcode(0x03)

        # PUSH addition_constant
        expect_opcode(0x01)
        addition_constant = code[pc]
        pc += 1

        # ADD
        expect_opcode(0x04)

        # ROTATE LEFT by the following immediate
        expect_opcode(0x06)
        rotate_count = code[pc]
        pc += 1

        # COMPARE against the following expected byte
        expect_opcode(0x07)
        expected = code[pc]
        pc += 1

        # Reverse: ROR, subtract, XOR.
        unrotated = ror8(expected, rotate_count)
        before_add = (unrotated - addition_constant) & 0xFF
        character = before_add ^ xor_constant

        if input_index != len(recovered):
            raise RuntimeError(
                f"Unexpected input index {input_index}; expected {len(recovered)}"
            )
        recovered.append(character)

        # The VM uses 0x0a as a one-byte advance at some boundaries.
        if pc < len(code) and code[pc] == 0x0A:
            pc += 1

    if len(recovered) != 34:
        raise RuntimeError(f"Recovered {len(recovered)} bytes, expected 34")

    print(recovered.decode("ascii"))


if __name__ == "__main__":
    main()
```

Run the solver:

```bash
python3 solve.py
```

Expected output:

```text
flag{inm_the_vm_theres_another_vm}
```

## 8. Verify the flag against the executable

Make sure the binary is executable and run it with the recovered argument:

```bash
chmod +x crackme
./crackme 'flag{inm_the_vm_theres_another_vm}'
```

Expected output:

```text
Correct! Well done.
```

The executable exits with status `0` for the recovered input.

## 9. Key takeaways

1. **Do not trust every printable string.** The readable flag-like string in `.rodata` is a decoy used by a separate, unusual argument-count branch.
2. **Recognize anti-debugging behaviour.** The `ptrace` calls can interfere with debugging, but they are separate from the real validation logic.
3. **Follow indirect dispatch.** The jump table at `0x2060` maps bytecode values to their native instruction handlers.
4. **Track program-counter changes carefully.** The VM uses variable advancement rules, and some operations consume immediate bytes while others advance only one byte.
5. **Reverse operations in the opposite order.** The character transformation uses XOR, addition, and rotation. Recovering the original byte requires undoing rotation, then addition, then XOR.

## Flag

```text
flag{inm_the_vm_theres_another_vm}
```
