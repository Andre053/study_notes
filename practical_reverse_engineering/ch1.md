# Chapter 1: x86 and x64
## Intro
- x86 is the 32-bit implementation of the Intel architecture (IA-32)
	- Defined in the Intel Software Development Manual
- Can operate in real or protected mode
	- Real-mode supports 16-bit instructions, is used when first powered on
	- Protected-mode supports 32-bit instructions, alongside virtual memory, paging, and other features
		- OS execute in protected-mode
- x86-64 is teh 64-bit extension of this architecture
- Privilege separation done through *ring levels*
	- Levels are 0-3, with 0 being the highest privileged
	- Typicaly only levels 0 and 3 are used, for kernel-space and user-space respectively
	- Ring level is encoded in the *cs* register, sometimes referred to as the current privilege level (CPL)
## Register Set and Data Types
- When in protected mode, x86 has eight 32-bit general purpose registers (GPRs)
	- EAX, EBX, ECX, EDX, EDI, ESI, EBP, and ESP
	- Some can be further divided into 8-bit and 16-bit registers
		- EAX divides into AX which divides into AH and AL
		- EBP divides into BP
		- ESI divides into SI
		- ESP divides into SP
		- EDI divides into DI
	- Common usage
		- ECX: Counter in loops
		- ESI: Source in string/mem operations
		- EDI: Destination in string/mem operations
		- EBP: Base frame pointer
		- ESP: Stack pointer
- Instruction pointer is stored in the EIP register
- Common data types
	- Bytes: 8 bits. Such as AL, BL, and CL
	- Word: 16 bits: Such as AX, BX, and CX
	- Double word: 32 bits. Such as EAX, EBX, and ECX
	- Quad word: 64 bits. Done by combining two registers, usually EDX:EAX
- The EFLAGS register stores the status of arithmetic operations and other execution states (e.g. trap flags)
	- 32-bit in size
	- ZF flag set to 1 is previous add operation resulted in zero
- Additional regiters that control low-level system mechanisms exist
	- Such as virtual memory, interrupts, and debugging
	- CR0 controls whether paging is on or off
	- CR2 contains linear address that caused a page fault
	- CR3 is the base address of a paging data structure
	- CR4 controls hardware virtualization settings
	- DR0-DR7 are used to set memory breakpoints. Only DR0-DR3 are usable, the others are for status
- Model-specific registers (MSRs) vary between processors
	- Read/written to through RDMSR/WRMSR instructions
	- Accessible only to code running in ring 0
	- Typically store special counters and implement low-level functionality
## Instruction Set
- Five general methods for data movement
	- Immediate to register
	- Register to register
	- Immediate to memory
	- Register to memory (and vice versa)
	- Memory to memory
		- Specific to x86, simplifying instructions
- ARM does not support memory to memory instructions and only supported fixed-length instructions of 2 or 4 bytes
	- x86 supports 1-16 byte length instructions
- Syntax
	- Intel: instruction dest, src
		- Memory access utilizes square brackets
	- AT&T: instruction src, dest
		- Instructions specify the operation width as a postfix
		- Registers are prefixed with %, immediates prefixed with $
- Data movement
	- Instructions operate on values which come from registers or main memory
	- Common instructions
		- mov: Moves data to a register or memory
		- movsb/movsw/movsd: Moves data with specified granularity between two memory addresses
			- Implicitly use EDI as dest and ESI as source
			- Automatically update source and destination address depending on the direction flag (DF) in EFLAGS
				- If DF is 0, the addresses are decremented, otherwise incremented
			- Typically used to implemnet string or memory copy functions when length is known at compile time
		- rep: Repeats instruction up to ECX times
		- lea: Evaluates expression and puts the result in the destination register
			- Uses square brackets but does not access memory (only exception to the rule)
		- scas: Implicitly compares AL/AX/EAX with data starting at mem address EDI
			- Automatically increments/decrements EDI depending on the DF bit in EFLAGS
			- Commonly used with REP to find a byte, word, or double-word in a buffer (ex. strlen() to find null byte)
		- stos: Writes the value AL/AX/EAX to EDI
			- Automatically increments/decrements EDI depending on the DF bit in EFLAGS 
			- Commonly used to initialize a buffer to a constant value (ex. memset())
		- lods: Reads value of specified granularity from ESI and stores into AL, AX, or EAX
	- Offsets can be a register or immediate
		- Often used to access structure members or data buffers at locations computed at runtime
		- General format: base + index * scale
		- Often used for loops
	- By moving longer pieces of data into memory, you can set multiple structure/buffer elements with a single instruction
		- Done by compilers to optimize execution
	- Can repeat movs instructions or use a combination of them depending on the amount of data that needs to be copied

## Exercise 1
This function uses a combination SCAS and STOS to do work. Explain what is the type of [EBP+8] and [EBP+C]. Explain what the snippet does.	

