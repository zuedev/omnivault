# Modern Warbird, A Deep Dive into Dynamic Code Obfuscation

## Introduction

At first glance, Warbird in modern Windows might seem like a straightforward system. However, behind its seemingly simple exterior lies a clever array of techniques designed to keep prying eyes at bay. In this exploration, we'll dive into the real-world obfuscation methods observed in actual binaries—revealing the art behind encrypting and decrypting code on the fly.

## Encryption Segments

One of Warbird’s standout features is its use of encryption segments. Instead of safeguarding individual functions, Warbird groups sets of functions into “segments” and encrypts them collectively at runtime. These segments aren’t necessarily logically related—functions in one segment may call upon those in another—but this cross-referencing is part of the strategy to prevent an attacker from simply dumping all decrypted code in one go.

In binaries that leverage these segments (take, for instance, `sppsvc.exe`), you'll notice multiple sections tagged as `?g_Encry`. Each section comes with an `ENCRYPTION_SEGMENT` structure that lays out the details of the segment. Here’s a peek into that structure:

```c
typedef struct _FEISTEL_ROUND_DATA {
    DWORD round_function_index; // Index of function used for round
    // Parameters for round functions
    DWORD param1;
    DWORD param2;
    DWORD param3;
} FEISTEL_ROUND_DATA;

typedef struct _ENCRYPTION_BLOCK {
    DWORD flags;
    DWORD rva;
    DWORD length;
} ENCRYPTION_BLOCK;

struct PRIVATE_RELOCATION_ITEM
{
    ULONG rva: 28;
    ULONG reloc_type: 4;
};

typedef struct _ENCRYPTION_SEGMENT {
    BYTE hash[32]; // SHA-256 hash of all segment data that follows
    ULONG segment_size;
    ULONG _padding0;
    ULONG segment_offset;
    ULONG reloc_table_offset; // Offset to a global table of PRIVATE_RELOCATION_ITEMs
    ULONG reloc_count; // Length of table
    ULONG _padding1;
    UINT64 preferred_image_base; // Preferred base address of PE as specified in headers
    ULONG segment_id; // Internal segment ID as described in Warbird configuration
    ULONG _padding2;
    UINT64 key; // Feistel cipher key
    FEISTEL_ROUND_DATA rounds[10];
    DWORD num_blocks; // Length of blocks array
    ENCRYPTION_BLOCK blocks[1]; // Blocks of data within module to be decrypted
} ENCRYPTION_SEGMENT;
```

At runtime, the magic happens via a system call to `NtQuerySystemInformation`, which decrypts these segments on demand. The decryption process is driven by the following argument structure:

```c
typedef struct _ENCRYPTION_SEGMENT_ARGS {
    UINT64 operation; // 1 (decrypt) or 2 (re-encrypt)
    PVOID segment; // Pointer to ENCRYPTION_SEGMENT
    PVOID base; // Base address of module
    UINT64 preferred_base; // Same as segment preferred_image_base
    PVOID reloc_table; // Address of the same table referred to by reloc_table_offset
    UINT64 reloc_count; // Number of relocations in global table
} ENCRYPTION_SEGMENT_ARGS
```

A call like this—where `operation` is set to `1`—decrypts the code in-place, allowing the binary to execute as usual. Later, when the encrypted functions are no longer needed, a similar call (with `operation` set to `2`) re-encrypts the segment, keeping the sensitive code under wraps.

## Heap Executes

While encryption segments add a robust layer of protection, they aren’t impervious. An attacker might still set a breakpoint within the encrypted code, wait for it to be called, dump the memory, and piece together the decrypted functions. To counter such tactics, Warbird introduces an even more dynamic technique: heap execute.

Heap executes work by allocating the decrypted code in the process heap before jumping to it. This method effectively sidesteps simple breakpoint strategies and complicates control flow analysis. Typically, you’ll find the related structures near the start of the `.text` section. Consider this structure, aligned to 8 bytes, which is designed to handle the decryption of a single function:

```c
// Struct is aligned to 8 bytes
typedef struct _HEAP_EXECUTE {
    BYTE hash[32];
    ULONG hexec_size;
    ULONG virt_stack_limit;
    ULONG hexec_offset:28;
    ULONG _padding0;
    ULONG checksum:8; // Unused
    ULONG unused0:8; // Unused
    ULONG rva:28;
    ULONG size:28;
    ULONG unused1:28; // Unused
    ULONG unused2:28; // Unused
    UINT64 key;
    FEISTEL_ROUND_DATA rounds[10];
} HEAP_EXECUTE;
```

When a heap execute is triggered via `NtQuerySystemInformation`, the decrypted code is placed in the heap, and execution jumps directly to it. Here’s a snapshot of the heap and stack layout during this process:

**Heap Layout:**

```
rip - 0x10: Call offset
rip - 0x08: NtQuerySystemInformation syscall number (usually 0x36)
rip: Decrypted code
```

**Stack Layout (with rsp as the stack pointer):**

```
rsp: Heap execute struct address
rsp + 0x08: Argument 1
rsp + 0x10: Argument 2
rsp + 0x18: Argument 3
...
```

The approach introduces some intriguing quirks. For instance, all external function calls must be adjusted by an offset stored on the heap. To illustrate, a call to `GetProcAddress` might be implemented as follows:

```asm
lea rax, GetProcAddress
call [rdx+rax]
```

For clarity during analysis, an instruction like `lea rdx, [rip - 7]` at the start of the heap-executed code can be replaced with `xor rdx, rdx`—a tweak that simplifies decompilation without undermining the underlying obfuscation. Additionally, calls to `NtQuerySystemInformation` are replaced with `syscall` instructions, with registers set up in a very particular fashion:

```
rax = syscall number from heap
r10d = 0xB9
rdx = address of arguments
r8d = size of arguments
r9 = 0
```

Before the function returns, a final system call resets the stack frame, ensuring a smooth transition back to the normal execution flow.

## Kernel-Level Decryption

Digging even deeper, when `NtQuerySystemInformation` is invoked with a `SYSTEM_INFORMATION_CLASS` value of `0xB9`, the process is handed off to a function called `WbDispatchOperation`. Depending on the operation value embedded in the arguments, different decryption or re-encryption tasks are performed. Here's a quick overview of the available operations:

|Operation Number|Operation|
|---|---|
|0|No-op|
|1|Decrypt encryption segment|
|2|Re-encrypt encryption segment|
|3|Heap execute|
|4|Heap execute return|
|5|Unused|
|6|Unused|
|7|Remove process|
|8|Unload module|

Each routine ensures that decryption is only performed on read-only executable pages belonging to binaries signed as a “Windows System Component.” This extra measure—akin to well-known exploit mitigations—provides an additional layer of defense, though a determined reverse engineer might bypass it by mapping a signed binary into their own process space.

At the heart of the decryption process lies the function `WarbirdCrypto::CCipherFeistel64::CallRoundFunction`. This mysterious routine, invoked by an undisclosed Feistel decryption routine, processes the following parameters:

```c
void FeistelDecrypt(
    FEISTEL_ROUND_DATA* rounds,
    BYTE* input,
    BYTE* output,
    ULONG length,
    UINT64 key,
    ULONG iv,
    BYTE* checksum
);
```

Armed with the knowledge of this function’s location, a reverse engineer could theoretically use the `ntoskrnl.exe` binary to decrypt Warbird-encrypted code—a challenge that remains an enticing puzzle for the more intrepid among us.
