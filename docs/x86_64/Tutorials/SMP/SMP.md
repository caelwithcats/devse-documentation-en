# Symmetric Multi Processing

## Introduction

What is SMP?

The SMP stands for symmetric multi processing

We use this term to mean multi processor. A kernel that supports SMP can have a huge performance boost. Knowing that __generally__ processors have 2 threads per cpu, for an 8 core processor we have 16 exploitable threads.

The SMP is different from NUMA, the NUMA processors are processors where some cores do not have access to all of the memory.

In this tutorial to implement the smp we take into account that you have already implemented in your kernel:

- [IDT](devse-documentation-en/x86_64/Structures/IDT)
- [GDT](devse-documentation-en/x86_64/Structures/GDT)
- [MADT](devse-documentation-en/x86_64/Devices/MADT)
- [LAPIC](devse-documentation-en/x86_64/Devices/LAPIC)
- [APIC](devse-documentation-en/x86_64/Devices/APIC)
- paging
- Your kernel is higher half
- Your kernel is 64-bit
- A timer system

You will be necessary to implement the interrupt [APIC] (devse-documentation-en/x86_64/Devices/APIC) for the other CPUs, which is not covered in this tutorial (for now)

## Get Current CPU Number

Obtaining the current CPU number will be very important for later.

to obtain the identifier number of the current CPU we must use the [APIC](devse-documentation-en/x86_64/Devices/APIC)

we must read in the apic at register 20
then we must shift the bits to 24

```cpp
// LAPIC_REGISTER = 20
uint32_t get_current_processor_id()
{
    return apic::read(LAPIC_REGISTER) >> 24;
}
```


## Obtain the Local APIC Entries

see: [LAPIC](devse-documentation-en/x86_64/Devices/LAPIC/)

To start the SMP you have to get the LAPIC entries from the MADT table

Each cpu has a LAPIC entry.
The number of cpu is therefore the number of LAPICs in the MADT.

LAPIC input 2 is important

__ACPI_ID__: used for acpi

and

__APIC_ID__: used for APIC, during initialization

__usually ACPI_ID and APIC_ID are equal__


it must be taken into account that the main cpu (the one that is booted at startup) is also in the list.
We must then separate this entry by comparing if the number of the current cpu is equal to the cpu number of the local APIC entry

```cpp
if(get_current_processor_id() == lapic_entry.apic_id)
{
    // then it's the main cpu
}
else
{
    // a cpu that we can use!
}
```

## Pre-Initialization

Before initializing the cpu, the ground must be prepared.

You have to prepare where you will place the IDT/page tables/GDT/ initialization code of your cpu

we will place everything like this:

| entry | address |
| ---- | ----- |
| trampoline code | 0x1000 |
| stack | 0x570 |
| gdt | 0x580 |
| idt | 0x590 |
| table page | 0x600 |
| jump address | 0x610 |

You should know that it will later change the FDT and the page table, all this is temporary and must be replaced, the stack, the GDT, and the page table.

#### GDT + IDT
To store the GDT and the IDT is simple
we can just use 64-bit instructions

sgdt

sidt

These instructions allow to store the GDT and the IDT in a precise address.

so

```intel
sgdt [0x580] ; GDT storage
sidt [0x590] ; IDT storage
```
#### Stack

for the stack we must store a valid __address__ at 0x570

```cpp
POKE(570) = stack_address + stack_size;
```

#### Trampoline Code

For the trampoline code you need assembly code delimited by

__trampoline_start and __trampoline_end

You will have to copy the trampline code from 0x1000 to the size of the trampoline.

donc 

```cpp
// knowing that TRAMPOLINE_START == 0x1000
uint64_t trampoline_len = (uint64_t)&trampoline_end - (uint64_t)&trampoline_start;
    
memcpy((void *)TRAMPOLINE_START, &trampoline_start, trampoline_len);
```
and in the assembly code:

```asm
trampoline_start:

    ; trampoline code

trampoline_end:
```

#### Jump address

The jump address is the function that the cpu will call after its initialization

#### Page table for the future CPUs

The page table can be a copy of the current CPU page table

but if it is a copy then after the initialization of the cpu try to give a copy and not keep the current table.

after having done all this we can go to the initialization of the CPU

## Loading the CPU

avant il faut demander à l'apic de charger le cpu 

il faut faire écrire au 2 registre de commande d'interruption (aussi appelé ICR)

il faut écrire au ICR1  (aka registre 0x300)
0b10100000000 (0x500)


cela veut dire d'envoyer l'interruption d'initialisation au cpu dans ICR2

il faut écrire au ICR2 l'id du processeur shifter de 24

on a donc 

```cpp
write(icr2, (apic_id << 24));
write(icr1, 0x500);
```

__ensuite il faut attendre 10 ms__ pour que le cpu s'initialise

on doit ensuite envoyer à l'apic l'addresse du trampoline pour demander au cpu d'aller en 0x1000

il faut envoyer comme la première étape le apic_id
(apic_id << 24)
mais il faut envoyer à l'icr1
le bit 11 et 10 pour demander aux cpu de charger la page envoyé du trampoline donc  (0x600)


