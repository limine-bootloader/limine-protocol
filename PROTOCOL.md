# The Limine Boot Protocol

The Limine boot protocol is a modern, portable, featureful, and extensible boot
protocol.

This file serves as the protocol's specification and as the official, centralised
collection of [features](#features) that the Limine boot protocol is comprised of.
Bootloaders may support extra unofficial features, but it is strongly recommended
to avoid fragmentation and submit new features by opening a pull request to the
[limine-protocol GitHub repository](https://github.com/Limine-Bootloader/limine-protocol).

The [limine.h](include/limine.h) file provides an implementation of all the
structures and constants described in this document, for the C and C++
languages.


---

## Table of Contents

- [General Notes](#general-notes)
- [Requests Delimiters](#requests-delimiters)
- [Limine Requests Section](#limine-requests-section)
- [Base Revisions](#base-revisions)
- [Base Revision Changes Summary](#base-revision-changes-summary)
  - [Base Revision 0](#base-revision-0)
  - [Base Revision 1](#base-revision-1)
  - [Base Revision 2](#base-revision-2)
  - [Base Revision 3](#base-revision-3)
  - [Base Revision 4](#base-revision-4)
  - [Base Revision 5](#base-revision-5)
  - [Base Revision 6](#base-revision-6)
- [Memory Layout at Entry](#memory-layout-at-entry)
- [Caching](#caching)
  - [x86-64](#x86-64)
  - [aarch64](#aarch64)
  - [riscv64](#riscv64)
  - [loongarch64](#loongarch64)
- [Machine State at Entry](#machine-state-at-entry)
  - [x86-64](#x86-64-1)
  - [aarch64](#aarch64-1)
  - [riscv64](#riscv64-1)
  - [loongarch64](#loongarch64-1)
- [Features](#features)
  - [Request](#request)
  - [Response](#response)
- [Feature List](#feature-list)
  - [Bootloader Info](#bootloader-info-feature)
  - [Executable Command Line](#executable-command-line-feature)
  - [Firmware Type](#firmware-type-feature)
  - [Stack Size](#stack-size-feature)
  - [HHDM (Higher Half Direct Map)](#hhdm-higher-half-direct-map-feature)
  - [Framebuffer](#framebuffer-feature)
  - [Paging Mode](#paging-mode-feature)
  - [MP (Multiprocessor)](#mp-multiprocessor-feature)
  - [RISC-V BSP Hart ID](#risc-v-bsp-hart-id-feature)
  - [Memory Map](#memory-map-feature)
  - [Entry Point](#entry-point-feature)
  - [Executable File](#executable-file-feature)
  - [Module](#module-feature)
  - [RSDP](#rsdp-feature)
  - [SMBIOS](#smbios-feature)
  - [EFI System Table](#efi-system-table-feature)
  - [TPM Event Log](#tpm-event-log-feature)
  - [EFI Memory Map](#efi-memory-map-feature)
  - [Date at Boot](#date-at-boot-feature)
  - [Executable Address](#executable-address-feature)
  - [Device Tree Blob](#device-tree-blob-feature)
  - [Bootloader Performance](#bootloader-performance-feature)
  - [x86-64 Keep IOMMU](#x86-64-keep-iommu-feature)
  - [TSC (Timestamp Counter) Frequency](#tsc-timestamp-counter-frequency-feature)
  - [Flanterm FB Init Params](#flanterm-fb-init-params-feature)
  - [Entropy](#entropy-feature)
- [File Structure](#file-structure)

---

## General Notes

The "executable" is the kernel or otherwise the freestanding application being loaded
by the Limine boot protocol compliant bootloader.

The Limine boot protocol does not enforce any specific executable binary format to use,
though ELF is strongly recommended.

Only 64-bit, Little Endian machines are supported or will be supported in the future.

All pointers are 64-bit wide. All non-NULL pointers point to the object with the
[Higher Half Direct Map](#hhdm-higher-half-direct-map-feature) (HHDM) offset already added
to them, unless otherwise noted.

Unless explicitly specified otherwise, pointer fields in response structures,
and elements of pointer arrays returned in responses, are non-NULL.

Strings passed in or returned by protocol structures are 0-terminated ASCII
strings unless otherwise noted.

Bits in request `flags` fields not defined by this specification are reserved
and must be set to zero by executables. Bits in response `flags` fields not
defined by this specification are reserved and should be ignored by executables.

All [responses](#response) and associated data structures are placed in
[bootloader-reclaimable memory](#memory-map-feature) regions.

The ABIs the Limine protocol uses and expects the executable to comply with are as follows:
  - **x86-64**: System V ABI without FP/SIMD
  - **aarch64**: AAPCS64 without FP/SIMD
  - **riscv64**: LP64 (soft-float)
  - **loongarch64**: LP64S (soft-float)

The executable can internally use FP/SIMD, but when interfacing with the Limine boot
protocol, the above are the expected ABIs.

## Requests Delimiters

The bootloader can be told to start and/or stop searching for [requests](#request)
(including [base revision](#base-revisions) tags) in an executable's loaded image by
placing start and/or end markers, on an 8-byte aligned boundary.

The bootloader will only accept requests placed between the last start marker found (if
there happen to be more than 1, which there should not, ideally) and the first end
marker found.
```c
#define LIMINE_REQUESTS_START_MARKER { 0xf6b8f4b39de7d1ae, 0xfab91a6940fcb9cf, \
                                       0x785c6ed015d3e316, 0x181e920a7852b9d9 }

#define LIMINE_REQUESTS_END_MARKER { 0xadc0e0531bb10d03, 0x9572709f31764c62 }
```

For base revisions [0](#base-revision-0) and [1](#base-revision-1), the requests
delimiters are *hints*. The bootloader can still search for requests and base revision
tags outside the delimited area if it doesn't support the hints.

[Base revision 2](#base-revision-2)'s sole difference compared to
[base revision 1](#base-revision-1) is that support for
request delimiters has to be provided and the delimiters must be honoured, if present,
rather than them just being a hint.

## Limine Requests Section

> [!WARNING]
> This behaviour is deprecated and removed as of [base revision 1](#base-revision-1)

For executables requesting deprecated [base revision 0](#base-revision-0),
if the executable file contains a `.limine_reqs` executable section, the bootloader
will, instead of scanning the whole executable's loaded image for [requests](#request),
fetch the requests from a NULL-terminated array of pointers to the provided requests,
contained inside said section.

## Base Revisions

The Limine boot protocol comes in several base revisions; so far, 7
base revisions are specified: [0 through 6](#base-revision-changes-summary).

Base revisions change certain behaviours of the Limine boot protocol
outside any specific feature. The specifics are going to be described as
needed throughout this specification, but are also coalesced in the
[Base Revision Changes Summary](#base-revision-changes-summary) section.

Base revision 0 through 5 are considered deprecated.
[Base revision 0](#base-revision-0) is the default base revision
an executable is assumed to be requesting and complying to if no base revision tag
is provided by the executable, for backwards compatibility.

A base revision tag is a set of 3 64-bit values placed somewhere in the loaded
executable image on an 8-byte aligned boundary; the first 2 values are a magic number
for the bootloader to be able to identify the tag, and the last value is the
requested base revision number.

```c
#define LIMINE_BASE_REVISION(N) { 0xf9562b2d5c95a6c8, 0x6a7b384944536bdc, (N) }
```

If a bootloader drops support for an older base revision, the bootloader must
fail to boot an executable requesting such base revision. If a bootloader does not yet
support a requested base revision (i.e. if the requested base revision is higher
than the maximum base revision supported), it may boot the executable using any
arbitrary revision it supports, and communicate failure to comply to the executable by
*leaving the 3rd component of the base revision tag unchanged*.
The bootloader may also refuse to boot executables requesting a base revision that
it does not yet support, and this is the expected and strongly recommended behaviour
for bootloaders moving forward, but it is not guaranteed since older bootloaders
may not support base revisions at all.
On the other hand, if the executable's requested base revision is supported,
*the 3rd component of the base revision tag must be set to 0 by the bootloader*.

> [!NOTE]
> This means that unlike when the bootloader drops support for an older base
> revision and *it* is responsible for failing to boot the executable, in case the
> bootloader does not yet support the executable's requested base revision,
> it is up to the executable itself to fail (or handle the condition otherwise),
> in order to deal with older bootloader implementations.

For any Limine-compliant bootloader supporting [base revision 3](#base-revision-3)
or greater, if choosing to boot an executable expecting a base revision the bootloader
does not yet support (which is discouraged for new bootloader implementations), it is
*mandatory* to load such executables using at least base revision 3, and it is
mandatory for it to always set the 2nd component of the base revision tag to the base
revision actually used to load the executable, regardless of whether it was the
requested one or not.

```c
#define LIMINE_BASE_REVISION_SUPPORTED(VAR) ((VAR)[2] == 0)

#define LIMINE_LOADED_BASE_REVISION_VALID(VAR) ((VAR)[1] != 0x6a7b384944536bdc)
#define LIMINE_LOADED_BASE_REVISION(VAR) ((VAR)[1])
```

## Base Revision Changes Summary

This section consolidates all changes introduced by each [base revision](#base-revisions)
for easy reference.

### Base Revision 0

This is the default base revision used if no base revision tag is provided.

- Supports the `.limine_reqs` executable section for providing a list of
    [requests](#request).
- Request delimiters (start/end markers) are treated as hints only.
- Identity mapping (starting at offset 0x1000) available.
- [HHDM](#hhdm-higher-half-direct-map-feature) (Higher Half Direct Map) covers **all**
    memory map regions.
- Memory between 0 and 0x1000 is **never** marked as usable in the
    [memory map](#memory-map-feature).
- **aarch64**: `TTBR0_EL1` points to bootloader-provided identity mapping page tables.

### Base Revision 1

**Changes from Base Revision 0**:
- Dropped support for the `.limine_reqs` executable section [request](#request) search
    method.
- Dropped identity mapping.
- [HHDM](#hhdm-higher-half-direct-map-feature) mappings no longer include
    [memory map regions](#memory-map-feature) of types:
  - Reserved
  - Bad memory
- **aarch64**: `TTBR0_EL1` is now **unspecified** and can be freely used by the executable.
- [Requests delimiters](#requests-delimiters) remain hints only.

### Base Revision 2

**Changes from Base Revision 1**:
- [Requests delimiters](#requests-delimiters) must now be **honoured** if present
    (no longer optional hints).
- All other behaviors remain the same as [base revision 1](#base-revision-1).

### Base Revision 3

**Changes from Base Revision 2**:
- [HHDM](#hhdm-higher-half-direct-map-feature) mapping becomes **restrictive** - only the
    following [memory map regions](#memory-map-feature) are mapped:
  - Usable
  - Bootloader reclaimable
  - Executable and modules
  - Framebuffer
- Dropped unconditional direct map of the first 4 GiB of memory to the
    [Higher Half Direct Map](#hhdm-higher-half-direct-map-feature).
- Memory between 0 and 0x1000 **can now** be marked as usable in the
    [memory map](#memory-map-feature).
- [RSDP](#rsdp-feature) address is returned as **physical** (base revision 3 **only**).
- [SMBIOS](#smbios-feature) entry point addresses are returned as **physical** (base revisions 3 and 4 **only**).
- [EFI system table](#efi-system-table-feature) address is returned as **physical** (base revisions 3 and 4 **only**).
- **Bootloader requirement**: Must support loading executables requesting higher
    unsupported revisions with at least base revision 3.
- **Bootloader requirement**: Must set the 2nd component of the base revision tag to the
    actual base revision used.

### Base Revision 4

**Changes from Base Revision 3**:
- [HHDM](#hhdm-higher-half-direct-map-feature) additionally maps the following
    [memory map regions](#memory-map-feature):
  - Reserved (Mapped)
  - ACPI reclaimable
  - ACPI NVS
- Added new [memory map](#memory-map-feature) region type: `LIMINE_MEMMAP_RESERVED_MAPPED`.
- Guaranteed that ACPI tables (RSDP, RSDT, XSDT, all tables pointed to by RSDT and XSDT,
    FACS, X_FACS, DSDT, X_DSDT) are mapped within `LIMINE_MEMMAP_ACPI_RECLAIMABLE`,
    `LIMINE_MEMMAP_ACPI_NVS`, or `LIMINE_MEMMAP_RESERVED_MAPPED`
    [memory map regions](#memory-map-feature).
- [RSDP](#rsdp-feature) address is returned as **virtual**
    **([HHDM](#hhdm-higher-half-direct-map-feature))** again (physical only in
    [base revision 3](#base-revision-3)).
- **aarch64**:
  - `MAIR_EL1.Attr0` is guaranteed to be `0xff` (Normal Write-Back RW-Allocate
      non-transient).
  - `MAIR_EL1.Attr1` is guaranteed to be the framebuffer's correct caching type.
  - All other `MAIR_EL1` entries are guaranteed unused unless specified by a request
      (no such requests are specified yet).

### Base Revision 5

**Changes from Base Revision 4**:
- Guaranteed that SMBIOS tables (32-bit and 64-bit entry points and their structure tables)
    are mapped as `LIMINE_MEMMAP_RESERVED_MAPPED` [memory map](#memory-map-feature) entries.
- [SMBIOS](#smbios-feature) addresses are returned as **virtual**
    **([HHDM](#hhdm-higher-half-direct-map-feature))** again (physical only in
    [base revision 3](#base-revision-3) and [base revision 4](#base-revision-4)).
- EFI Runtime Services code and data memory regions (`EfiRuntimeServicesCode`,
    `EfiRuntimeServicesData`) are reported as `LIMINE_MEMMAP_RESERVED_MAPPED`
    [memory map](#memory-map-feature) entries (instead of `LIMINE_MEMMAP_RESERVED`).
- The EFI system table and the data it references that remains valid after
    `ExitBootServices` (runtime services table, configuration table array, and firmware
    vendor string) are guaranteed to be mapped as `LIMINE_MEMMAP_RESERVED_MAPPED`
    [memory map](#memory-map-feature) entries. Note: the UEFI specification does not
    guarantee these reside within `EfiRuntimeServicesCode` or `EfiRuntimeServicesData`
    memory regions, so they are explicitly mapped by the bootloader.
- [EFI system table](#efi-system-table-feature) address is returned as **virtual**
    **([HHDM](#hhdm-higher-half-direct-map-feature))** again (physical only in
    [base revision 3](#base-revision-3) and [base revision 4](#base-revision-4)).
- **x86-64**: Extra control registers and descriptor table registers have more
    strictly defined states. See [x86-64 machine state](#x86-64-1) for details.
- **x86-64**: I/O APIC redirection table entries with NMI and ExtINT delivery modes
    are also masked.
- **x86-64**: All Intel VT-d IOMMUs have DMA translation and interrupt remapping disabled.
    All AMD-Vi IOMMUs are disabled.
- **x86-64**: The local APIC is initialised to a well-defined state on all processors
    (BSP and APs). See [x86-64 machine state](#x86-64-1) for details.

### Base Revision 6

**Changes from Base Revision 5**:
- **aarch64**: FP/SIMD/SVE are disabled at entry (`CPACR_EL1` = 0).
- **aarch64**: The executable is entered at EL2 with VHE if the bootloader is at
    EL2 and VHE is supported, and at EL1 otherwise. The executable is never
    entered at EL2 without VHE.
- **aarch64**: Extra system registers and `PSTATE` have more strictly defined
    states. See [aarch64 machine state](#aarch64-1) for details.
- **riscv64**: Extra CSRs have more strictly defined states. See
    [riscv64 machine state](#riscv64-1) for details.
- **loongarch64**: FP/SIMD/LBT are disabled at entry (`CSR.EUEN` = 0).
- **loongarch64**: Extra CSRs have more strictly defined states. See
    [loongarch64 machine state](#loongarch64-1) for details.

## Memory Layout at Entry

Executable segments are mapped in the higher half at or above
`0xffffffff80000000`. Non-relocatable executables must already request
higher-half virtual addresses. Relocatable lower-half executables are supported:
when the lowest loadable virtual address is below `0xffffffff80000000`, the
bootloader applies a slide of at least `0xffffffff80000000 - min_vaddr`, where
`min_vaddr` is the lowest loadable virtual address in the executable image.

A "slide" is an offset applied to the executable's base load address.

At handoff, the executable will be properly loaded and mapped with appropriate
MMU permissions, as supervisor, at each loadable virtual address plus the
applied slide.

For ELF executables, `PT_LOAD` permissions are reflected in the mappings:
`PF_X` controls executability and `PF_W` controls writability; read access is
always available.

No specific physical memory placement is guaranteed, except that the loaded executable image
is guaranteed to be physically contiguous. The virtual-to-physical mapping of
the loaded executable image uses a single uniform offset. In order to determine
where the executable is loaded in physical memory, see the
[Executable Address feature](#executable-address-feature).

Alongside the loaded executable, the bootloader will set up memory mappings as such:

```
 Base Physical Address |                               | Base Virtual Address
 ----------------------+-------------------------------+-----------------------
                       |    (4 GiB - 0x1000) and any   |
  0x0000000000001000   |  additional memory map region |  0x0000000000001000
                       |    (Base revision 0 only)     |
 ----------------------+-------------------------------+-----------------------
                       |     4 GiB and additional      |
  0x0000000000000000   |  memory map regions depending |      HHDM start
                       |       on base revision        |
```
Where "HHDM start" is returned by the [Higher Half Direct Map feature](#hhdm-higher-half-direct-map-feature).
These mappings are supervisor, read, write, execute (-rwx).

When a memory map region is mapped to the Higher Half Direct Map, mappings will use a minimum page size
of 4KiB; if a region's start or end address is not 4KiB aligned, the mappings will overshoot the region
boundaries in order to align to 4KiB while also covering the entire region.

Because framebuffer regions are mapped with a different caching type (see the
[caching section](#caching)), any usable or reserved memory map region that shares
a page with a framebuffer region will be trimmed to avoid the overlap. Memory map
regions of any other type that overlap a framebuffer page boundary are not
permitted and will cause the bootloader to panic.

For [base revision 0](#base-revision-0), the above-4GiB identity and HHDM mappings cover any memory
map region.

For [base revisions 1](#base-revision-1) and [2](#base-revision-2), the above-4GiB HHDM mappings do not
comprise memory map regions of types:
 - Reserved
 - Bad memory

For [base revision 3](#base-revision-3) or greater, the only memory map regions mapped to the HHDM are:
 - Usable
 - Bootloader reclaimable
 - Executable and modules
 - Framebuffer

Additionally, the unconditional direct map of the first 4GiB is dropped, and only memory map regions
of complying types are mapped in.

For [base revision 4](#base-revision-4) or greater, the following regions are also mapped in addition
to those mapped by [base revision 3](#base-revision-3):
 - Reserved (Mapped)
 - ACPI reclaimable
 - ACPI NVS

The bootloader page tables are in [bootloader-reclaimable memory](#memory-map-feature) (see the
[Memory Map feature](#memory-map-feature)), and their specific layout is undefined as long as they provide
the above memory mappings.

If the executable is a position independent executable, the bootloader is free to
relocate it as it sees fit, potentially performing slide randomisation.

The HHDM base is selected by the bootloader and may vary between boots,
including for randomisation. Executables must discover it using the
[HHDM feature](#hhdm-higher-half-direct-map-feature) and must not assume a
fixed offset.

## Caching

### x86-64

The executable, loaded at or above `0xffffffff80000000` in virtual memory, sees all of its
segments mapped using write-back (WB) caching at the page tables level. That being `PAT[0]`, if
the PAT is supported.

All HHDM and identity map memory regions are mapped using write-back (WB) caching at the page
tables level (again, `PAT[0]`), except framebuffer regions which are mapped using write-combining
(WC) caching at the page tables level (`PAT[5]`, if the CPU supports the PAT, see below).

If the CPU supports the PAT (Page Attribute Table), its layout is specified to be as follows:
```
PAT0 -> WB
PAT1 -> WT
PAT2 -> UC-
PAT3 -> UC
PAT4 -> WP
PAT5 -> WC
PAT6 -> unspecified
PAT7 -> unspecified
```

The MTRRs are left as the firmware set them up.

### aarch64

The executable, loaded at or above `0xffffffff80000000` in virtual memory, sees all of its
segments mapped using Normal Write-Back RW-Allocate non-transient caching mode (`MAIR_EL1.Attr0`).

All HHDM and identity map memory regions are mapped using the Normal Write-Back RW-Allocate
non-transient caching mode (guaranteed to be in `MAIR_EL1.Attr0` for
[base revision 4](#base-revision-4) or greater), except for the framebuffer regions, which are
mapped in using an unspecified caching mode (guaranteed to be in `MAIR_EL1.Attr1` for
[base revision 4](#base-revision-4) or greater), correct for use with the framebuffer on the platform.

For base revisions < 4, the `MAIR_EL1` register will at least contain entries for the above-mentioned
caching modes, in an unspecified order.

For [base revision 4](#base-revision-4) and greater, `MAIR_EL1.Attr0` is guaranteed to be `0xff` (AKA Normal
Write-Back RW-Allocate non-transient caching mode), `MAIR_EL1.Attr1` is guaranteed to
be the entry used to map the framebuffer, of the correct caching type for it, and all
other entries in `MAIR_EL1` are guaranteed unused unless otherwise specified by a request
(no such requests are specified yet).

### riscv64

The executable, loaded at or above `0xffffffff80000000`, in virtual memory, and all HHDM and
identity map memory regions are mapped with the default `PBMT=PMA`.

If the `Svpbmt` extension is available, all framebuffer memory regions are mapped
with `PBMT=NC` to enable write-combining optimizations.

If the `Svpbmt` extension is not available, no PMAs can be overridden (effectively,
everything is mapped with `PBMT=PMA`).

### loongarch64

The executable, loaded at or above `0xffffffff80000000`, in virtual memory, sees all of its
segments mapped using the Coherent Cached (CC) memory access type (MAT).

All HHDM and identity map memory regions are mapped using the Coherent Cached (CC)
MAT, except for the framebuffer regions, which are mapped in using the
Weakly-ordered UnCached (WUC) MAT.

## Machine State at Entry

### x86-64

`rip` will be the entry point as defined as part of the executable file format,
unless the [Entry Point feature](#entry-point-feature) is requested, in which case, the value
of `rip` is going to be taken from there.

At entry, `CS` is loaded with `0x28` and `DS`, `ES`, `SS`, `FS`, `GS` are loaded
with `0x30`, pointing to the 64-bit code and data descriptors respectively. In
64-bit mode, segment limits are not enforced and bases for CS, DS, ES, and SS
are forced to zero. FS and GS bases are set to 0.

The GDT register is loaded to point to a GDT, in [bootloader-reclaimable memory](#memory-map-feature),
with at least the following entries, starting at offset 0:

  - Null descriptor
  - 16-bit code descriptor. Base = `0`, limit = `0xffff`. Readable.
  - 16-bit data descriptor. Base = `0`, limit = `0xffff`. Writable.
  - 32-bit code descriptor. Base = `0`, limit = `0xffffffff`. Readable.
  - 32-bit data descriptor. Base = `0`, limit = `0xffffffff`. Writable.
  - 64-bit code descriptor. Base = `0`, limit irrelevant. Readable.
  - 64-bit data descriptor. Base = `0`, limit irrelevant. Writable.

The IDT is in an undefined state. Executable must load its own.

IF flag, VM flag, and direction flag are cleared on entry. Other flags
undefined.

PE is enabled (`cr0`), ET is enabled (`cr0`), WP is enabled (`cr0`), PG is enabled (`cr0`),
PAE is enabled (`cr4`), LME and LMA are enabled (`EFER`). NX is enabled (`EFER`)
(if it is available).
If 5-level paging is requested and available, then 5-level paging is enabled
(LA57 bit in `cr4`).

For [base revision 5](#base-revision-5) or greater, the following machine
state is also guaranteed:

- All other `cr0`, `cr4`, and `EFER` bits beyond those specified above are cleared.
- `RFLAGS` is set to `0x00000002`.
- The task register is loaded with base 0 and limit 0. No TSS is present.
- The LDTR is loaded with the NULL selector. No LDT is present.
- The IDTR is loaded with base 0 and limit 0. No IDT is present.

The A20 gate is opened.

The legacy PICs (if available) have all IRQs masked.

For each existing I/O APIC, its redirection table entries with Fixed (0b000)
or Lowest Priority (0b001) delivery mode are masked.
For [base revision 5](#base-revision-5) or greater, entries with NMI (0b100)
or ExtINT (0b111) delivery mode are also masked. The rest of the entries beyond the
mask flag is left as set by firmware. Entries with other delivery modes are entirely
left as set by firmware.

For [base revision 5](#base-revision-5) or greater, all Intel VT-d IOMMUs have DMA
translation and interrupt remapping disabled, and all AMD-Vi IOMMUs are disabled.
This can be overridden by the [x86-64 "Keep IOMMU" feature](#x86-64-keep-iommu-feature).

For [base revision 5](#base-revision-5) or greater, the local APIC on each processor
(BSP and APs), if available, is initialised as follows:

- The local APIC is enabled (`IA32_APIC_BASE` bit 11) and software-enabled (SVR bit 8).
- The Spurious Interrupt Vector Register is set to `0x1FF`.
- The Task Priority Register is set to 0.
- All LVT entries (LINT0 and LINT1 (if no MADT override, see below), Timer, Thermal
  Monitor (if present), Performance Counter (if present), CMCI (if present), Error)
  whose delivery mode is Fixed (0b000), Lowest Priority (0b001), NMI (0b100), or
  ExtINT (0b111) have their mask bit set. The rest of the entry beyond the mask flag
  is left as set by firmware. Entries with other delivery modes are entirely left as
  set by firmware.
- If MADT Local APIC NMI (type 4) or Local x2APIC NMI (type 0x0A) entries are
  present, the corresponding LINT entries are configured with NMI delivery mode,
  polarity and trigger mode derived from the MPS INTI flags, and masked. This only
  applies if the given LINT entry's original firmware-set delivery mode is Fixed (0b000),
  Lowest Priority (0b001), NMI (0b100), or ExtINT (0b111). LINT entries with other
  delivery modes are entirely left as set by firmware.

If booted by EFI, boot services are exited.

`rsp` is set to point to the top of a stack, in [bootloader-reclaimable memory](#memory-map-feature),
which is at least 64KiB (65536 bytes) in size, or the size specified in the
[Stack Size feature](#stack-size-feature). An invalid return address of 0 is pushed
to this stack before jumping to the executable's entry point.

All other general purpose registers (`rax`-`r15`) are set to 0.

### aarch64

`PC` will be the entry point as defined as part of the executable file format,
unless the [Entry Point feature](#entry-point-feature) is requested, in which case,
the value of `PC` is going to be taken from there.

The contents of the `VBAR_EL1` register are undefined, and the executable must load
its own.

The `MAIR_EL1` register contents are described above, in the [caching section](#caching).

For [base revision 6](#base-revision-6) or greater, the executable is entered in
little-endian AArch64 at either EL1 or EL2, depending on the firmware handoff
state. If the bootloader is running at EL2 and VHE is supported by the hardware,
the executable is entered at EL2 with VHE enabled. Otherwise, the executable is
entered at EL1. The executable is never entered at EL2 without VHE.
For base revisions less than 6, the executable is always entered at EL1.

In all cases, all interrupts are masked (`PSTATE.{D, A, I, F}` are set to 1).
For [base revision 6](#base-revision-6) or greater, all other `PSTATE` fields are
set to 0. For earlier base revisions, other `PSTATE` fields are undefined.

At entry: the MMU (`SCTLR_EL1.M`) is enabled, the I-Cache and D-Cache
(`SCTLR_EL1.{I, C}`) are enabled, data alignment checking (`SCTLR_EL1.A`) is
disabled. SP alignment checking (`SCTLR_EL1.{SA, SA0}`) is enabled. For
[base revision 6](#base-revision-6) or greater, other fields of `SCTLR_EL1` are
0, except bits 29, 28, 23, 22, 20, 11, 8, and 7 which are set to 1. For earlier
base revisions, other fields are reset to 0 or to their reserved value.

For [base revision 6](#base-revision-6) or greater, `CPACR_EL1` is 0. The
executable must enable the relevant `CPACR_EL1` fields before executing any
FP/SIMD/SVE instruction.

The used translation granule size for both `TTBR0_EL1` and `TTBR1_EL1` is 4KiB.

`TCR_EL1.{T0SZ, T1SZ}` are set to 16 under 4-level paging, or 12 under 5-level
paging. Additionally, for 5-level paging, `TCR_EL1.DS` is set to 1.
For [base revision 6](#base-revision-6) or greater, the following `TCR_EL1`
fields are also guaranteed: `TCR_EL1.IPS` is set to match the hardware's
physical address size (from `ID_AA64MMFR0_EL1.PARange`). `TCR_EL1.{TG0, TG1}`
are set to 4KiB granule. `TCR_EL1.{SH0, SH1}` are set to Inner Shareable.
`TCR_EL1.{IRGN0, ORGN0, IRGN1, ORGN1}` are set to Write-Back RW-Allocate.
All other fields of `TCR_EL1` are 0. For earlier base revisions,
`TCR_EL1.{SH0, SH1}` are set to Outer Shareable and other fields beyond
`T0SZ`, `T1SZ`, and `DS` are unspecified.

`TTBR1_EL1` points to the bootloader-provided higher half page tables.
For [base revision 0](#base-revision-0), `TTBR0_EL1` points to the bootloader-provided identity
mapping page tables, and is unspecified for all other base revisions and can
thus be freely used by the executable.

If booted by EFI, boot services are exited.

`SP` is set to point to the top of a stack, in [bootloader-reclaimable memory](#memory-map-feature),
which is at least 64KiB (65536 bytes) in size, or the size specified in the
[Stack Size feature](#stack-size-feature).

All other general purpose registers (including `X29` and `X30`) are set to 0.
`X30` being 0 means the executable must not return from the entry point.

#### EL2 entry

If entered at EL2, VHE is active (`HCR_EL2.{E2H, TGE}` set to 1). All `*_EL1`
register guarantees described above still apply. Due to VHE register redirection,
`*_EL1` accesses from EL2 transparently access the EL2 register bank.

Additionally:
- `HCR_EL2`: `E2H` = 1, `TGE` = 1, `RW` = 1, `SWIO` = 1. Other bits are 0.
- `CPTR_EL2`: 0. The executable must enable the relevant fields before
  executing any FP/SIMD/SVE instruction.
- `CNTHCTL_EL2`: Bits 0 and 1 are set to 1 (timer/counter access not trapped
  from EL0). All other bits are 0.
- `HSTR_EL2`: 0 (no system register trapping).
- All `*_EL12` registers (real EL1 state): Undefined.

### riscv64

At entry the machine is executing in Supervisor mode.

`pc` will be the entry point as defined as part of the executable file format,
unless the [Entry Point feature](#entry-point-feature) is requested, in which case, the
value of `pc` is going to be taken from there.

`x1`(`ra`) is set to 0, the executable must not return from the entry point.

`x2`(`sp`) is set to point to the top of a stack, in [bootloader-reclaimable memory](#memory-map-feature),
which is at least 64KiB (65536 bytes) in size, or the size specified in the
[Stack Size feature](#stack-size-feature).

`x3`(`gp`) is set to 0, executable must load its own global pointer if needed.

`x5`(`t0`) contains the entry point address. All other general purpose registers
are set to 0.

If booted by EFI, boot services are exited.

For [base revision 6](#base-revision-6) or greater, `stvec` is set to 0. The
executable must load its own trap vector. For earlier base revisions, `stvec` is
in an undefined state.

For [base revision 6](#base-revision-6) or greater, `sstatus` is set to
`0x200000000` (`UXL` = 2, all other fields 0). The executable must set the
relevant `sstatus` fields before executing any FP or vector instruction.
For earlier base revisions, `sstatus.SIE` is set to 0; other fields are
unspecified.

`sie` is set to 0.

`satp` is configured with the paging mode specified by the
[Paging Mode feature](#paging-mode-feature). For [base revision 6](#base-revision-6)
or greater, `ASID` is guaranteed to be 0. `PPN` points to the
bootloader-provided page tables.

### loongarch64

At entry the machine is executing in PLV0.

`$pc` will be the entry point as defined as part of the executable file format,
unless the [Entry Point feature](#entry-point-feature) is requested, in which case, the
value of `$pc` is going to be taken from there.

`$r1`(`$ra`) is set to 0, the executable must not return from the entry point.

`$r3`(`$sp`) is set to point to the top of a stack, in [bootloader-reclaimable memory](#memory-map-feature),
which is at least 64KiB (65536 bytes) in size, or the size specified in the
[Stack Size feature](#stack-size-feature).

`$r12`(`$t0`) contains the entry point address. All other general purpose
registers are set to 0.

If booted by EFI, boot services are exited.

`PG` in `CSR.CRMD` is 1, `DA` is 0, `IE` is 0 and `PLV` is 0. For
[base revision 6](#base-revision-6) or greater, `DATF` = 1 (CC), `DATM` = 1
(CC), `WE` = 0, and all other fields are 0. For earlier base revisions, other
fields are unspecified.

For [base revision 6](#base-revision-6) or greater, `CSR.EUEN` is 0. The
executable must enable the relevant `CSR.EUEN` fields before executing any
FP/SIMD instruction. `CSR.ECFG` is 0 (all interrupt enables cleared).

For [base revision 6](#base-revision-6) or greater, `CSR.EENTRY` is 0 and
`CSR.MERRENTRY` is 0. The executable must load its own exception handlers. For
earlier base revisions, these are in an undefined state.

For [base revision 6](#base-revision-6) or greater, `CSR.DMW0` is set to `0x11`
(`VSEG` = 0, `PLV0` = 1, `MAT` = CC). `CSR.DMW1`, `CSR.DMW2`, and `CSR.DMW3`
are 0. For earlier base revisions, `CSR.DMW0-3` are in an undefined state.

`CSR.TLBRENTRY` is filled with a provided TLB refill handler.

Paging is enabled with 4-level page tables. For [base revision 6](#base-revision-6)
or greater, `CSR.PGDL` and `CSR.PGDH` point to the bootloader-provided page
table roots. `CSR.PWCL` and `CSR.PWCH` are configured for 4-level paging with
4KiB pages. `CSR.STLBPS` is set to 12 (4KiB).

## Features

The protocol is centered around the concept of request/response - collectively
named "features" - where the executable requests some action or information from
the bootloader, and the bootloader responds accordingly, if it is capable of
doing so.

In C terms, a feature is comprised of 2 structures: the request, and the response.

### Request

A request has 3 mandatory members at the beginning of the structure:
```c
struct limine_example_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_example_response *response;
    ... optional members follow ...
};
```
* `id` - The ID of the request. This is an 8-byte aligned magic number that the
    bootloader will scan for inside the loaded executable image to find requests.
    Request IDs are composed of 4 64-bit unsigned integers, but the first 2 are
    common to every request:
    ```c
    #define LIMINE_COMMON_MAGIC 0xc7b1dd30df4c8b88, 0x0a82e883a194f07b
    ```
    Requests may be located anywhere inside the loaded executable image as long as they are
    8-byte aligned. There may only be 1 of the same request. The bootloader will refuse
    to boot an executable with multiple of the same request IDs.
* `revision` - The revision of the request that the executable provides. This starts at 0 and is
bumped whenever new members or functionality are added to the request structure.
Bootloaders process requests in a backwards compatible manner, *always*. This
means that if the bootloader does not support the revision of the request,
it will process the request as if it were the highest revision that the bootloader
supports.
* `response` - This field is filled in by the bootloader at load time, with a
pointer to the response structure, if the request was successfully processed.
If the request is unsupported or was not successfully processed, this field
is *left untouched*, meaning that if it was set to `NULL`, it will stay that
way.

### Response

A response has only 1 mandatory member at the beginning of the structure:
```c
struct limine_example_response {
    uint64_t revision;
    ... optional members follow ...
};
```
* `revision` - Like for requests, bootloaders will instead mark responses with a
revision number. This revision is not coupled between requests and responses,
as they are bumped individually when new members are added or functionality is
changed. Bootloaders will set the revision to the one they provide, and this is
*always backwards compatible*, meaning higher revisions support all that lower
revisions do.

This is all there is to features. For a list of official Limine features, read
the "Feature List" section below.

## Feature List

### Bootloader Info Feature

ID:
```c
#define LIMINE_BOOTLOADER_INFO_REQUEST_ID { LIMINE_COMMON_MAGIC, 0xf55038d8e2a1202f, 0x279426fcf5f59740 }
```

Request:
```c
struct limine_bootloader_info_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_bootloader_info_response *response;
};
```

Response:
```c
struct limine_bootloader_info_response {
    uint64_t revision;
    char *name;
    char *version;
};
```

`name` and `version` are 0-terminated ASCII strings containing the name and
version of the loading bootloader.

### Executable Command Line Feature

ID:
```c
#define LIMINE_EXECUTABLE_CMDLINE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x4b161536e598651e, 0xb390ad4a2f1f303a }
```

Request:
```c
struct limine_executable_cmdline_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_executable_cmdline_response *response;
};
```

Response:
```c
struct limine_executable_cmdline_response {
    uint64_t revision;
    char *cmdline;
};
```

`cmdline` is a 0-terminated ASCII string containing the command line associated with the
booted executable. This is a pointer to the same memory as the `string` member of the `executable_file`
structure of the [Executable File feature](#executable-file-feature).

### Firmware Type Feature

ID:
```c
#define LIMINE_FIRMWARE_TYPE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x8c2f75d90bef28a8, 0x7045a4688eac00c3 }
```

Request:
```c
struct limine_firmware_type_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_firmware_type_response *response;
};
```

Response:
```c
struct limine_firmware_type_response {
    uint64_t revision;
    uint64_t firmware_type;
};
```

`firmware_type` is an enumeration that can have one of the following values:
```c
#define LIMINE_FIRMWARE_TYPE_X86BIOS 0
#define LIMINE_FIRMWARE_TYPE_EFI32 1
#define LIMINE_FIRMWARE_TYPE_EFI64 2
#define LIMINE_FIRMWARE_TYPE_SBI 3
```

### Stack Size Feature

ID:
```c
#define LIMINE_STACK_SIZE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x224ef0460a8e8926, 0xe1cb0fc25f46ea3d }
```

Request:
```c
struct limine_stack_size_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_stack_size_response *response;
    uint64_t stack_size;
};
```

* `stack_size` - The requested stack size in bytes (also used for MP processors).

Response:
```c
struct limine_stack_size_response {
    uint64_t revision;
};
```

### HHDM (Higher Half Direct Map) Feature

ID:
```c
#define LIMINE_HHDM_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x48dcf1cb8ad2b852, 0x63984e959a98244b }
```

Request:
```c
struct limine_hhdm_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_hhdm_response *response;
};
```

Response:
```c
struct limine_hhdm_response {
    uint64_t revision;
    uint64_t offset;
};
```

* `offset` - the virtual address offset of the beginning of the higher half
direct map.

### Framebuffer Feature

ID:
```c
#define LIMINE_FRAMEBUFFER_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x9d5827dcd881dd75, 0xa3148604f6fab11b }
```

Request:
```c
struct limine_framebuffer_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_framebuffer_response *response;
};
```

Response:
```c
struct limine_framebuffer_response {
    uint64_t revision;
    uint64_t framebuffer_count;
    struct limine_framebuffer **framebuffers;
};
```

* `framebuffer_count` - How many framebuffers are present.
* `framebuffers` - Pointer to an array of `framebuffer_count` pointers to
`struct limine_framebuffer` structures.

> [!NOTE]
> If no framebuffer is available, no response will be provided.

```c
// Constants for `memory_model`
#define LIMINE_FRAMEBUFFER_RGB 1

struct limine_framebuffer {
    void *address;
    uint64_t width;
    uint64_t height;
    uint64_t pitch;
    uint16_t bpp; // Bits per pixel
    uint8_t memory_model;
    uint8_t red_mask_size;
    uint8_t red_mask_shift;
    uint8_t green_mask_size;
    uint8_t green_mask_shift;
    uint8_t blue_mask_size;
    uint8_t blue_mask_shift;
    uint8_t unused[7];
    uint64_t edid_size;
    void *edid;

    /* Response revision 1 */
    uint64_t mode_count;
    struct limine_video_mode **modes;
};
```

`edid` points to the screen's EDID blob, if available, else NULL.

`modes` is an array of `mode_count` pointers to `struct limine_video_mode` describing the
available video modes for the given framebuffer.

```c
struct limine_video_mode {
    uint64_t pitch;
    uint64_t width;
    uint64_t height;
    uint16_t bpp;
    uint8_t memory_model;
    uint8_t red_mask_size;
    uint8_t red_mask_shift;
    uint8_t green_mask_size;
    uint8_t green_mask_shift;
    uint8_t blue_mask_size;
    uint8_t blue_mask_shift;
};
```

### Paging Mode Feature

The Paging Mode feature allows the executable to control which paging mode is enabled
before control is passed to it.

ID:
```c
#define LIMINE_PAGING_MODE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x95c1a0edab0944cb, 0xa4e5cb3842f7488a }
```

Request:
```c
struct limine_paging_mode_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_paging_mode_response *response;
    uint64_t mode;
    /* Request revision 1 and above */
    uint64_t max_mode;
    uint64_t min_mode;
};
```

* `mode` - the preferred paging mode by the OS; the bootloader should always aim
to pick this mode unless unavailable or overridden by the user in the bootloader's
configuration file.
* `max_mode` - the highest paging mode in numerical order that the OS supports. The
bootloader will refuse to boot the OS if no paging modes of this type or lower
(but equal or greater than `min_mode`) are available.
* `min_mode` - the lowest paging mode in numerical order that the OS supports. The
bootloader will refuse to boot the OS if no paging modes of this type or greater
(but equal or lower than `max_mode`) are available.

If no Paging Mode Request is provided, the values of `mode`, `max_mode`, and `min_mode`
that the bootloader assumes are `LIMINE_PAGING_MODE_<arch>_DEFAULT`,
`LIMINE_PAGING_MODE_<arch>_DEFAULT`, and `LIMINE_PAGING_MODE_<arch>_MIN`, respectively.

If request revision 0 is used, the values of `max_mode` and `min_mode` that the
bootloader assumes are the value of `mode` and `LIMINE_PAGING_MODE_<arch>_MIN`,
respectively.

Response:
```c
struct limine_paging_mode_response {
    uint64_t revision;
    uint64_t mode;
};
```

The response indicates which paging mode was actually enabled by the bootloader.

#### x86-64

Values assignable to `mode`, `max_mode`, and `min_mode`:
```c
#define LIMINE_PAGING_MODE_X86_64_4LVL 0
#define LIMINE_PAGING_MODE_X86_64_5LVL 1

#define LIMINE_PAGING_MODE_X86_64_DEFAULT LIMINE_PAGING_MODE_X86_64_4LVL
#define LIMINE_PAGING_MODE_X86_64_MIN LIMINE_PAGING_MODE_X86_64_4LVL
```

#### aarch64

Values assignable to `mode`, `max_mode`, and `min_mode`:
```c
#define LIMINE_PAGING_MODE_AARCH64_4LVL 0
#define LIMINE_PAGING_MODE_AARCH64_5LVL 1

#define LIMINE_PAGING_MODE_AARCH64_DEFAULT LIMINE_PAGING_MODE_AARCH64_4LVL
#define LIMINE_PAGING_MODE_AARCH64_MIN LIMINE_PAGING_MODE_AARCH64_4LVL
```

#### riscv64

Values assignable to `mode`, `max_mode`, and `min_mode`:
```c
#define LIMINE_PAGING_MODE_RISCV_SV39 0
#define LIMINE_PAGING_MODE_RISCV_SV48 1
#define LIMINE_PAGING_MODE_RISCV_SV57 2

#define LIMINE_PAGING_MODE_RISCV_DEFAULT LIMINE_PAGING_MODE_RISCV_SV48
#define LIMINE_PAGING_MODE_RISCV_MIN LIMINE_PAGING_MODE_RISCV_SV39
```

#### loongarch64

Values assignable to `mode`, `max_mode`, and `min_mode`:
```c
#define LIMINE_PAGING_MODE_LOONGARCH_4LVL 0

#define LIMINE_PAGING_MODE_LOONGARCH_DEFAULT LIMINE_PAGING_MODE_LOONGARCH_4LVL
#define LIMINE_PAGING_MODE_LOONGARCH_MIN LIMINE_PAGING_MODE_LOONGARCH_4LVL
```

### MP (Multiprocessor) Feature

ID:
```c
#define LIMINE_MP_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x95a67b819a1b857e, 0xa0b61b723b6a73e0 }
```

Request:
```c
#define LIMINE_MP_REQUEST_X86_64_X2APIC (1 << 0)

struct limine_mp_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_mp_response *response;
    uint64_t flags;
};
```

* `flags` - Bit 0: Enable x2APIC, if possible. (x86-64 only)

> [!NOTE]
> On x86-64, if firmware has already enabled x2APIC and bit 0 is clear, the
> bootloader will try to disable x2APIC before handoff. If the MP request is
> present and x2APIC cannot be disabled, the bootloader will fail to boot the
> executable.

> [!NOTE]
> The presence of this request will prompt the bootloader to bootstrap
> the secondary processors. This will not be done if this request is not present.

> [!NOTE]
> If this request is supported, even on single-processor system, a response will be provided,
> containing only the bootstrap processor's entry.

> [!NOTE]
> To release an application processor, initialise any data it must observe,
> including `extra_argument`, then publish the entry point with an atomic store
> to `goto_address` using release semantics or stronger. A sequentially
> consistent store is sufficient. The parked processor observes this store with
> acquire semantics before jumping to the entry point.
>
> The `reserved` and `reserved1` fields of `struct limine_mp_info` are reserved
> for bootloader use. Executables must not modify them; parked application
> processors use the `reserved` field internally, including to hold the
> bootloader-provided stack pointer.

#### x86-64:

Response:

```c
#define LIMINE_MP_RESPONSE_X86_64_X2APIC (1 << 0)

struct limine_mp_response {
    uint64_t revision;
    uint32_t flags;
    uint32_t bsp_lapic_id;
    uint64_t cpu_count;
    struct limine_mp_info **cpus;
};
```

* `flags` - Bit 0: x2APIC has been enabled.
* `bsp_lapic_id` - The Local APIC ID of the bootstrap processor.
* `cpu_count` - How many CPUs are present. It includes the bootstrap processor.
* `cpus` - Pointer to an array of `cpu_count` pointers to
`struct limine_mp_info` structures.

Executables can test `LIMINE_MP_RESPONSE_X86_64_X2APIC` in `flags` to
determine whether x2APIC mode was enabled by the bootloader.

> [!NOTE]
> The MTRRs of APs will be synchronised by the bootloader to match
> the BSP, as Intel SDM requires (Vol. 3A, 12.11.5).

```c
struct limine_mp_info;

typedef void (*limine_goto_address)(struct limine_mp_info *);

struct limine_mp_info {
    uint32_t processor_id;
    uint32_t lapic_id;
    uint64_t reserved;
    limine_goto_address goto_address;
    uint64_t extra_argument;
};
```

* `processor_id` - ACPI Processor UID as specified by the MADT
* `lapic_id` - Local APIC ID of the processor as specified by the MADT
* `reserved` - Reserved for bootloader use.
* `goto_address` - An atomic write to this field causes the parked CPU to
jump to the written address, on a stack whose size follows the
[Stack Size feature](#stack-size-feature). A pointer to the
`struct limine_mp_info` structure of the CPU is passed in `RDI`. Other than
that, the CPU state will be the same as described for the bootstrap
processor. This field is unused for the structure describing the bootstrap
processor. For all CPUs, this field is guaranteed to be NULL when control is first passed
to the bootstrap processor.
* `extra_argument` - A free for use field.

#### aarch64:

Response:

```c
struct limine_mp_response {
    uint64_t revision;
    uint64_t flags;
    uint64_t bsp_mpidr;
    uint64_t cpu_count;
    struct limine_mp_info **cpus;
};
```

* `flags` - Always zero
* `bsp_mpidr` - MPIDR of the bootstrap processor, in the same form as the
`mpidr` field of `struct limine_mp_info`: the affinity fields of `MPIDR_EL1`
with all other bits, such as `Res1`, `U`, and `MT`, masked off.
* `cpu_count` - How many CPUs are present. It includes the bootstrap processor.
* `cpus` - Pointer to an array of `cpu_count` pointers to
`struct limine_mp_info` structures.

```c
struct limine_mp_info;

typedef void (*limine_goto_address)(struct limine_mp_info *);

struct limine_mp_info {
    uint32_t processor_id;
    uint32_t reserved1;
    uint64_t mpidr;
    uint64_t reserved;
    limine_goto_address goto_address;
    uint64_t extra_argument;
};
```

* `processor_id` - ACPI Processor UID as specified by the MADT (always 0 on non-ACPI systems).
* `mpidr` - MPIDR of the processor as specified by the MADT or device tree:
the affinity fields with all other bits, such as `Res1`, `U`, and `MT`, masked off.
* `reserved1` and `reserved` - Reserved for bootloader use.
* `goto_address` - An atomic write to this field causes the parked CPU to
jump to the written address, on a stack whose size follows the
[Stack Size feature](#stack-size-feature). A pointer to the
`struct limine_mp_info` structure of the CPU is passed in `X0`. Other than
that, the CPU state will be the same as described for the bootstrap
processor. This field is unused for the structure describing the bootstrap
processor. For all CPUs, this field is guaranteed to be NULL when control is first passed
to the bootstrap processor.
* `extra_argument` - A free for use field.

#### riscv64

Response:

```c
struct limine_mp_response {
    uint64_t revision;
    uint64_t flags;
    uint64_t bsp_hartid;
    uint64_t cpu_count;
    struct limine_mp_info **cpus;
};
```

* `flags` - Always zero
* `bsp_hartid` - Hart ID of the bootstrap processor as reported by the EFI RISC-V Boot Protocol or the SBI.
* `cpu_count` - How many CPUs are present. It includes the bootstrap processor.
* `cpus` - Pointer to an array of `cpu_count` pointers to
`struct limine_mp_info` structures.

```c
struct limine_mp_info;

typedef void (*limine_goto_address)(struct limine_mp_info *);

struct limine_mp_info {
    uint64_t processor_id;
    uint64_t hartid;
    uint64_t reserved;
    limine_goto_address goto_address;
    uint64_t extra_argument;
};
```

* `processor_id` - ACPI Processor UID as specified by the MADT (always 0 on non-ACPI systems).
* `hartid` - Hart ID of the processor as specified by the MADT or Device Tree.
* `reserved` - Reserved for bootloader use.
* `goto_address` - An atomic write to this field causes the parked CPU to
jump to the written address, on a stack whose size follows the
[Stack Size feature](#stack-size-feature). A pointer to the
`struct limine_mp_info` structure of the CPU is passed in `x10`(`a0`). Other than
that, the CPU state will be the same as described for the bootstrap
processor. This field is unused for the structure describing the bootstrap
processor. For all CPUs, this field is guaranteed to be NULL when control is first passed
to the bootstrap processor.
* `extra_argument` - A free for use field.

#### loongarch64

Response:

```c
struct limine_mp_response {
    uint64_t revision;
    uint64_t flags;
    uint64_t bsp_phys_id;
    uint64_t cpu_count;
    struct limine_mp_info **cpus;
};
```

* `flags` - Always zero
* `bsp_phys_id` - Physical CPU ID of the bootstrap processor (as read from `CSR.CPUID`).
* `cpu_count` - How many CPUs are present. It includes the bootstrap processor.
* `cpus` - Pointer to an array of `cpu_count` pointers to
`struct limine_mp_info` structures.

```c
struct limine_mp_info;

typedef void (*limine_goto_address)(struct limine_mp_info *);

struct limine_mp_info {
    uint64_t processor_id;
    uint64_t phys_id;
    uint64_t reserved;
    limine_goto_address goto_address;
    uint64_t extra_argument;
};
```

* `processor_id` - ACPI Processor UID as specified by the MADT (always 0 on non-ACPI systems).
* `phys_id` - Physical CPU ID of the processor as specified by the MADT or device tree.
* `reserved` - Reserved for bootloader use.
* `goto_address` - An atomic write to this field causes the parked CPU to
jump to the written address, on a stack whose size follows the
[Stack Size feature](#stack-size-feature). A pointer to the
`struct limine_mp_info` structure of the CPU is passed in `$a0`. Other than
that, the CPU state will be the same as described for the bootstrap
processor. This field is unused for the structure describing the bootstrap
processor. For all CPUs, this field is guaranteed to be NULL when control is first passed
to the bootstrap processor.
* `extra_argument` - A free for use field.

### RISC-V BSP Hart ID Feature

ID:
```c
#define LIMINE_RISCV_BSP_HARTID_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x1369359f025525f9, 0x2ff2a56178391bb6 }
```

Request:
```c
struct limine_riscv_bsp_hartid_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_riscv_bsp_hartid_response *response;
};
```

Response:
```c
struct limine_riscv_bsp_hartid_response {
    uint64_t revision;
    uint64_t bsp_hartid;
};
```

* `bsp_hartid` - The Hart ID of the boot processor.

> [!NOTE]
> This request contains the same information as `limine_mp_response.bsp_hartid` from the
> [MP feature](#mp-multiprocessor-feature), but doesn't boot up other APs.

> [!NOTE]
> On non-RISC-V platforms, no response will be provided.

### Memory Map Feature

ID:
```c
#define LIMINE_MEMMAP_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x67cf3d9d378a806f, 0xe304acdfc50c3c62 }
```

Request:
```c
struct limine_memmap_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_memmap_response *response;
};
```

Response:
```c
struct limine_memmap_response {
    uint64_t revision;
    uint64_t entry_count;
    struct limine_memmap_entry **entries;
};
```

* `entry_count` - How many memory map entries are present.
* `entries` - Pointer to an array of `entry_count` pointers to
`struct limine_memmap_entry` structures.

```c
// Constants for `type`
#define LIMINE_MEMMAP_USABLE                 0
#define LIMINE_MEMMAP_RESERVED               1
#define LIMINE_MEMMAP_ACPI_RECLAIMABLE       2
#define LIMINE_MEMMAP_ACPI_NVS               3
#define LIMINE_MEMMAP_BAD_MEMORY             4
#define LIMINE_MEMMAP_BOOTLOADER_RECLAIMABLE 5
#define LIMINE_MEMMAP_EXECUTABLE_AND_MODULES 6
#define LIMINE_MEMMAP_FRAMEBUFFER            7
#define LIMINE_MEMMAP_RESERVED_MAPPED        8

struct limine_memmap_entry {
    uint64_t base;
    uint64_t length;
    uint64_t type;
};
```

* `LIMINE_MEMMAP_USABLE` entries represent regions of the address space that are usable RAM,
and do not contain other data, the executable, bootloader information, or anything valuable,
and are therefore free for use.

* `LIMINE_MEMMAP_RESERVED` entries represent regions of the address space that are
reserved for unspecified purposes by the firmware, hardware, or otherwise, and should not
be touched by the executable.

* `LIMINE_MEMMAP_ACPI_RECLAIMABLE` entries represent regions of the address space containing
ACPI related data, such as ACPI tables and AML code. The executable should make absolutely
sure that no data contained in these regions is still needed before deciding to reclaim
these memory regions for itself. Refer to the ACPI specification for further information.

* `LIMINE_MEMMAP_ACPI_NVS` entries represent regions of the address space used for ACPI
non-volatile data storage. Refer to the ACPI specification for further information.

* `LIMINE_MEMMAP_BAD_MEMORY` entries represent regions of the address space that contain
bad RAM, which may be unreliable, and therefore these regions should be treated the same
as reserved regions.

* `LIMINE_MEMMAP_BOOTLOADER_RECLAIMABLE` entries represent regions of the address space
containing RAM used to store bootloader or firmware information that should be available
to the executable (or, in some cases, hardware, such as for MP trampolines). The executable
should make absolutely sure that no data contained in these regions is still needed before
deciding to reclaim these memory regions for itself.

* `LIMINE_MEMMAP_EXECUTABLE_AND_MODULES` entries are meant to have an illustrative purpose
only, and are not authoritative sources to be used as a means to find the addresses of the
executable or modules. One must use the specific Limine features ([Executable Address](#executable-address-feature) and
[Module](#module-feature) features) to do that.

* `LIMINE_MEMMAP_FRAMEBUFFER` entries represent regions of the address space containing
memory-mapped framebuffers. These entries exist for illustrative purposes only, and are
not to be used to acquire the address of any framebuffer. One must use the [Framebuffer
feature](#framebuffer-feature) for that.

* `LIMINE_MEMMAP_RESERVED_MAPPED` ([base revision 4](#base-revision-4) or greater) entries represent regions
of the address space containing the ACPI tables as described by the [Memory Layout at Entry](#memory-layout-at-entry)
section, if the firmware did not already map them within either an ACPI reclaimable
or an ACPI NVS region. For [base revision 5](#base-revision-5) or greater, these entries additionally
contain SMBIOS tables, EFI Runtime Services code and data, and the EFI system table along
with the data it references (see [Base Revision 5](#base-revision-5) for details).

For [base revisions](#base-revisions) <= 2, memory between 0 and 0x1000 is never marked as usable memory.

For [base revision 4](#base-revision-4) or greater, ACPI tables (that being RSDP, RSDT, XSDT, all
tables pointed to by RSDT and XSDT, FACS, X_FACS, DSDT, X_DSDT - if present) are guaranteed
to be mapped within `LIMINE_MEMMAP_ACPI_RECLAIMABLE`, `LIMINE_MEMMAP_ACPI_NVS`, or
`LIMINE_MEMMAP_RESERVED_MAPPED` regions.

The entries are guaranteed to be sorted by base address, lowest to highest.

Usable and bootloader reclaimable entries are guaranteed to be 4096 byte aligned for
both base and length.

Usable and bootloader reclaimable entries are guaranteed not to overlap with any other
entry. To the contrary, all non-usable entries (including executable/modules) are
not guaranteed any alignment, nor is it guaranteed that they do not overlap
other entries.

#### EFI Memory Map Entry Type to Limine Memory Map Type

In case the booting firmware is EFI, the following EFI memory map entry types to Limine memory map type
are guaranteed to be upheld, unless overridden by any previous rules:

* EfiLoaderCode, EfiLoaderData -> `BOOTLOADER_RECLAIMABLE`
* EfiBootServicesCode, EfiBootServicesData -> `BOOTLOADER_RECLAIMABLE`
* EfiRuntimeServicesCode, EfiRuntimeServicesData -> `RESERVED_MAPPED` for [base revision](#base-revisions) >= 5, else, `RESERVED`.
* EfiACPIReclaimMemory -> `ACPI_RECLAIMABLE`
* EfiACPIMemoryNVS -> `ACPI_NVS`
* EfiConventionalMemory -> `USABLE`
* [anything else] -> `RESERVED`

### Entry Point Feature

ID:
```c
#define LIMINE_ENTRY_POINT_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x13d86c035a1cd3e1, 0x2b0caa89d8f3026a }
```

Request:
```c
typedef void (*limine_entry_point)(void);

struct limine_entry_point_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_entry_point_response *response;
    limine_entry_point entry;
};
```

* `entry` - The requested entry point. This must be a non-NULL function pointer
within the loaded executable image.

Response:
```c
struct limine_entry_point_response {
    uint64_t revision;
};
```

### Executable File Feature

ID:
```c
#define LIMINE_EXECUTABLE_FILE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0xad97e90e83f1ed67, 0x31eb5d1c5ff23b69 }
```

Request:
```c
struct limine_executable_file_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_executable_file_response *response;
};
```

Response:
```c
struct limine_executable_file_response {
    uint64_t revision;
    struct limine_file *executable_file;
};
```

* `executable_file` - Pointer to the `struct limine_file` structure for the
executable file (see [File Structure](#file-structure) below). The `string`
member is a pointer to the same memory as the `cmdline` value reported by the
[Executable Command Line feature](#executable-command-line-feature).

### Module Feature

ID:
```c
#define LIMINE_MODULE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x3e7e279702be32af, 0xca1c4f3bd1280cee }
```

Request:
```c
#define LIMINE_INTERNAL_MODULE_REQUIRED (1 << 0)
#define LIMINE_INTERNAL_MODULE_COMPRESSED (1 << 1)

struct limine_internal_module {
    const char *path;
    const char *string;
    uint64_t flags;
};

struct limine_module_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_module_response *response;

    /* Request revision 1 */
    uint64_t internal_module_count;
    struct limine_internal_module **internal_modules;
};
```

* `internal_module_count` - How many internal modules are passed by the executable.
* `internal_modules` - Pointer to an array of `internal_module_count` pointers to
`struct limine_internal_module` structures.

> [!NOTE]
> Internal modules are honoured if the module response has revision >= 1.

As part of `struct limine_internal_module`:

* `path` - Non-NULL path to the module to load. This path is *relative* to the
location of the executable. The path may be suffixed with `#` followed by a
128-character hexadecimal blake2b hash of the file contents, in which case the
bootloader will verify the hash before honouring the module.
* `string` - Non-NULL string associated with the given module. Use an empty
string if no module string is needed.
* `flags` - Flags changing module loading behaviour:
  - `LIMINE_INTERNAL_MODULE_REQUIRED`: Fail if the requested module is not found.
  - `LIMINE_INTERNAL_MODULE_COMPRESSED`: The module is GZ-compressed and should be
    decompressed by the bootloader. This is honoured if the response is revision 2
    or greater.

Internal Limine modules are guaranteed to be loaded *before* user-specified
(configuration) modules, and thus they are guaranteed to appear before user-specified
modules in the `modules` array in the response.

> [!NOTE]
> When Secure Boot is active, every loaded file must have an associated blake2b
> hash, and internal modules are no exception: a `path` lacking a `#<hash>` suffix
> will cause the bootloader to panic, regardless of the `LIMINE_INTERNAL_MODULE_REQUIRED`
> flag. There is no separate hash field on `struct limine_internal_module`; the
> hash must be appended to the `path` string. Executables intended to be bootable
> under Secure Boot must therefore embed the precomputed hash of each internal
> module in the path they hand to the bootloader.

Response:
```c
struct limine_module_response {
    uint64_t revision;
    uint64_t module_count;
    struct limine_file **modules;
};
```

* `module_count` - How many modules are present.
* `modules` - Pointer to an array of `module_count` pointers to
`struct limine_file` structures (see [File Structure](#file-structure) below).

> [!NOTE]
> If no modules are available, no response will be provided.

### RSDP Feature

ID:
```c
#define LIMINE_RSDP_REQUEST_ID { LIMINE_COMMON_MAGIC, 0xc5e77b6b397e7b43, 0x27637845accdcf3c }
```

Request:
```c
struct limine_rsdp_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_rsdp_response *response;
};
```

Response:
```c
struct limine_rsdp_response {
    uint64_t revision;
    void *address;
};
```

* `address` - Address of the RSDP table. Physical for [base revision 3](#base-revision-3) **only**.

> [!NOTE]
> If ACPI is not available, no response will be provided.

### SMBIOS Feature

ID:
```c
#define LIMINE_SMBIOS_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x9e9046f11e095391, 0xaa4a520fefbde5ee }
```

Request:
```c
struct limine_smbios_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_smbios_response *response;
};
```

Response:
```c
struct limine_smbios_response {
    uint64_t revision;
    void *entry_32;
    void *entry_64;
};
```

* `entry_32` - Address of the 32-bit SMBIOS entry point. NULL if not present. Physical for [base revision](#base-revisions) 3 and 4 only.
* `entry_64` - Address of the 64-bit SMBIOS entry point. NULL if not present. Physical for [base revision](#base-revisions) 3 and 4 only.

> [!NOTE]
> If SMBIOS is not available (that being neither a 32, nor a 64-bit entry points are available), no
> response will be provided.

### EFI System Table Feature

ID:
```c
#define LIMINE_EFI_SYSTEM_TABLE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x5ceba5163eaaf6d6, 0x0a6981610cf65fcc }
```

Request:
```c
struct limine_efi_system_table_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_efi_system_table_response *response;
};
```

Response:
```c
struct limine_efi_system_table_response {
    uint64_t revision;
    void *address;
};
```

* `address` - Address of EFI system table. Physical for [base revision](#base-revisions) 3 and 4 only.

> [!NOTE]
> If EFI is not available, no response will be provided.

### TPM Event Log Feature

ID:
```c
#define LIMINE_TPM_EVENT_LOG_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x98e094fc7e76e979, 0xee8d8775c54e1d1f }
```

Request:
```c
struct limine_tpm_event_log_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_tpm_event_log_response *response;
};
```

Response:
```c
struct limine_tpm_event_log_response {
    uint64_t revision;
    uint64_t format;
    uint64_t size;
    void *address;
};
```

* `format` - Format of the event log. One of:
```c
#define LIMINE_TPM_EVENT_LOG_FORMAT_TCG_1_2 1
#define LIMINE_TPM_EVENT_LOG_FORMAT_TCG_2   2
```
* `size` - Size in bytes of the raw event data at `address`.
* `address` - Address (HHDM, in [bootloader reclaimable memory](#memory-map-feature)) of the
    captured TCG event log, or NULL if `size` is 0. The buffer holds the raw
    event stream as defined by the indicated `format`, with no additional
    framing.

> [!NOTE]
> The bootloader captures the firmware event log via `EFI_TCG2_PROTOCOL.GetEventLog()` while
> boot services are alive and copies it into memory that survives `ExitBootServices()`. The
> event stream covers all measurements made before handoff, including those performed by the
> bootloader itself.

> [!NOTE]
> If a TPM is not available, or the firmware does not implement `EFI_TCG2_PROTOCOL`, or the
> event log retrieval fails, no response will be provided.

### EFI Memory Map Feature

ID:
```c
#define LIMINE_EFI_MEMMAP_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x7df62a431d6872d5, 0xa4fcdfb3e57306c8 }
```

Request:
```c
struct limine_efi_memmap_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_efi_memmap_response *response;
};
```

Response:
```c
struct limine_efi_memmap_response {
    uint64_t revision;
    void *memmap;
    uint64_t memmap_size;
    uint64_t desc_size;
    uint64_t desc_version;
};
```

* `memmap` - Address (HHDM, in [bootloader reclaimable memory](#memory-map-feature)) of the EFI memory map.
* `memmap_size` - Size in bytes of the EFI memory map.
* `desc_size` - EFI memory map descriptor size in bytes.
* `desc_version` - Version of EFI memory map descriptors.

> [!NOTE]
> This feature provides data suitable for use with `RT->SetVirtualAddressMap()`, provided
> [HHDM](#hhdm-higher-half-direct-map-feature) offset is subtracted from `memmap`.

> [!NOTE]
> If EFI is not available, no response will be provided.

### Date at Boot Feature

ID:
```c
#define LIMINE_DATE_AT_BOOT_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x502746e184c088aa, 0xfbc5ec83e6327893 }
```

Request:
```c
struct limine_date_at_boot_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_date_at_boot_response *response;
};
```

Response:
```c
struct limine_date_at_boot_response {
    uint64_t revision;
    int64_t timestamp;
};
```

* `timestamp` - The UNIX timestamp, in seconds, taken from the system RTC, representing the date and time of boot.

### Executable Address Feature

ID:
```c
#define LIMINE_EXECUTABLE_ADDRESS_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x71ba76863cc55f63, 0xb2644a48c516a487 }
```

Request:
```c
struct limine_executable_address_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_executable_address_response *response;
};
```

Response:
```c
struct limine_executable_address_response {
    uint64_t revision;
    uint64_t physical_base;
    uint64_t virtual_base;
};
```

* `physical_base` - The physical base address of the executable.
* `virtual_base` - The virtual base address of the executable.

### Device Tree Blob Feature

ID:
```c
#define LIMINE_DTB_REQUEST_ID { LIMINE_COMMON_MAGIC, 0xb40ddb48fb54bac7, 0x545081493f81ffb7 }
```

Request:
```c
struct limine_dtb_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_dtb_response *response;
};
```

Response:
```c
struct limine_dtb_response {
    uint64_t revision;
    void *dtb_ptr;
};
```

* `dtb_ptr` - Virtual (HHDM) pointer to the device tree blob, in [bootloader reclaimable memory](#memory-map-feature).

> [!NOTE]
> If no DTB is available, no response will be provided.

> [!NOTE]
> Information contained in the `/chosen` node may not reflect the information
> given by bootloader tags, and as such the `/chosen` node properties should be ignored.

> [!NOTE]
> If the DTB contained `memory@...` nodes, they will get removed.
> Executables may not rely on these nodes and should use the [Memory Map feature](#memory-map-feature) instead.

### Bootloader Performance Feature

ID:
```c
#define LIMINE_BOOTLOADER_PERFORMANCE_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x6b50ad9bf36d13ad, 0xdc4c7e88fc759e17 }
```

Request:
```c
struct limine_bootloader_performance_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_bootloader_performance_response *response;
};
```

Response:
```c
struct limine_bootloader_performance_response {
    uint64_t revision;
    uint64_t reset_usec;
    uint64_t init_usec;
    uint64_t exec_usec;
};
```

* `reset_usec` - time of system reset in microseconds relative to an arbitrary point in the past.
* `init_usec` - time of bootloader initialisation in microseconds relative to an arbitrary point in
the past.
* `exec_usec` - time of executable handoff in microseconds relative to an arbitrary point in the
past.

> [!NOTE]
> Data provided by this feature is purely informational. The ACPI Firmware Performance Data
> Table may have more correct data and should be preferred if it exists. Bootloaders may implement
> this feature using the FPDT.

> [!NOTE]
> The bootloader may assume `reset_usec` is zero if it cannot or does not know the time of
> system reset, due to implementation or platform restrictions. `reset_usec` will usually be 0 or a
> value near zero, but may be any value relative to any point in the past.

### x86-64 Keep IOMMU Feature

ID:
```c
#define LIMINE_X86_64_KEEP_IOMMU_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x8ebaabe51f490179, 0x2aa86a59ffb4ab0f }
```

Request:
```c
struct limine_x86_64_keep_iommu_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_x86_64_keep_iommu_response *response;
};
```

Response:
```c
struct limine_x86_64_keep_iommu_response {
    uint64_t revision;
};
```

If this feature is requested, the bootloader will not disable IOMMUs (Intel VT-d, AMD-Vi)
that were left enabled by the firmware at hand-off. This is intended for security-conscious
executables that wish to preserve DMA protection set up by firmware.

If this feature is not requested, the bootloader reserves the right to disable any active
IOMMUs before handing control to the executable. This is especially of note for base revisions
5 and greater, where the bootloader is mandated to disable VT-d and AMD-Vi IOMMUs, unless
this feature is requested.

> [!NOTE]
> On non-x86 platforms, no response will be provided.

### TSC (Timestamp Counter) Frequency Feature

ID:
```c
#define LIMINE_TSC_FREQUENCY_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x10f2ee1d87d195e4, 0xf747a2b78f6ddb31 }
```

Request:
```c
struct limine_tsc_frequency_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_tsc_frequency_response *response;
};
```

Response:
```c
struct limine_tsc_frequency_response {
    uint64_t revision;
    uint64_t frequency;
};
```

* `frequency` - The frequency of the primary timestamp counter, in Hz.

The primary timestamp counter is the counter read by the `RDTSC` instruction on x86-64,
`CNTPCT_EL0` on aarch64, `RDTIME` on riscv64, and `RDTIME.D` on loongarch64.

> [!NOTE]
> The frequency value provided by this feature is best-effort, and may not be fully precise
> depending on the platform and the method used by the bootloader to determine it.

> [!NOTE]
> If the bootloader is unable to determine the timestamp counter frequency, no response
> will be provided.

### Flanterm FB Init Params Feature

This feature provides the parameters used by the bootloader to initialise its
[Flanterm](https://github.com/Mintsuki/Flanterm) framebuffer terminal instances.
This allows the executable to initialise Flanterm in the same way as the bootloader,
reproducing the same terminal appearance (wallpaper, colours, font, etc.).

Entries in this response correspond by index to framebuffers in the
[Framebuffer Feature](#framebuffer-feature) response. If a framebuffer does not
support a Flanterm terminal (e.g. non-32bpp), its entry will be zeroed.

ID:
```c
#define LIMINE_FLANTERM_FB_INIT_PARAMS_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x3259399fe7c5f126, 0xe01c1c8c5db9d1a9 }
```

Request:
```c
struct limine_flanterm_fb_init_params_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_flanterm_fb_init_params_response *response;
};
```

Response:
```c
struct limine_flanterm_fb_init_params_response {
    uint64_t revision;
    uint64_t entry_count;
    struct limine_flanterm_fb_init_params **entries;
};
```

* `entry_count` - The number of entries. Matches `framebuffer_count` from the
Framebuffer Feature response.
* `entries` - Pointer to an array of `entry_count` pointers to
`struct limine_flanterm_fb_init_params` structures.

> [!NOTE]
> This feature requires the [Framebuffer Feature](#framebuffer-feature) to also be
> requested. If no framebuffers are available, no response will be provided.

```c
// Constants for `rotation`
#define LIMINE_FLANTERM_FB_ROTATE_0 0
#define LIMINE_FLANTERM_FB_ROTATE_90 1
#define LIMINE_FLANTERM_FB_ROTATE_180 2
#define LIMINE_FLANTERM_FB_ROTATE_270 3

struct limine_flanterm_fb_init_params {
    uint32_t *canvas;
    uint64_t canvas_size;
    uint32_t ansi_colours[8];
    uint32_t ansi_bright_colours[8];
    uint32_t default_bg;
    uint32_t default_fg;
    uint32_t default_bg_bright;
    uint32_t default_fg_bright;
    void *font;
    uint64_t font_width;
    uint64_t font_height;
    uint64_t font_spacing;
    uint64_t font_scale_x;
    uint64_t font_scale_y;
    uint64_t margin;
    uint64_t rotation;
};
```

* `canvas` - Pointer to a pre-rendered background canvas buffer, or NULL if no
wallpaper is configured. The buffer is `canvas_size` bytes and contains 32-bit
pixels in the same format as the associated framebuffer, laid out at the
framebuffer's width and height.
* `canvas_size` - Size of the canvas buffer in bytes.
* `ansi_colours` - The 8 standard ANSI colours (black, red, green, brown, blue,
magenta, cyan, grey).
* `ansi_bright_colours` - The 8 bright ANSI colours.
* `default_bg` - Default background colour.
* `default_fg` - Default foreground colour.
* `default_bg_bright` - Default bright background colour.
* `default_fg_bright` - Default bright foreground colour.
* `font` - Pointer to VGA-style font bitmap data with 256 glyphs. This points
to the actual font data, including the built-in default font if no custom font
is configured. Its size in bytes is `font_width * font_height * 256 / 8`.
* `font_width` - Font character width in pixels (always 8 for VGA fonts).
* `font_height` - Font character height in pixels.
* `font_spacing` - Extra horizontal spacing between characters in pixels.
* `font_scale_x` - Horizontal font scale factor.
* `font_scale_y` - Vertical font scale factor.
* `margin` - Terminal margin in pixels from the screen edge.
* `rotation` - Display rotation, one of the `LIMINE_FLANTERM_FB_ROTATE_*` constants.

### Entropy Feature

ID:
```c
#define LIMINE_ENTROPY_REQUEST_ID { LIMINE_COMMON_MAGIC, 0x65ea80255d5682c5, 0x9117240723f493eb }
```

Request:
```c
struct limine_entropy_request {
    uint64_t id[4];
    uint64_t revision;
    struct limine_entropy_response *response;
    uint64_t value_count;
};
```

* `value_count` - The number of random 64-bit values requested.

Response:
```c
struct limine_entropy_response {
    uint64_t revision;
    uint64_t value_count;
    uint64_t *values;
};
```

* `value_count` - The number of random 64-bit values provided. This is the count
    requested, unless that exceeds an implementation-defined limit.
* `values` - Address (HHDM, in [bootloader reclaimable memory](#memory-map-feature)) of the
    array of `value_count` random 64-bit values, or 0 if `value_count` is 0.

> [!NOTE]
> The values are best effort: the bootloader draws them from the strongest sources
> available to it, such as dedicated CPU instructions or firmware services, and falls
> back to weaker sources, such as a timing-seeded PRNG, only where nothing stronger
> exists. They are suitable for seeding an RNG, but carry no guarantee of
> hardware-grade entropy.

## File Structure

```c
struct limine_uuid {
    uint32_t a;
    uint16_t b;
    uint16_t c;
    uint8_t d[8];
};

#define LIMINE_MEDIA_TYPE_GENERIC 0
#define LIMINE_MEDIA_TYPE_OPTICAL 1
#define LIMINE_MEDIA_TYPE_TFTP 2

struct limine_file {
    uint64_t revision;
    void *address;
    uint64_t size;
    char *path;
    char *string;
    uint32_t media_type;
    uint32_t unused;
    uint8_t tftp_ipv4[4];
    uint32_t tftp_port;
    uint32_t partition_index;
    uint32_t mbr_disk_id;
    struct limine_uuid gpt_disk_uuid;
    struct limine_uuid gpt_part_uuid;
    struct limine_uuid part_uuid;
};
```

* `revision` - Revision of the `struct limine_file` structure.
* `address` - The address of the file. This is always at least 4KiB aligned.
* `size` - The size of the file. Regardless of the file size, all loaded
modules are guaranteed to have all 4KiB chunks of memory they cover for
themselves exclusively.
* `path` - The 0-terminated ASCII path of the file within the volume, with a
leading slash.
* `string` - A 0-terminated ASCII string associated with the file.
* `media_type` - Type of media file resides on.
* `tftp_ipv4` - If not all zero bytes, this is the IPv4 address of the TFTP
server the file was loaded from, in dotted-decimal octet order.
* `tftp_port` - Likewise, but port.
* `partition_index` - 1-based partition index of the volume from which the
file was loaded. If 0, it means invalid or unpartitioned.
* `mbr_disk_id` - If non-0, this is the ID of the disk the file was loaded
from as reported in its MBR.
* `gpt_disk_uuid` - If not all zero bytes, this is the UUID of the disk the
file was loaded from as reported in its GPT.
* `gpt_part_uuid` - If not all zero bytes, this is the UUID of the partition the
file was loaded from as reported in the GPT.
* `part_uuid` - If not all zero bytes, this is the UUID of the filesystem of the
partition the file was loaded from.

A UUID with all bytes zero indicates that the field is unset.
