# Preloader stack

The load address of the Preloader is `0x12001000` and it's a 32-bit ARMv7 binary. The top of the stack is `0x1201E650` and the initial stack pointer is set to `0x1201F24C`. The stack size is `0xC00` bytes long.

Both the stack address and the stack size are loaded to set SP at a high address, so it's safe to assume that the stack grows downwards. The stack is initialized with `0xDEADBEEF` at the top of the stack, which seems to be a buffer overflow detection mechanism.

By analyzing the preloader image, we can observe a lot of `push` and `pop` instructions, which are synonyms for `stmfd` and `ldmfd` respectively (store and load multiple registers full descending), which means that we're dealing with a full descending stack.

The stack is initialized at the top of the preloader image, which is what gets executed first. The stack is then set to the calculated start address, and the execution is branched to the main function:
```assembly
120010c0 bc  00  1f  e5    ldr   r0, =0x1201E650  ; Load the top of the stack into r0
120010c4 bc  10  1f  e5    ldr   r1, =0x12019EF8  ; Load the stack size address into r1
120010c8 10  21  9f  e5    ldr   r2, =0xDEADBEEF  ; Load 0xDEADBEEF into r2 (buffer overflow detection?)
120010cc 00  20  80  e5    str   r2, [r0]         ; Store 0xDEADBEEF at the top of the stack
120010d0 00  10  91  e5    ldr   r1, [r1]         ; Load stack size (0xC00) from address `0x12019EF8`
120010d4 04  10  41  e2    sub   r1, r1, #0x4     ; Subtract 4 from the stack size (0xC00 - 4 = 0xBFC)
120010d8 01  10  80  e0    add   r1, r0, r1       ; Calculate the initial stack pointer position (0x1201E650 - 0xBFC = 0x1201F24C)
120010dc 01  d0  a0  e1    mov   sp, r1           ; Set the stack pointer to the calculated start address (0x1201F24C)
120010e0 42  00  00  ea    b     main             ; Branch to the main (non-ASM) function (main)
```

The following table tries to keep track of the stack operations that are performed in the preloader image, starting from the top of the stack:
| Operation                      | SP           | LR Address   | Comment                                      |
|--------------------------------|--------------|--------------|----------------------------------------------|
| `sub sp, #0x84`                | `0x1201F1B4` | `0x1201F248` | SP moved down by `0x84` for local allocation |
| Initial stack pointer set      | `0x1201F24C` | `0x1201F248` | SP points here after setup                   |
| `push {r4, r5, r6, r7, lr}`    | `0x1201F238` | `0x1201F248` | `LR` saved at `0x1201F248`, SP at `0x1201F238` |
| Entry point (initial SP)       | `0x1201F24C` | -            | Preloaded with top address for execution     |
| Store `0xDEADBEEF`             | `0x1201E650` | -            | Buffer overflow detection                    |