```cpp

write(icr2, (apic_id << 24));
write(icr1, 0x600 | ((uint32_t)trampoline_addr / 4096));
```
maintenant vous pouvez commencer à coder le code du trampoline ! 


## Le Code Du Trampoline 

note: pour débugger vous pouvez utiliser ce code 

```asm
mov al, 'a'
mov dx, 0x3F8
out dx, al
```
le code output le charactère a dans le port com0
c'est utile temporairement pour debugger, c'est la solution la plus courte est simple. Bien sûr le code est temporaire


pour le trampoline il faut savoir que le cpu est initialisé en 16bit, il faut donc le passer comme ceci 

16bit => 32bit => 64bit 


on doit donc faire comme ceci

```asm
[bits 16]
trampoline_start:

trampoline_16:
    ;...

[bits 32]
trampoline_32:
    ;...

[bits 64]
trampoline_64:
    ;...

trampoline_end:
```

#### Le Code 16-Bits

pour passer de 16bit à 32bit il faut initialiser une gdt et mettre le bit 0 du cr0 à 1 pour activer le protected mode

```asm
    cli ; désactiver les interrupt
    mov ax, 0x0 ; mettre tout à 0
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
```

__pour le chargement de la gdt__

il faut que avant le trampoline_end il y ait une structure de gdt pour le 16bit

il faut alors : 
```asm
align 16
gdt_16:
    dw gdt_16_end - gdt_16_start - 1
    dd gdt_16_start - trampoline_start + TRAMPOLINE_BASE

align 16
gdt_16_start:
    ; null selector 0x0
    dq 0
    ; cs selector 8
    dq 0x00CF9A000000FFFF
    ; ds selector 16
    dq 0x00CF92000000FFFF
gdt_16_end:
```

et dans le code on peut faire

```asm
    lgdt [gdt_16 - trampoline_start + TRAMPOLINE_BASE]
```

il faut ensuite faire

```asm
    mov eax, cr0
    or al, 0x1
    mov cr0, eax
```

et pour finir on peut jump dans le code 32bit

```
    jmp 0x8:(trampoline32 - trampoline_start + TRAMPOLINE_BASE)
```
le jmp 0x8:...

permet de dire de loader le segment de code de la gdt


#### Le Code 32 Bits

il faut commencer par charger la table de page dans le cr3

```asm
mov eax, dword [0x600]
mov cr3, eax
```
et ensuite activer le paging, et le PAE du cr4

en mettant les bit 5 et 7 du registre cr4

```asm
mov eax, cr4
or eax, 1 << 5
or eax, 1 << 7
mov cr4, eax
```

il faut ensuite activer le long mode en écrivant le bit 8 du MSR de l'EFER
(L'extended Feature Enable Register)

```asm
    mov ecx, 0xc0000080 ; registre efer
    rdmsr

    or eax,1 << 8 
    wrmsr
```

il faut, ensuite activer le paging dans le registre cr0 en activant le bit 31

```
    mov eax, cr0
    or eax, 1 << 31
    mov cr0, eax
```

pour finir on doit charger une gdt 64bit 

Il faut donc avoir une structure gdt avant le trampoline end
```asm

align 16
gdt_64:
    dw gdt_64_end - gdt_64_start - 1
    dd gdt_64_start - trampoline_start + TRAMPOLINE_BASE

align 16
gdt_64_start:
    ; null selector 0x0
    dq 0
    ; cs selector 8
    dq 0x00AF98000000FFFF
    ; ds selector 16
    dq 0x00CF92000000FFFF
gdt_64_end:
```
et donc charget la gdt 
```asm

lgdt [gdt_64 - trampoline_start + TRAMPOLINE_BASE]
    
```

et pour passer au 64bit on doit jump comme ceci 
```
    jmp 8:(trampoline64 - trampoline_start + TRAMPOLINE_BASE)
```
ceci met le code segment à 8 


#### Le Code 64 Bits

en 64 bit il faut setup les registres ds/ss/es/ par rapport à votre gdt 

```
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov ax, 0x0
    mov fs, ax
    mov gs, ax
```

il faut ensuite charger la gdt/ et l'idt
par rapport au addresse de stockage utilisé 
```
lgdt [0x580]
lidt [0x590]
```

on doit aussi charger la stack

```
mov rsp, [0x570]
mov rbp, 0x0
```
on doit ensuite passer du code copié du trempoline au code physique 
donc on doit faire

```
    jmp virtual_code

virtual_code:
```

dans le virtual code on doit activer certains bit de cr4 et cr0
__si vous voulez le sse, vous devez l'activer ici__

il faut donc activer le bit 1 et désactiver le 2 du registre cr0 pour le monitoring du multi processor et l'émulation 

```
    mov rax, cr0
    btr eax, 2
    bts eax, 1
    mov cr0, rax
```

il faut pour terminer l'initialisation du smp faire

```
    mov rax, [0x610]
    jmp rax
```


maintenant vous avez un cpu d'initialisé ! 

## Dernière Pensée

mais il reste encore beaucoup de chose à faire !
un système de lock, mettre à jour le multitasking, initialiser les cpu avec une gdt/idt/... unique etc...

## Ressources

- [manuel intel](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)
- [osdev](https://wiki.osdev.org/Main_Page)
