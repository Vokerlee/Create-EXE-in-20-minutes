# Create-EXE-in-60-minutes
This repository is a guide how to create an executable file in Windows OS. It helps not only to create, but also to understand in more detail how .exe files are arranged. For each step in the guide, a corresponding byte-code is attached for convenience.

## Introduction
The structure of .exe file can be considered as follows:
1. [DOS Header](#dos-header)
2. [DOS Stub](#dos-stub)
3. [NT Header](#nt-header)
    * [NT File Header](#nt-file-header)
    * [NT Optional Header](#nt-optional-header)
4. [Sections header](#section-header)
5. [Program segments](#program-segments)

So let's figure out what are all these contraptions. Of course, let's deal with DOS things first.

## DOS Header
DOS Header is first bytes in our .exe program. Its aim is to describe how the program should act if it is launched on DOS OS. According to MS source codes, it consists of the following stuff:

```C++
struct IMAGE_DOS_HEADER {               // DOS .EXE header
    WORD   e_magic;                     // Magic number
    WORD   e_cblp;                      // Bytes on last page of file
    WORD   e_cp;                        // Pages in file
    WORD   e_crlc;                      // Relocations
    WORD   e_cparhdr;                   // Size of header in paragraphs
    WORD   e_minalloc;                  // Minimum extra paragraphs needed
    WORD   e_maxalloc;                  // Maximum extra paragraphs needed
    WORD   e_ss;                        // Initial (relative) SS value
    WORD   e_sp;                        // Initial SP value
    WORD   e_csum;                      // Checksum
    WORD   e_ip;                        // Initial IP value
    WORD   e_cs;                        // Initial (relative) CS value
    WORD   e_lfarlc;                    // File address of relocation table
    WORD   e_ovno;                      // Overlay number
    WORD   e_res[4];                    // Reserved words
    WORD   e_oemid;                     // OEM identifier (for e_oeminfo)
    WORD   e_oeminfo;                   // OEM information; e_oemid specific
    WORD   e_res2[10];                  // Reserved words
    LONG   e_lfanew;                    // File address of new exe header
  };
```
Not all these fields are necessary (they can be just filled with zeros). So let's fill the most interesting ones:

```C++
// =================================================================================================================
e_magic    = 'ZM'       // (in other way "MZ") Must be always filled with this word (Mark Zbikowski — 
                        // a former Microsoft Architect). It is an identifier that our program is 
                        // executable.
// =================================================================================================================
e_cp       = 0x0003     // It's the size of the "entire" MZ format executable (3 pages). This field is 
                        // intended for loading programs under DOS.
// =================================================================================================================                   
e_cblp     = 0x0090     // This value is bytes on last page of file. It means that in DOS anything 
                        // past the last byte in the last page of executable file is ignored. 
                        // When MS-DOS loads an MZ format executable it copies everything in the file 
                        // after the headers up until this limit. So the fact most PE-files have this
                        // field set to a value bigger than the MS-DOS stub (about it read on) just 
                        // means that the PE-file headers' and part of PE-file section data will be 
                        // loaded into memory when the executable is run under MS-DOS.
// =================================================================================================================                        
e_cparhdr  = 0x0004     // The amount of bytes of DOS Header in paragraphes — 64 bytes or 4 paragraphes. 
                        // (Remember about alignment)
// ================================================================================================================= 
e_minalloc = 0x0010     // The minimal amount of dynamic memory you can use.
// ================================================================================================================= 
e_minalloc = 0xFFFF     // The maximum amount of dynamic memory you can use.
// =================================================================================================================
e_sp       = 0x00B8     // Specifies the initial stack pointer value, which is the absolute value that 
                        // must be loaded into the SP register before the program is given control.
// =================================================================================================================                        
e_lfarlc   = 0x0040     // Specifies the file address of the relocation table, or more specifically, 
                        // the offset from the start of the file to the relocation pointer table.
                        // This value must be used to locate the relocation pointer table (rather
                        // than assuming a fixed location) because variable-length information
                        // pertaining to program overlays can occur before this table, causing its
                        // position to vary. A value of 0x40 in this field generally indicates a
                        // different kind of executable file, not a DOS 'MZ' type.
// ================================================================================================================= 
e_lfanew   = 0x00B0     // This field is the address of the beginning of NT Header. So it is the size 
                        // of DOS Header and DOS Stub in bytes (64 + 112 = 176 = 0x00B0).
```
There are two important aims of this header. First of all it is `e_lfanew` — the address of header that is directly related to the work of the program in Windows OS (NT Header — see further), and the second is the description how the program will behave, if is is launched on DOS OS, using the code from DOS Stub.

- [DOS Header byte-code] <details><summary></summary>
    ```
	4D 5A 90 00 03 00 00 00 04 00 10 00 FF FF 00 00
    B8 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 B0 00 00 00
    ```


## DOS Stub
If you are attentive you already know, that this part is to take 112 bytes. This part of .exe file is to describe the program behaviour on DOS OS. In short, it prints the message "This program cannot be run in DOS mode." and exits from the program. So the assembler code is as follows:

```asm
.code
push cs           ; Keep in mind Code Segment(CS) (where we are in memory)
pop ds            ; Data Segment(DS) = Code Segment(CS)

mov dx, 0x0E      ; The address of the string DS+DX, which will be printed until '$' (the end of the string) 
mov ah, 0x09      ; The number of instruction (print regime)
int 0x21          ; 0x21 DOS interrupt

mov ax, 0x4C01    ; The number of instruction 0x4C (exit from the program) 
int 0x21          ; 0x21 DOS interrupt

.data
load db "This program cannot be run in DOS mode.\x0D\x0A$" ; The output string
```

- [DOS Stub byte-code] <details><summary></summary>
    ```
	0E 1F BA 0E 00 B4 09 CD 21 B8 01 4C CD 21 54 68
    69 73 20 70 72 6F 67 72 61 6D 20 63 61 6E 6E 6F
    74 20 62 65 20 72 75 6E 20 69 6E 20 44 4F 53 20
    6D 6F 64 65 2E 0D 0D 0A 24 00 00 00 00 00 00 00
    5D 5C 6D C1 19 3D 03 92 19 3D 03 92 19 3D 03 92
    97 22 10 92 1E 3D 03 92 E5 1D 11 92 18 3D 03 92
    52 69 63 68 19 3D 03 92 00 00 00 00 00 00 00 00
    ```

## NT Header
It is the most important header, because it is directly connected with the work of the program. The components of this header depends on the system capacity. It can be x64 or x86. There are few differences, so let's consider more general case — x86.

```C++
struct IMAGE_NT_HEADERS {
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER32 OptionalHeader;
};
```
Here we can see `Signature`, which role is the same as `e_magic` in DOS Header. It should be "PE" (program executable) or 'EP'. `FileHeader` is common for both x64 and x86. But x64 architecture has `IMAGE_OPTIONAL_HEADER64 OptionalHeader`. Let's look what these headers are.

### NT File Header
This file has the following structure:
```C++
struct IMAGE_FILE_HEADER {
    WORD    Machine;
    WORD    NumberOfSections;
    DWORD   TimeDateStamp;
    DWORD   PointerToSymbolTable;
    DWORD   NumberOfSymbols;
    WORD    SizeOfOptionalHeader;
    WORD    Characteristics;
}
```
Here is the description of each field:
```C++
// =================================================================================================================
Machine = IMAGE_FILE_MACHINE_I386    // The minimal level of machine, which can execute the program. 
                                     // IMAGE_FILE_MACHINE_I386 = 0x014c
                                     // There is no sence to make big demands for execution. (I386 is enough)
                                     // Types of machines you can find here:
                                     // https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#machine-types
// =================================================================================================================
NumberOfSections     = 0x0003        // The number of sections in the program. For example .text section,
                                     // .idata and .data section. Also there are .rdata, .bss and so on.
                                     // This indicates the size of the section table, which immediately 
                                     // follows the headers.
// =================================================================================================================  
TimeDateStamp        = 0x0003        // The low 32 bits of the number of seconds since 00:00 January 1,
                                     // 1970 (a C run-time time_t value), which indicates when the file was created.
// =================================================================================================================
SizeOfOptionalHeader = sizeof(IMAGE_OPTIONAL_HEADER32)
                                     // Also can be sizeof(IMAGE_OPTIONAL_HEADER64)
                                     // Remember about alignment!
// =================================================================================================================  
Characteristics      = IMAGE_FILE_EXECUTABLE_IMAGE | IMAGE_FILE_32BIT_MACHINE
                                     // The flags that indicate the attributes of the file.
                                     // IMAGE_FILE_EXECUTABLE_IMAGE = 0x0002
                                     // IMAGE_FILE_32BIT_MACHINE    = 0x0100
                                     // There is no sence to add something more, but if you are interested in 
                                     // you can see it: 
                                     // https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#characteristics
// =================================================================================================================  
```

Okey, continue.

### NT Optional Header

Despite the fact that this header is optional, it is neseccary for .exe file.
The structure of this header:

```C++
struct IMAGE_OPTIONAL_HEADER32 {

    // Standard fields.

    WORD    Magic;
    BYTE    MajorLinkerVersion;
    BYTE    MinorLinkerVersion;
    DWORD   SizeOfCode;
    DWORD   SizeOfInitializedData;
    DWORD   SizeOfUninitializedData;
    DWORD   AddressOfEntryPoint;
    DWORD   BaseOfCode;
    DWORD   BaseOfData;

    // NT additional fields.

    DWORD   ImageBase;
    DWORD   SectionAlignment;
    DWORD   FileAlignment;
    WORD    MajorOperatingSystemVersion;
    WORD    MinorOperatingSystemVersion;
    WORD    MajorImageVersion;
    WORD    MinorImageVersion;
    WORD    MajorSubsystemVersion;
    WORD    MinorSubsystemVersion;
    DWORD   Win32VersionValue;
    DWORD   SizeOfImage;
    DWORD   SizeOfHeaders;
    DWORD   CheckSum;
    WORD    Subsystem;
    WORD    DllCharacteristics;
    DWORD   SizeOfStackReserve;
    DWORD   SizeOfStackCommit;
    DWORD   SizeOfHeapReserve;
    DWORD   SizeOfHeapCommit;
    DWORD   LoaderFlags;
    DWORD   NumberOfRvaAndSizes;
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
}
```
So there are a lot of fields, hard work wate for us...
```C++
// =================================================================================================================
Magic = IMAGE_NT_OPTIONAL_HDR32_MAGIC   // Similarly can be IMAGE_NT_OPTIONAL_HDR64_MAGIC
                                        // IMAGE_NT_OPTIONAL_HDR32_MAGIC = 0x10b (the most common value).
// =================================================================================================================
ImageBase = 0x00400000                  // It is the address in virtual memory, where the image will be loaded
                                        // 0x00400000 is the most common value (and is default).
                                        // Must be a multiple of 64K!
// =================================================================================================================  
AddressOfEntryPoint = 0x1000            // The address of the entry point relative to the image base when 
                                        // the executable file is loaded into memory after being launched. 
                                        // For program images, this is the starting address.
                                        // 0x1000 is the most common value.
// =================================================================================================================
BaseOfCode = 0x1000                     // The address of code section relative to the image base.
                                        // In our guide it is the first section, so it equals to AddressOfEntryPoint. 
                                        // Of course, AddressOfEntryPoint can differ from BaseOfCode, for example 
                                        // AddressOfEntryPoint = 0x1034, but BaseOfCode should be 0x1000 in any way,
                                        // because it 0x1000 is the most common value.
// ================================================================================================================= 
BaseOfData = DATA_START                 // The address of code section relative to the image base.
                                        // DATA_START is the address of code section relative to the AddressOfEntryPoint
                                        // This value you can count only after your decide the structure 
                                        // of all sections in your program. For example, if the size of code section
                                        // is 0x5000 and after it import data section follows and has the same 0x5000 size,
                                        // and data section follows immediately after import data section, 
                                        // DATA_START = BaseOfData = BaseOfCode + 0x5000 * 2 = 0xA000
// =================================================================================================================
SectionAlignment = 0x1000               // The alignment (in bytes) of sections when they are loaded into memory. 
                                        // It must be greater than or equal to FileAlignment. 
                                        // The default is the page size for the architecture.
// ================================================================================================================= 
SizeOfImage                             // The size (in bytes) of the image, including all headers, as the image is 
                                        // loaded in memory. It must be a multiple of SectionAlignment.
                                        // So headers take 0x1000 size, and if all k sections has the same size in
                                        // virtual memory N, the value of SizeOfImage is 0x1000 + k*N
// ================================================================================================================= 
FileAlignment = 0x200                   // The alignment factor (in bytes) that is used to align the raw data 
                                        // of sections in the image file. The value should be a power of 2 between 
                                        // 512 and 64 K, inclusive. The default is 512. 
                                        // In short, it is the alignment of sections in .exe file.
// ================================================================================================================= 
SizeOfHeaders = 0x400                   // The combined size of an MS-DOS stub, PE header, and section headers 
                                        // rounded up to a multiple of FileAlignment.
// =================================================================================================================
Subsystem = IMAGE_SUBSYSTEM_WINDOWS_CUI // The subsystem that is required to run this image.
                                        // IMAGE_SUBSYSTEM_WINDOWS_CUI = 3 is a console application.
                                        // IMAGE_SUBSYSTEM_WINDOWS_GUI = 2 is a graphic application.
// =================================================================================================================
NumberOfRvaAndSizes = IMAGE_NUMBEROF_DIRECTORY_ENTRIES
                                        // The number of data-directory entries in the remainder of the optional header. 
                                        // Each describes a location and size.
                                        // IMAGE_NUMBEROF_DIRECTORY_ENTRIES = 16 (a default number).
                                        // Read more there:
                                        // https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#optional-header-data-directories-image-only
// =================================================================================================================
DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress = IMPORT_START;
                                        // IMPORT_START is the address of import data section relative to the AddressOfEntryPoint.
                                        // By analogy with DATA_START, 
                                        // IMPORT_START = BaseOfCode + 0x5000 = 0x6000
                                        // IMAGE_DIRECTORY_ENTRY_IMPORT = 2.
                                        // It fills the virtual address of import data section. Of course, you can 
                                        // describe all fileds. Read more there:
                                        // https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#optional-header-data-directories-image-only
// =================================================================================================================
```
OK, we are too close to finish the .exe file description.
If you are too lazy, here is the defines for all other elements in `DataDirectory`:
```C++
// Directory Entries
#define IMAGE_DIRECTORY_ENTRY_EXPORT            0   // Export Directory
#define IMAGE_DIRECTORY_ENTRY_IMPORT            1   // Import Directory
#define IMAGE_DIRECTORY_ENTRY_RESOURCE          2   // Resource Directory
#define IMAGE_DIRECTORY_ENTRY_EXCEPTION         3   // Exception Directory
#define IMAGE_DIRECTORY_ENTRY_SECURITY          4   // Security Directory
#define IMAGE_DIRECTORY_ENTRY_BASERELOC         5   // Base Relocation Table
#define IMAGE_DIRECTORY_ENTRY_DEBUG             6   // Debug Directory
//      IMAGE_DIRECTORY_ENTRY_COPYRIGHT         7   // (X86 usage)
#define IMAGE_DIRECTORY_ENTRY_ARCHITECTURE      7   // Architecture Specific Data
#define IMAGE_DIRECTORY_ENTRY_GLOBALPTR         8   // RVA of GP
#define IMAGE_DIRECTORY_ENTRY_TLS               9   // TLS Directory
#define IMAGE_DIRECTORY_ENTRY_LOAD_CONFIG      10   // Load Configuration Directory
#define IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT     11   // Bound Import Directory in headers
#define IMAGE_DIRECTORY_ENTRY_IAT              12   // Import Address Table
#define IMAGE_DIRECTORY_ENTRY_DELAY_IMPORT     13   // Delay Load Import Descriptors
#define IMAGE_DIRECTORY_ENTRY_COM_DESCRIPTOR   14   // COM Runtime descriptor
```

Taking into consideration all the examples, we have 3 program sections, located in the next sequence: code, import data and data. Let's set `IMPORT_SIZE = CODE_SIZE = DATA_SIZE = 0x5000`, then `IMPORT_START = 0x1000 + CODE_SIZE = 0x6000`, `DATA_START = 0x1000 + CODE_SIZE + IMPORT_SIZE = 0xA000`. So we have `SizeOfImage = 0x1000 + 3 * CODE_SIZE = 0xF000`.

- [NT Header byte-code example] <details><summary></summary>
    ```
	50 45 00 00 4C 01 03 00 84 75 BA 60 00 00 00 00
    00 00 00 00 E0 00 02 01 0B 01 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 10 00 00 00 10 00 00
    00 B0 00 00 00 00 40 00 00 10 00 00 00 02 00 00
    00 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00
    00 00 01 00 00 04 00 00 00 00 00 00 03 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 10 00 00 00 00 00 00 00 00 00 00 00
    00 60 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00
    ```
    
## Section Headers

After all headers we should describe not less important sections headers. All such headers have the same pattern, implemented through the following structure:

```C++
#define IMAGE_SIZEOF_SHORT_NAME 8

struct _IMAGE_SECTION_HEADER {
    BYTE    Name[IMAGE_SIZEOF_SHORT_NAME];
    union {
            DWORD   PhysicalAddress;
            DWORD   VirtualSize;
    } Misc;
    DWORD   VirtualAddress;
    DWORD   SizeOfRawData;
    DWORD   PointerToRawData;
    DWORD   PointerToRelocations;
    DWORD   PointerToLinenumbers;
    WORD    NumberOfRelocations;
    WORD    NumberOfLinenumbers;
    DWORD   Characteristics;
}
```

If we continue the demonstrated example in the last chapter, we have code, import data and data sections. The main fields can be described in the following way:

```C++
// =================================================================================================================
Name = ".text" / ".idata" / ...         // It is the name of section. There are ".data", ".rdata" and other names.
// =================================================================================================================
Misc.VirtualSize = SECTION_SIZE         // The size of described section, when it is loaded in memory after the program is launched.
                                        // If we follow our example, SECTION_SIZE = IMPORT_SIZE = CODE_SIZE = DATA_SIZE = 0x5000
// =================================================================================================================  
VirtualAddress = VRT_ADDR               // Taking into account all other fields, you are to count this field for all sections.
                                        // If we follow our example, for text section VRT_ADDR = AddressOfEntryPoint,
                                        // for import data and data sections respectively VRT_ADDR equals to IMPORT_START and DATA_START
// =================================================================================================================
SizeOfRawData                           // The size of the initialized data on disk, in bytes. This value must be
                                        // a multiple of the FileAlignment member of the IMAGE_OPTIONAL_HEADER structure. 
                                        // If this value is less than the VirtualSize member, the remainder of 
                                        // the section is filled with zeroes.
                                        // If the section contains only uninitialized data, the member is zero.
                                        // For example, SizeOfRawData = 0x1000.
// ================================================================================================================= 
PointerToRawData                        // A file pointer to the first page within the .exe file. This value must 
                                        // be a multiple of the FileAlignment member of the IMAGE_OPTIONAL_HEADER structure. 
                                        // If a section contains only uninitialized data, set this member is zero.
                                        // For example, for our sections PointerToRawData = 0x400 / 0x1400 / 0x2400 respectively.
// =================================================================================================================
Characteristics                         // The characteristics of the section. 
                                        // For example:
                                        // 1) Code section: 
                                        // Characteristics = IMAGE_SCN_CNT_CODE | IMAGE_SCN_MEM_EXECUTE | IMAGE_SCN_MEM_READ
                                        // 2) Import data section: 
                                        // Characteristics = IMAGE_SCN_CNT_INITIALIZED_DATA | IMAGE_SCN_MEM_READ
                                        // 3) Code section: 
                                        // Characteristics = IMAGE_SCN_MEM_READ | IMAGE_SCN_MEM_WRITE | IMAGE_SCN_CNT_INITIALIZED_DATA
                                        // What these values mean, you can look by the link below (official documentation).
// =================================================================================================================
```

If you want to see more about section headers, check [official documentation](https://docs.microsoft.com/ru-ru/windows/win32/api/winnt/ns-winnt-image_section_header?redirectedfrom=MSDN).

And for considered examples:
- [Section Headers byte-code] <details><summary></summary>
    ```
                            2E 74 65 78 74 00 00 00
    00 50 00 00 00 10 00 00 00 10 00 00 00 04 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 20 00 00 60
    2E 69 64 61 74 61 00 00 00 50 00 00 00 60 00 00
    00 10 00 00 00 14 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 40 00 00 40 2E 64 61 74 61 00 00 00
    00 50 00 00 00 B0 00 00 00 10 00 00 00 24 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 40 00 00 C0
    ```
