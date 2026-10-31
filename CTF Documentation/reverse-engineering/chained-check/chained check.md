# Chained check

## Challenge Overview

The challenge provides an ELF 64-bit Linux executable named `crackme_med`.

The objective is to reverse engineer the binary and recover the correct 34-byte input that causes the program to print:

`Correct! Well done.`

## 1. Initial Analysis

First, inspect the binary to identify its file type and available strings.

```bash
file crackme_med
strings -a crackme_med
```

The output confirms that this is a 64-bit ELF executable.

The strings reveal three useful messages:
- usage: %s <flag>
- Incorrect.
- Correct! Well done.
This suggests that the program accepts an input argument and validates it against some internal conditions.

## 2. Disassembling the Binary
Use objdump to examine the main function.
objdump -d -Mintel crackme_med


The disassembly reveals two major validation stages.
Check 1: Input Length
cmp eax, 0x22


The value 0x22 is 34 in decimal.

Therefore, the input must contain exactly 34 bytes.
Check 2: Whole-Input Checksum
The following instructions perform a byte-by-byte addition:
```bash
mov ecx, 0x0

add cl, BYTE PTR [rax]
add rax, 0x1
cmp rax, rsi
```

After processing all 34 bytes, the program checks:
cmp cl, 0x29


This means the sum of all input bytes must equal 0x29 modulo 256.

In mathematical terms:
(X0 + X1 + ... + X33) mod 256 = 0x29


This is a relatively weak validation condition.

Many different strings could satisfy the checksum, so passing this check does not mean the correct flag has been found.

## 3. Understanding the Chained Comparison
The interesting part of the challenge is the second validation loop.

The relevant assembly instructions are:
mov esi, 0xd
mov ecx, 0xffffffab

lea r8, [rip+0xe62]

mov edi, ecx
movzx ecx, BYTE PTR [rbx+rax*1]

mov edx, esi
xor edx, ecx
add edx, edi

cmp dl, BYTE PTR [r8+rax*1]


The important detail is:
mov edi, ecx


This instruction executes before ecx is overwritten with the current input byte.

Initially, ecx contains:
0xFFFFFFAB


However, after the first iteration, ecx contains the current input byte.

Consequently, the value added during each iteration changes.
- First iteration: the addition uses 0xAB.
- Subsequent iterations: the addition uses the previous input byte.
The key observation is that the comparison is chained.

Each input byte influences the validation of the next byte.

## 4. Reversing the Comparison
The program performs the following operation for each byte:
Expected[i] = ((Key[i] XOR Input[i]) + Previous[i]) mod 256


Where:
Key[i] = 13 + (7 * i)


And:
Previous[0] = 0xAB
Previous[i] = Input[i - 1]


To recover the input, reverse the operations.

First, subtract the previous byte:
Temporary = (Expected[i] - Previous[i]) mod 256


Then reverse the XOR:
Input[i] = Temporary XOR Key[i]


Combining both operations:
Input[i] = ((Expected[i] - Previous[i]) mod 256)
           XOR (13 + 7 * i)


This allows the input to be recovered sequentially, starting with the first byte.

## 5. Extracting the Expected Bytes
The binary stores the expected comparison bytes in its .rodata section.

Inspect the section using:
objdump -s -j .rodata crackme_med


The expected bytes begin at address 0x2040.

Since the input is 34 bytes long, extract the 34 expected bytes and apply the inverse operation described above.

## 6. Recovering the Flag
The recovered input is:
flag{ch41ned_xor_1s_ju5t_a_w4rmup}


## 7. Verification
Run the binary with the recovered input:
chmod +x crackme_med

```bash
./crackme_med 'flag{ch41ned_xor_1s_ju5t_a_w4rmup}'
```

Expected output:
Correct! Well done.


The recovered input satisfies all validation conditions:
Validation	Result
Input length	34 bytes
Byte sum modulo 256	0x29
Chained byte comparisons	Passed
Program validation	Passed

## 8. Key Takeaways
This challenge demonstrates several useful reverse-engineering concepts:
1. Static analysis: Using strings, file, and objdump to understand an executable without source code.
2. Assembly analysis: Following register assignments and identifying how values change between loop iterations.
3. XOR reversal: XOR is reversible when the same key is applied again.
4. Chained validation: A previous input byte can influence the validation of the current byte.
5. Register reuse: A register initially holding a constant can be overwritten, changing the behaviour of subsequent loop iterations.
The main trick was not the XOR operation itself, but recognizing that the addition was not using the same constant on every iteration.
Flag
```bash
flag{ch41ned_xor_1s_ju5t_a_w4rmup}
```