01: 8B 7D 08		mov		edi, [ebp+8]	; stores [base ptr + 8] into edi
02: 8B D7			mov		edx, edi		; saves edi into edx	
03: 33 C0			xor		eax, eax		; clears eax
04: 83 C9 FF		or		ecx, 0FFFFFFFFh ; sets ecx to all 1s, scasb would run for a long time without a conditional stop
05: F2 AE			repne	scasb			; repeats scasb, starts at edi and compares to eax which is zero -> finds zero location
06: 83 C1 02		add		ecx, 2			; adds two to ecx which was all 8 Fs
07: F7 D9			neg		ecx				; replaces ecx with it's two's complement, setting it to 1
08: 8A 45 0C		mov		al, [ebp+0Ch]	; stores [base pointer + C] into al which is an 8-bit register
09: 8B FA			mov		edi, edx		; stores edx which contains location of first zero into edi
10: F3 AA			rep		stosb			; runs stosb, which is [base ptr + C] twice (val of ecx), setting locations to the value of al
11: 8B C2			mov		eax, edx		; stores edx, the location of the initial zero + 2, into eax

Answer:
	[EBP+8] access memory at address EBP + 8 bytes and is stored in EDI which is a 32-bit register, seems to be the location of a string or similar array 
	[EBP+C] accesses memory at address EBP + C bytes and is stored in AL which is an 8-bit register, seems to be location of a setter value
	This snippet stores the location of an array, searches it for zeros, returns the first zero location, then sets the zero location to an 8-bit value stored at [EBP+C], stores the final location into EAX

## Arithmetic Operations
- Fundamental arithmetic operations are natively supported by x86 instruction set
- Bit-level operations like AND, OR, XOR, NOT, and left/right shift have native instructions
- Left and right shifts optimize multiplication and division operations where the multiplicand and divisor are a power of two
	- This optimization is known as *strength reduction*
- Unsigned and signed multiplication is done through MUL and IMUL respectively
	- MUL can only operate on register or memory values, and is multiplied with either AL, AX, or EAX, stored in AX, DX:AX, or EDX:EAX
	- IMUL has three forms:
		- IMUL reg/mem
		- IMUL reg1, reg2/mem
		- IMUL reg1, reg2/mem, imm		
- Unsigned and signed division is done through DIV and IDIV respectively
	- Only take a divisor as a param
	- Will use AX, DX:AX, or EDX:EAX as the dividend
	- Resulting quotient/remainder pair stored in AL/AH, AX/DX, or EAX/EDX

## Stack Operations and Function Invocation
- The stack is fundamental for programming languages and OS
	- Local variables are stored on a functions' stack space
	- The OS stores state information on the stack when changing ring levels
- A stack is a contiguous memory region that grows downward
	- Pointed to by ESP
- Operations
	- PUSH decrements ESP and writes data at the location pointed to by ESP
	- POP reads data and increments ESP
	- Note: Default auto-increment is 4, but can be changed to 1 or 2; in practice, OS requires double-word aligned (4)
- ESP can be directly modified by other instructions like ADD and SUb
- Programming functions are implemented using the stack data structure
	- CALL: Pushes return address on the stack and changes EIP to the call destination
	- RET: Pops the address stored on the top of the stack into EIP 
	- A *calling convention* is a set of rules dictating how function calls work at the machine level
		- Defined by the Application Binary Interface (ABI)
		- Specifies how params should be passed, return value stored, etc.
		- Popular calling conventions:
			- CDECL: Params pushed on stack from right-to-left, caller must clean up. Return stored in EAX. Non-vol registers are: EBP, ESP, EBX, ESI, EDI
			- STDCALL: Same as CDECL but callee must clean up. Return stored in EAX. Non-vol registers are: EBP, ESP, EBX, ESI, EDI
			- FASTCALL: First two params are passed in ECX and EDX, the rest on the stack. Return stored in EAX. Non-vol registers are: EBP, ESP, EBX, ESI, EDI
	- Function prologue establishes a new function frame
		- Pushes EBP on the stack and sets EBP to the current stack pointer
	- EBP when used to access params is known as the base frame pointer
		- Compiler can generate code without using EBP with optimization called *frame pointer omission*
		- This allows local vars and params to be accessed relative to ESP, so EBP can be used as a general register (increasing resources available)
	- Function epilogue concludes the function and returns to previous function frame
		- Sets the stack pointer to the base frame pointer and pops the saved EBP into EBP
		- Top of the stack now contains the return address
	- Cleanup can be done by shrinking the stack by the size of the previous function frame

## Exercises
1. Given what you learned about CALL and RET, explain how you would read the value of EIP? Why can't you just do MOV EAX, EIP?
2. Come up with at least two code sequences to set EIP to 0xAABBCCDD
3. In the example function, addme, what would happen if the stack pointer were not properly restored before executing RET?
4. In all the calling conventions explained, the return value is stored in a 32-bit register (EAX). What happens when the return value does not fit in a 32-bit register? Write a program to experiment and evaluate your answer. Does the mechanism change from compiler to compiler?






