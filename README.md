# Factorial in RISC-V Assembly (RV32I) Using Only Additions

A RISC-V (RV32I) assembly program that reads an integer from the keyboard and computes its factorial **using only additions**: every multiplication is replaced by repeated addition. The program follows the standard function-call conventions (prologue/epilogue, stack frames aligned to 16 bytes, argument and return value in `a0`).

> Project developed for the course **Computer Systems Architecture**, BSc in **Electrical and Computer Engineering**, **Instituto Politécnico do Cávado e Ave (IPCA)**, **2st year**.

The assignment was to translate the following C program to RV32I assembly:

```c
#include <stdio.h>

int fatorial(int n);

int main() {
    int num;
    printf("Digite um numero inteiro: ");
    scanf("%d", &num);
    int resultado = fatorial(num);
    printf("O fatorial de %d = %d\n", num, resultado);
    return 0;
}

int fatorial(int n) {
    int resultado = 1;
    for (int i = 1; i <= n; i++) {
        resultado *= i;
    }
    return resultado;
}
```

---

## Repository structure

```
.
├── fatorial.s      # assembly source (RV32I)
└── README.md
```

---

## Running the program

The `ecall` numbers used (see below) follow the **RARS** convention, so the easiest way to run it is with [RARS](https://github.com/TheThirdOne/rars):

1. Open `fatorial.s` in RARS (*File → Open*).
2. Assemble it (*Run → Assemble*, `F3`).
3. Run it (*Run → Go*, `F5`) and type a number in the *Run I/O* console.

Example:

```
Digite um número inteiro: 5
O fatorial de 5 = 120
```

---

## How it works

### The algorithm: multiplication by repeated addition

RV32I has no multiply instruction (that comes with the `M` extension), so the factorial is computed with two nested loops:

- **Outer loop** (`fatorial_loop`): `t0` counts down from `n` to `1`. At each step the running result `t1` is multiplied by `t0`.
- **Inner loop** (`somas_loop`): performs that multiplication by adding `t1` to an accumulator `t3`, `t0` times.

Trace for `n = 3`:

| Outer step | `t0` | `t1` before | Inner loop (`t3`) | `t1` after |
|---|---|---|---|---|
| 1 | 3 | 1 | 1 + 1 + 1 | 3 |
| 2 | 2 | 3 | 3 + 3 | 6 |
| 3 | 1 | 6 | 6 | 6 |
| end | 0 | 6 | – | returns `6` |

The inner loop runs `t0` times in each step, so the whole computation needs only `n(n+1)/2` additions. For `n = 0` the outer loop ends immediately and the result stays `1`.

### Program flow

1. **`_start`** (equivalent to `main()`): sets up its stack frame, prints the prompt, reads the integer (`ecall` 5) and stores it at `0(sp)`.
2. It calls **`fatorial`** with `jal`. The number is already in `a0` because `ecall` 5 returns the value read there.
3. `fatorial` computes the result with the two loops and returns it in `a0`.
4. Back in `_start`, the result is stored at `4(sp)`, and then the program prints `O fatorial de `, the input number, ` = ` and the result.
5. The program ends with `ecall` 10 (exit).

### Registers used in `fatorial`

| Register | Role |
|---|---|
| `a0` | Argument `n` on entry, return value on exit |
| `t0` | Outer-loop counter, from `n` down to `1` |
| `t1` | Running result (starts at `1`) |
| `t2` | Inner-loop counter (`i` in the C code) |
| `t3` | Accumulator for the current multiplication |

Only temporary (`t`) registers are used for the calculation. They don't need to be saved by the function, so no local variables are stored on the stack. The loop counter `i` of the C code lives in `t2`, and the comparison `i <= n` becomes `bne t2, t0, somas_loop`, which repeats the inner loop until `t2` reaches `t0`.

### Prologue and epilogue

Both functions reserve a 16-byte stack frame, which keeps `sp` aligned to 16 bytes as the calling convention requires.

**`_start` prologue**

| Instruction | Purpose |
|---|---|
| `andi sp, sp, -16` | Aligns `sp` down to a multiple of 16 bytes |
| `addi sp, sp, -16` | Reserves 16 bytes (the stack grows towards lower addresses) |
| `sw ra, 12(sp)` | Saves the return address, because `jal fatorial` overwrites `ra` |
| `sw s0, 8(sp)` | Saves the caller's frame pointer |
| `mv s0, sp` | Sets the frame pointer to the new frame |

**`_start` epilogue**: restores `ra` and `s0` from the stack, releases the 16 bytes with `addi sp, sp, 16`, and ends with `li a7, 10` + `ecall`. It exits through a system call instead of `jr ra`, because `_start` is the program's entry point and was not called by another function.

**`fatorial` prologue**: the same four instructions as `_start`, without the `andi` (the stack is already aligned when the function is called).

**`fatorial` epilogue**

| Instruction | Purpose |
|---|---|
| `lw ra, 12(sp)` | Restores the return address |
| `lw s0, 8(sp)` | Restores the caller's frame pointer |
| `addi sp, sp, 16` | Releases the frame |
| `jr ra` | Returns to `_start` |

### Stack frames

`_start` frame (16 bytes):

| Offset | Content |
|---|---|
| `12(sp)` | `ra` (return address) |
| `8(sp)` | `s0` (frame pointer) |
| `4(sp)` | Factorial result |
| `0(sp)` | Number read from the keyboard |

`fatorial` frame (16 bytes): `ra` at `12(sp)` and `s0` at `8(sp)`. The other 8 bytes are unused. Since the function is iterative, not recursive, stack usage is constant: 32 bytes in total.

### System calls (`ecall`)

| `a7` | Service |
|---|---|
| `1` | Print integer |
| `4` | Print string |
| `5` | Read integer |
| `10` | Exit program |

---

## Limitations

- **Integer overflow:** results are 32-bit signed integers, so the largest correct input is **12** (`12! = 479001600`). From `13!` on the result overflows. For `n = 100`, `100! ≈ 9.3 × 10^157`, far outside the 32-bit range.
- **Stack overflow is not a concern:** the program uses a fixed 32 bytes of stack, regardless of the input.
- **Negative input is not handled:** the outer loop only ends when the counter reaches `0`, so a negative `n` would not terminate in a reasonable time.
- Comments and messages in the source code are in Portuguese.

---

## Author

**Ivo Azevedo**
