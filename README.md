<!---
{
  "id": "f87c7e89-ece7-4c55-af54-16a3b3b7435f",
  "depends_on": [
    "AND",
    "302c98a7-cbea-435c-ada2-bbf7538429a2",
    "81f2e303-d35c-4857-9cb7-190e3c5372b0",
    [
      "OR",
      "718193ef-11a1-408d-af23-4b10c24d490d", 
      "99787eda-617a-4a68-b9a4-d60ec5c5c303"  
    ]
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2025-06-05",
  "keywords": ["C Compiler", "Inline Assembly", "Syscall", "Objdump", "Locals and Globals", "Primitive Types"]
}
--->

# C Compiler: Working with Primitive Types and Inspecting Binaries

> In this exercise you will learn how to use basic C types such as `int`, `char`, and `float` and observe how the compiler transforms these into machine code. Furthermore we will explore how to analyze object files and executables using `objdump`, and how inline assembly can directly invoke system calls by manipulating registers.

## Introduction

Up until now, we have mostly focused on the general structure of the compiler and how it translates high-level instructions into machine code. In this exercise, we will dive a little deeper and explore how simple C constructs are represented internally.

The C language provides a set of **primitive data types**, such as:

* `char`: typically 1 byte
* `int`: typically 4 bytes on most modern systems
* `float`: typically 4 bytes, following IEEE-754 floating point standard

When you declare and manipulate these types in your C code, the compiler generates machine code that reflects these operations, assigning memory addresses (either on the stack for locals, or in the `.data` / `.bss` section for globals).

Unlike the previous exercise, where the compiler managed everything for us, we will now intentionally compile partial code and analyze intermediate representations to understand what the compiler emits at each stage. This gives us visibility into:

* how variables are laid out in memory,
* how data types affect the generated instructions,
* how the calling conventions handle return values and argument passing.

We will also explore a very low-level capability of C: embedding **inline assembly**. This allows us to directly control CPU registers and issue instructions that are normally not accessible from plain C. In particular, we will craft a simple system call manually, without using the C library.

This level of understanding is crucial for systems programming, operating system development, and reverse engineering, where knowing the ABI (Application Binary Interface) and calling conventions can make the difference between success and failure.

> 🔍 **Why `objdump`?**
> The `objdump` utility allows you to disassemble binaries and object files, revealing the exact instructions the CPU will execute. You can use it to correlate your C code with the resulting machine instructions, providing valuable insights into how compilers translate high-level abstractions into hardware operations.

### Further Readings and Other Sources

* [System V AMD64 ABI (PDF)](https://gitlab.com/x86-psABIs/x86-64-ABI/-/raw/master/x86-64-ABI.pdf)
* [Linux System Calls Table](https://filippo.io/linux-syscall-table/)
* [GCC Inline Assembly Guide](https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)
* [Introduction to objdump (YouTube)](https://www.youtube.com/watch?v=qJ1jYvLkxKg)

## Tasks

### Task 1: Simple Program with Primitive Types

Create the following C file:

```C
// file: primitives.c

int global_int = 42;
char global_char = 'A';
float global_float = 3.1415;

int main() {
    int local_int = global_int + 5;
    char local_char = global_char + 1;
    float local_float = global_float * 2.0;

    return local_int;
}
```

#### a) Compile and inspect

* Compile normally:
  `gcc -Wall -o primitives primitives.c`
* Inspect with `file`, `ls -l`, `strings`, and `objdump -d primitives`.
* Observe where globals are stored (`.data` section), where locals appear on the stack, and how return values are handled via registers (`eax` / `rax`).

#### b) Compile only to object file:

* `gcc -Wall -c primitives.c -o primitives.o`
* Inspect with:
  `objdump -d primitives.o`
  `objdump -s -j .data primitives.o`

#### c) Analyze:

* What instructions are used for `float` multiplication? movss  0x2ec6(%rip),%xmm0
                                                        addss  %xmm0,%xmm0  Instead of multiplieing, it works by simply addin 2 times the register %xmm0 instead of multiplieng times 2.0
  
* Where are `char` manipulations visible in the assembly? movzbl 0x2ed0(%rip), %eax 
                                                          add   $0x1, %eax
                                                          mov   %al, -0x9(%rbp)
  
* Which registers are used for returning `int` from `main`? -0x8(%rbp) and %eax for returning

---

### Task 2: Compiling a Standalone Function

Now create a second file:

```C
// file: function.c

int add(int a, int b) {
    int sum = a + b;
    return sum;
}
```

#### a) Compile to object file only:

* `gcc -Wall -c function.c -o function.o`

#### b) Inspect with objdump:

* `objdump -d function.o`

#### c) Observe:

* How parameters are passed into the function (e.g. registers `rdi`, `rsi`).
    mov    %edi,-0x14(%rbp)
   mov    %esi,-0x18(%rbp)
   mov    -0x14(%rbp),%edx
   mov    -0x18(%rbp),%eax
  Now follows a simplified explanation:
  the 2 variables of the functions get loaded into the storage, than they get loaded into %eax and %edx registers. Those registers are than added...
* How return value is placed into `eax` / `rax`. eax is stored on rbp that is loaded into eax, rbp gets restored and eax is returned with ret.

---

### Task 3: Inline Assembly — Writing to Terminal

Create the following file:

```C
// file: syscall.c

char message[] = "Hello via syscall\n";

int main() {
    long len = 17; // length of message
    long fd = 1;   // stdout

    asm volatile (
        "movq $1, %%rax\n"     // syscall number for write
        "movq %0, %%rdi\n"     // file descriptor
        "movq %1, %%rsi\n"     // pointer to message
        "movq %2, %%rdx\n"     // message length
        "syscall\n"
        :
        : "r"(fd), "r"(message), "r"(len)
        : "%rax", "%rdi", "%rsi", "%rdx"
    );

    return 0;
}
```

#### a) Compile:

* `gcc -Wall -o syscall syscall.c`

#### b) Inspect:

* Use `objdump -d syscall` to find where the syscall is issued.
* Verify which registers are loaded before `syscall` is executed.   mov    $0x1,%rax
  mov    %rcx,%rdi
#### c) Change Output Stream:

* Try changing the `fd` to `2` and verify that the message goes to `stderr`. Works

---

## Questions

1. How are `int`, `char`, and `float` values represented in memory? int = 4 Byte , float = 4 Byte, char= 1Byte
2. What is the purpose of the `.data` section in the object file? The .data section is used to declare variables and give them their value.
3. How does the compiler handle return values from `main`? he does this by loading data in to the %eax register. With ret he is returning that. 
4. Which registers are used for argument passing in `add()`? %edi and %esi
5. Why do we need to declare clobbered registers in inline assembly? Otherwise the compiler thinks that they are empty and uses the old content from them. That could cause bugs, because the program then uses wrong content.
6. What is the syscall number for `write` on x86\_64 Linux? Its 1.
7. How does `objdump` help you correlate source code with generated machine code? It shows me the interpretation of my code, it helps me to make my code more efficient.

## Advice

In this exercise, you got your first taste of how closely C can interact with the hardware. Although C is a high-level language, you are only one layer away from manipulating registers and issuing system calls directly. Inline assembly is a powerful tool — but should be used sparingly in real-world code. Always remember to analyze your binaries using `objdump` and compare the generated assembly with your expectations. As you grow more confident, try using `gcc -O1` or `-O2` to observe how optimizations modify your generated code. Later exercises will deepen your understanding of calling conventions, stack layout, and more advanced system interaction.
