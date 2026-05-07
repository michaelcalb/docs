---
title: bingOS
---
[[index|🏠 Return to Hub]] [🐈‍⬛ Project's Repository](https://github.com/michaelcalb/bingOS)

---

> [!NOTE] This OS was designed for legacy BIOS and it uses a 32-bit kernel. [[#Clarification]]
# Introduction
Have you ever wondered, how can you simply press the power button on your PC and suddenly your whole OS appears in front of you? All that button does is just enable energy to pass through the hardware, so how does this work?

Before we talk about the actual OS code inside your drive (HDD, SSD) we need to talk about your **motherboard code**, also known as the **firmware**. Yes, your motherboard, specially modern ones, have **millions** of lines of firmware code.

If you are a computer person, you probably have already heard terms like "BIOS", "MBR", "GPT", "UEFI" and "CSM", and if you *really* are a computer person, maybe "A20 pin" and "CHS" too. If those are new to you, don't worry, this article will explain everything to you, from the moment you press the power button, until you login into your OS.
# Clarification
Modern motherboards (from around 2012 onwards) come with a system called **UEFI** (Unified Extensible Firmware Interface). In short, this is a mini Operating System built-in inside your motherboard designed to make the life of OS developers easier, doing all the hard-lifting for you.

Since we want to learn how the booting system works from the start, we will build an OS following the standards before UEFI was a thing. However, what you will see today is not 100% "obsolete" and "unusable" code. For today's standards, we will essentially be writing the very foundations of the UEFI system itself.

Modern OSs are also built on top of the CPU's 64-bit mode, which allows more advanced features but a whole new complexity level to our code. To enable the 64-bit mode, you still need to first write some code in 16-bit, then enable 32-bit, write more code and finally enable 64-bit. So for modern standards, this bootloader is not technically wrong, just incomplete.

However, some parts of the **explanations** and **code** you're about to see *can* be considered **obsolete**. The OS I built uses some technologies and methods considered old for today's standards. They will all be explained in due time, as well as their modern alternatives.

I'm just one guy and I would **not** be crazy enough to build a whole, modern OS from scratch by myself. I only did this project to learn the basics of how an OS truly works, but I ended up falling into a big rabbit hole and now I'm writing this whole article about this subject.

Just try to keep this in mind: this article was written with the 1980-2012 OS standards in mind. Treat everything like it unless specifically told otherwise.
# A Bit of History
The first IBM PCs around the 80s came with a very basic firmware code, only containing the BIOS (Basic Input/Output System), and its main purpose was to find a drive with a **Bootloader**.

The bootloader is the first contact of the machine with our OS code. Once we turn on the computer, the firmware starts looking for a bootable device, but how does it know when it has found one?
# Finding a Bootable Device
The **legacy BIOS** was designed with the **MBR** (Master Boot Record) drive system in mind. This drive model doesn't know what files or directories are, it simply stores data, zeroes and ones. This partition scheme used CHS (Cylinder-Head-Sector) addressing.

The goal of the legacy BIOS was to read the first **512 bytes** (AKA Sector 1) of any disk connected, until it finds the 2-byte "magic signature" at the end of the sector: `0x55AA`. If this value was found, then the current disk is a bootable device and the **bootloader** starts. This value has to be hardcoded by the developer and stored at bytes 510 and 511.

In modern machines (post-2012), a drive with **GPT** (GUID Partition Table) became the standard, which knows what files and directories are and stores them as such. Modern **UEFI** motherboards don't look for Sectors anymore, but instead specific boot files, with the `.efi` file extension.

However, until the legacy BIOS was being phased out in favor of UEFI (2020 onwards), every UEFI motherboard had a togglable configuration called **CSM** (Compatibility Support Mode), which enabled the legacy BIOS for backward compatibility with old hardware. During this time, SSDs also came with a way to "fake" CHS coordinates to match their way of storing data, essentially lying to older hardware for backwards compatibility. Some GPT drives also store a "fake" MBR to prevent old hardware from accidentally wiping the whole drive due to misunderstanding.
# Initiating the Boot
Now that our device has been successfully found by the motherboard, we need to actually start the booting sequence and load the kernel in, all in less than 512 bytes[^1]. For that, we use **Assembly**. The machine will start reading code from top-to-bottom:
> [!NOTE] Assembly, despite being a simple programming language, can be very unintuitive and hard to understand specially when trying to build an OS from scratch, as you are communicating directly to the hardware and have to follow a very strict set of rules, some intentional by design, most not.
## `boot.asm`
### 1. Setup and Constants
```asm
[org 0x7c00]
[bits 16]

KERNEL_OFFSET  equ 0x1000
KERNEL_SECTORS equ 16

CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start
```
- `[org 0x7c00]` tells the assembler where the boot.asm code itself will be loaded in memory (physical address).
- `[bits 16]` tells the CPU to read the code in 16-bit "Real Mode"[^2].

- `KERNEL_OFFSET` constant sets the memory address where the kernel will be loaded.
- `KERNEL_SECTORS`: remember how the bootloader is set on Sector 1? Well, we also have to specify how many more sectors the kernel itself occupies, with each sector having 512 bytes of space. The sector count defined can be greater than the actual number of sectors the kernel will use.

- `CODE_SEG` and `DATA_SEG` define the byte offsets for both the code and data segments inside our Global Descriptor Table (GDT), which we will need later on.
### 2. Entry Point
```asm
start:
    cli
    cld
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7c00
    sti
```
- `start:` is a label marking the entry point.
	-  `cli` clears and disables interrupts. This is useful to prevent crashes in case of an unexpected hardware trigger while we are managing (physical) memory.
	- `cld` clears the Direction Flag, ensuring string operations (like `lodsb`) process memory in the forward direction (low to high addresses).
	- `xor ax, ax` executes a XOR operation between `ax` and `ax`, which always results in `0`. This is a fast way to always set the value of `ax` to `0`. `ax` is a 16-bit register known as the **Accumulator**[^7], primarily used for mathematical operations.
	- `mov (ds, es, ss), ax` sets the **Data**[^3], **Extra**[^4] and **Stack**[^5] **Segment** registers to `0`.
	- `mov sp, 0x7c00` sets the **Stack Pointer** at `0x7c00`. The stack grows downwards, so it will never collide with our bootloader code.
	- `sti` re-enables interrupts.
### 3. Saving the Drive
```asm
mov [BOOT_DRIVE], dl

mov ax, 0x0003
int 0x10
```
- `mov [BOOT_DRIVE], dl`: the BIOS stores the ID of the drive we booted from in the `dl` register. We save it to a variable for later use.

- `mov ax, 0x0003` and `int 0x10` cause a BIOS interrupt that clears the screen and forces the video card into standard 80x25 text mode.
### 4. Core Execution Flow
```asm
call load_kernel
call enable_a20
```
- `call load_kernel` reads the disk and loads the kernel into memory. ([[#6. Loading the Kernel]])
- `call enable_a20` enables the A20 line. ([[#7. Unlocking More RAM]])
### 5. Entering Protected Mode
```asm
cli
lgdt [gdt_descriptor]

mov eax, cr0
or eax, 0x1
mov cr0, eax

jmp CODE_SEG:init_pm
```
- `cli` disables interrupts again.
- `lgdt [gdt_descriptor]` loads the GDT, defining the memory layout for 32-bit mode.

- `mov eax, cr0`: `eax` works just like `ax`, but it uses 32-bit instead of 16-bit. `cr0` refers to the CPU's Control Register 0. This line copies the current value of `cr0` into the `eax` Accumulator.
- `or eax, 0x1`: this performs a bitwise **OR** operation between the value stored in `eax` and `0x1`, which always sets the lowest bit to `1`.
- `mov cr0, eax`: finally, we write the modified value back to the CPU. This is the exact moment the CPU becomes a 32-bit processor (Protected Mode[^6]).

- `jmp CODE_SEG:init_pm` jumps to our 32-bit code. ([[#9. 32-Bit]])
### 6. Loading the Kernel
```asm
load_kernel:
    xor ax, ax
    mov es, ax
    mov bx, KERNEL_OFFSET

    mov ah, 0x02
    mov al, KERNEL_SECTORS
    mov ch, 0x00
    mov cl, 0x02
    mov dh, 0x00
    mov dl, [BOOT_DRIVE]
    int 0x13

    jc disk_error
    cmp al, KERNEL_SECTORS
    jne disk_error

    ret
```
- `xor ax, ax` resets `ax` to `0`, then `mov es, ax` sets the Extra Segment to `0`.
- `mov bx, KERNEL_OFFSET` sets the destination address in RAM where the read data will be written.

- `mov ah, 0x02`: when `int 0x13` is executed, the BIOS looks at `ah` to know what to do. `0x02` means "Read Sectors Into Memory".
- `mov al, KERNEL_SECTORS`: `al` tells the BIOS how many sectors to read. We read all kernel sectors in a single call.
- `mov ch, 0x00`, `mov cl, 0x02`, `mov dh, 0x00` and `mov dl, [BOOT_DRIVE]`: this sets up the CHS physical disk address. `ch` sets the Cylinder to `0`. `cl` sets the starting Sector to `2` (the sector immediately after our bootloader). `dh` sets the Head to `0`. `dl` passes the ID of the booted drive.
- `int 0x13`: all the instructions are passed to physically execute the disk read.

- `jc disk_error`: `jc` means "Jump if Carry". During the disk read, if any error occurs, the CPU's Carry Flag is flipped to `1` by the BIOS. If that occurs, this line jumps to `disk_error`.
- `cmp al, KERNEL_SECTORS` and `jne disk_error`: after a successful read, `al` contains the number of sectors actually read. We compare it against the expected count. If they don't match (`jne` = Jump if Not Equal), we also jump to `disk_error`.
- `ret` returns to wherever `load_kernel` was invoked.
### 7. Unlocking More RAM
```asm
enable_a20:
    in al, 0x92
    or al, 00000010b
    out 0x92, al
    ret
```
This code is needed for the CPU to have full access of the machine's RAM, essentially flipping a bit in the motherboard, opening the physical logic gate and turning the A20 switch "ON".

Without this little trick, our OS code would only be able to use up to 1 Megabyte of RAM, which was the default Intel defined for the old CPUs. After some time, the IBM team discovered this trick and for a long time, this part of the code was essential for any OS. That is, until UEFI came in and Intel finally decided to physically remove this pin from newer CPUs.

However, for some years, this was a digital pin, meaning it was not physically in the CPU, but the configuration still existed and had to be flipped. Since I'm running everything inside QEMU, which simulates the original old hardware behavior, I'm keeping this.

Fun fact: back in the day, Intel promised all their hardware would be backwards compatible, that's why it took so long for them to fix this issue. After 20 years, they finally broke their promise, removing the A20 pin from their CPUs.
### 8. Printing and Error Handling
```asm
print_string:
    lodsb
    cmp al, 0
    je .done

    mov ah, 0x0e
    int 0x10
    jmp print_string

.done:
    ret
```
- `lodsb` loads the byte pointed to by `si` (Source Index) into `al`, then automatically increments `si` to the next byte (thanks to the `cld` we set earlier).
- `cmp al, 0` and `je .done`: checks if the current character is a null terminator (`0`). If so, the string is finished and we jump to `.done`.
- `mov ah, 0x0e` and `int 0x10`: BIOS interrupt for "Teletype Output". It prints the character stored in `al` to the screen.
- `jmp print_string` loops back to process the next character.

```asm
disk_error:
    xor ax, ax
    mov ds, ax
    xor bx, bx
    mov si, disk_error_msg
    call print_string

.hang:
    jmp .hang

disk_error_msg db 13, 10, "Disk read error.", 13, 10, 0
BOOT_DRIVE db 0
```
- If a disk error occurs, this routine resets the segment registers, loads the address of the error message string into `si`, and calls `print_string`.
- `.hang: jmp .hang` is an infinite loop that intentionally freezes the machine, since there is no recovery from a failed disk read.
- `disk_error_msg` stores the error string. `13, 10` is a CR+LF newline, and `0` is the null terminator.
- `BOOT_DRIVE` is a 1-byte variable that stores the drive ID the BIOS provided at boot.
### 9. 32-Bit
```asm
[bits 32]

init_pm:
    mov ax, DATA_SEG
    mov ds, ax
    mov ss, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    mov ebp, 0x90000
    mov esp, ebp

    jmp KERNEL_OFFSET
```
- `[bits 32]` tells the assembler to generate 32-bit code instead of the default 16-bit. Even though we already changed the CPU to 32-bit mode, we also have to specify to the compiler which code is 16-bit and 32-bit.
- `mov ax, DATA_SEG`: loads the offset of the data segment inside the GDT to `ax`.
- `mov .., ax`: copies the `ax` value into all the data-related segment registers.

- `mov ebp, 0x90000` and `mov esp, ebp`: sets the stack for our kernel code to store variables starting from `0x90000`.

- `jmp KERNEL_OFFSET`: it's time to say goodbye to our bootloader. This line tells the CPU to start executing the code it finds at the `KERNEL_OFFSET` address. We use `jmp` instead of `call` because the kernel will never return to the bootloader, so there is no need to save a return address. Our bootloader's job is done.
### 10. Global Descriptor Table
```asm
gdt_start:
gdt_null:
    dq 0

gdt_code:
    dw 0xffff
    dw 0x0000
    db 0x00
    db 10011010b
    db 11001111b
    db 0x00

gdt_data:
    dw 0xffff
    dw 0x0000
    db 0x00
    db 10010010b
    db 11001111b
    db 0x00

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1
    dd gdt_start
```
>Changing the CPU from 16-bit (Real Mode) to 32-bit (Protected Mode) is a big conceptual shift. From now on, the code no longer refers to memory as a physical address but as an index to a valid and secure place. You might also be wondering what the hell is the GDT (Global Descriptor Table).
>
Until now, all the segment registers stored physical pieces of memory address. It was basically raw math both to write and read memory, but there was **no security**. Any program could set a segment register value to anything and overwrite any part of the code. Entering 32-bit Protected Mode helps to prevent that from happening by using the **Global Descriptor Table**.

>Now, each segment register holds a **Selector**, which is just an index number pointing to a row in a highly secure database table (the GDT) that holds information like the **Base Address** and **Size Limit** on each row, referring to different indexes.
>
The GDT for this OS has a **Flat Memory Model** set up, meaning both the code and data segments start at address `0x00000000` and span the full 4 GB of addressable memory. This effectively gives the kernel unrestricted access to all memory. This is the same model used by Linux and most modern OSs. In those systems, the actual memory protection is handled by a different mechanism called paging, not by segmentation.
- `dq 0`: the x86 CPU hardware mandates that the very first entry in the GDT must be completely empty. If any code accidentally loads a null selector (index `0`) into a segment register, the CPU instantly triggers a fault instead of letting the program access random memory. `dq` means Define Quad-word, which uses 8 bytes (**every GDT entry is 8 bytes long**).
- `gdt_code:`
	- `dw 0xffff` this value is the first part of the maximum size (Limit, bits 0-15).
	- `dw 0x0000` and `db 0x00` set the Base Address (bits 0-15 and 16-23) to `0x00000000`, which is the absolute bottom of RAM.
	- `db 10011010b`: this is the security heart of the `gdt_code` area. Reading each bit from left to right:
		- `1` (Present): tells the CPU this segment is valid and currently in memory.
		- `00` (Privilege): this is **Ring 0**, the highest kernel-level privilege. If user-mode applications (Ring 3) try to access this, the hardware firewall blocks it.
		- `1` (Descriptor Type): `1` means this is either code or data.
		- `1` (Executable): yes, this is a code segment.
		- `0` (Conforming): code in lower privilege rings cannot jump into this segment.
		- `1` (Readable): allows the CPU to read the code instructions. (Code segments are never writable).
		- `0` (Accessed): the CPU will flip this to `1` when it uses the segment.
	- `db 11001111b`: this byte is split in half:
		- Right half (`1111`): the last 4 bits of the Limit. Combined with the `0xffff` from earlier, our total limit is `0xfffff`.
		- Left half (`1100`): represents the flags. The first bit (`1`) is the Granularity flag. It tells the CPU to multiply our limit by 4 Kilobytes (`0xfffff * 4096 = 4 Gigabytes`). This means we can address 4 GB of RAM. The second bit (`1`) is the Size flag. It tells the CPU this is a 32-bit segment.
	- `db 0x00`: the final piece of the Base Address (bits 24-31), cementing it at a flat `0x00000000`.
- `gdt_data:`
	- This **seems** like an almost exact copy of the Code Segment, with the only visible difference being a single bit. However, flipping that specific bit (Executable bit) also **changes the definition of the bits next to it**, so essentially it was a change of 3 bits instead of 1:
		- The actual bit flipped (Executable): flipped to `0`, meaning this is not executable code, but data.
		- Direction bit (3rd from the right): `0` means that this data segment grows upwards in memory.
		- Writable bit (2nd from the right): `1` means the CPU is allowed to write variables into this block.

### 11. Magic Signature
```asm
times 510 - ($ - $$) db 0
dw 0xaa55
```
- `times 510 - ($ - $$) db 0`: Remember how we are limited to 512 bytes and needed a "Magic Signature" at the last 2 bytes? This line of code makes sure our current bootloader code occupies exactly 510 bytes, filling the remaining space (if any), with zeroes.
- `dw 0xaa55`: finally, write the "Magic Signature" bytes at the very end of our bootloader, making our drive recognized as a bootable device.
As you read at the beginning of this article, the machine needs to read the byte values `55 AA`, so why are we writing `0xaa55`?

That's because x86 CPUs use **little-endian** byte ordering, meaning the least significant byte is stored first in memory. When we write `dw 0xaa55` (Define Word), the assembler stores it as the two bytes `55 AA` in memory, which is exactly what the BIOS expects to find.
## `kernel_entry.asm`
The bootloader jumps to `KERNEL_OFFSET` (`0x1000`), but our kernel is written in C. We can't jump directly into C code because the CPU needs a tiny piece of Assembly to know *which* C function to run first. Think of it as a receptionist that greets you at the door and tells you where to go. `kernel_entry.asm` is that receptionist: it's the **bridge** between the bootloader and the C kernel's `kernel_main` function.
```asm
[bits 32]

section .text.entry

global _start
extern kernel_main

_start:
    call kernel_main

.hang:
    hlt
    jmp .hang

section .note.GNU-stack noalloc noexec nowrite progbits
```
- `[bits 32]`: this code runs in 32-bit Protected Mode, matching the state the bootloader left the CPU in.
- `section .text.entry`: places this code in a special section name. The linker script is configured to place `.text.entry` **before** all other code, guaranteeing that `_start` sits at the very beginning of the kernel binary (address `0x1000`). This is critical because the bootloader blindly jumps to `0x1000`, so whatever is there must be valid executable code.
- `global _start`: exports the `_start` symbol so the linker can find it and use it as the entry point.
- `extern kernel_main`: declares that `kernel_main` is defined elsewhere (in `kernel.c`), allowing the linker to resolve the reference.
- `call kernel_main`: transfers execution to the C kernel. Unlike `jmp`, `call` pushes a return address onto the stack, so if `kernel_main` ever returns, execution continues at `.hang`.
- `.hang`: an infinite loop using `hlt` (Halt). The `hlt` instruction puts the CPU into a low-power sleep state. The `jmp .hang` ensures that even if something wakes the CPU, it goes back to sleep. This is a safety net. A kernel should never return, but if it does, the machine freezes cleanly instead of executing random garbage from memory.
- `section .note.GNU-stack noalloc noexec nowrite progbits`: a metadata section that marks the stack as **non-executable**. This is a security measure that prevents malicious code from being executed if it somehow ends up in the stack memory.
## `linker.ld`
When we compile our code, we get separate `.o` files (one for `kernel_entry.asm`, one for `kernel.c`). The linker script tells the computer how to **glue** those pieces together into a single binary, and most importantly, *where* in memory each piece should go.
```ld
OUTPUT_FORMAT(elf32-i386)
OUTPUT_ARCH(i386)
ENTRY(_start)

SECTIONS
{
    . = 0x1000;

    .text : {
        *(.text.entry)
        *(.text*)
    }

    .rodata : {
        *(.rodata*)
    }

    .data : {
        *(.data*)
    }

    .bss : {
        *(COMMON)
        *(.bss*)
    }

    /DISCARD/ : {
        *(.comment*)
        *(.eh_frame*)
        *(.note*)
    }
}
```
- `OUTPUT_FORMAT(elf32-i386)`: tells the linker to produce a 32-bit ELF file for our CPU type. ELF is just a container format (like a `.zip` for code). Later, we strip the container away with `objcopy` to get a raw binary the CPU can run directly.
- `OUTPUT_ARCH(i386)`: confirms we are targeting a 32-bit x86 CPU.
- `ENTRY(_start)`: tells the linker that `_start` (from `kernel_entry.asm`) is where execution begins. This is mostly useful for debugging tools.

- `. = 0x1000`: this is the **location counter**. It tells the linker: "start placing code at memory address `0x1000`". This **must** match the `KERNEL_OFFSET` in `boot.asm`, because that's where the bootloader expects to find the kernel. Since our GDT uses a Flat Memory Model (base = `0`), the addresses in this file directly correspond to physical RAM addresses.

- `.text` (Code Section):
	- `*(.text.entry)` is listed **first**, so `kernel_entry.asm`'s `_start` ends up at the very beginning of the binary (address `0x1000`). This is what makes the bootloader's `jmp KERNEL_OFFSET` land on valid code.
	- `*(.text*)` collects all other compiled code.

- `.rodata` (Read-Only Data): stores constants like string literals and the ASCII art. This data cannot be modified at runtime.

- `.data` (Initialized Data): stores global/static variables that have an explicit initial value.

- `.bss` (Uninitialized Data):
	- `*(COMMON)` collects global variables that were declared but not given a value.
	- `*(.bss*)` collects all other uninitialized variables. The `.bss` section takes up no space in the actual binary file on disk. The memory is simply zeroed out when loaded.

- `/DISCARD/`: removes junk sections that the compiler adds by default but we don't need:
	- `.comment`: compiler version strings.
	- `.eh_frame`: exception handling tables (we don't use exceptions).
	- `.note`: miscellaneous metadata.
	Discarding these keeps our kernel binary small and avoids wasting disk sectors.
## `kernel.c`
The kernel is where the actual OS logic lives. However, this is **not** a normal C program. When you normally write C, there's a whole OS underneath handling things for you (memory, screen, keyboard, etc.). Here, there is nothing. No OS underneath, no libraries, nothing. Our code talks directly to the hardware. This is called a **freestanding** environment.
### Hardware Interaction
The CPU has numbered "ports" that connect to different hardware devices (keyboard, screen, etc.), kind of like numbered mailboxes. The kernel uses two tiny functions to talk to these ports:
- `inb(port)`: reads one byte from a hardware port (like checking a mailbox).
- `outb(port, value)`: writes one byte to a hardware port (like putting a letter in a mailbox).

These functions use inline assembly because only raw CPU instructions can access ports. They are used to:

- **Control the VGA cursor**: writing to ports `0x3D4`/`0x3D5` moves the blinking text cursor on screen.
- **Read keyboard input**: port `0x64` tells us if a key was pressed, port `0x60` tells us *which* key was pressed.

### VGA Text Mode
The screen is controlled by writing directly to a special region of RAM that the video card is always watching, starting at address `0xb8000`. This is called the VGA text buffer. Each character on screen is represented by 2 bytes:
- Byte 0: the ASCII character code.
- Byte 1: the color attribute (foreground + background).

With an 80x25 character grid, the full screen is `80 * 25 * 2 = 4000` bytes of memory. Writing a byte to address `0xb8000` changes the top-left character on screen instantly. No function calls, no drivers, no OS layer in between.

### Program Structure
The kernel implements a minimal interactive shell:
1. **Boot screen**: displays ASCII art and a welcome message on startup.
2. **Command prompt**: prints `bingOS> ` and waits for keyboard input.
3. **Input handling**: converts PS/2 scancodes to ASCII characters using a lookup table, supports backspace, and buffers up to 32 characters.
4. **Command execution**: on Enter, the input buffer is compared against known commands (`help`, `bingus`) using a custom `string_equals` function. Unknown commands print an error message.

The kernel runs an infinite loop (`for (;;)`) that continuously checks the keyboard for new keys. This technique is called **polling**: the CPU keeps asking "is there a key yet?" over and over. The recommended approach in real OSs is to use **interrupts**, where the keyboard *notifies* the CPU only when a key is pressed, letting the CPU do other work in the meantime. Polling was used here because it's much simpler to implement and good enough for this project.

The kernel never returns. If it somehow did, `kernel_entry.asm`'s `.hang` loop would catch it and safely freeze the machine.

[^1]: In modern systems with UEFI firmware, this byte limit doesn't exist.
[^2]: The CPU 16-bit Real Mode is the default mode your CPU "wakes up" when your machine is booted. This mode comes with a lot of limitations, since it was used for computers in the 80s. For example, it can only "see" 1 Megabyte of RAM, has no security at all and any program can overwrite any memory address.
[^3]: The Data Segment (`ds`) is the default hardware "window" the CPU looks through whenever your code asks to read or write standard variables into memory.
[^4]: The Extra Segment (`es`) is similar to the Data Segment, but it's mostly used as a "destination" window, so the CPU can quickly copy large chunks of data from one place to another.
[^5]: The Stack Segment (`ss`) is the hardware window dedicated entirely to the Stack: the temporary scratchpad where functions store their local variables and remember where they need to return when they finish running.
[^6]: The Protected Mode is the upgrade from the 16-bit mode to a more secure 32-bit one. It unlocks access to 4 Gigabytes of RAM and enforces the use of GDT, strictly defining what memory can be read, written or executed.
[^7]: The General-Purpose Registers are used by the CPU to do fast math and hold temporary values, labeled as: `ax` (Accumulator, for math), `bx` (Base, for memory addresses), `cx` (Counter, for loops) and `dx` (Data, for hardware I/O). There are also the Pointers: `si` (Source Index), `di` (Destination Index), `bp` (Base Pointer) and `sp` (Stack Pointer). In 32-bit mode, they all start with an "`e`" for Extended.
