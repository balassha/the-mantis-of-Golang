# Package pe implements access to PE (Microsoft Windows Portable Executable) files

### About PE

The Portable Executable format is a file format for executables, object code, DLLs, FON Font files, and others used in 32-bit and 64-bit versions of Windows operating systems. The PE format is a data structure that encapsulates the information necessary for the Windows OS loader to manage the wrapped executable code.

This includes dynamic library references for linking, API export and import tables, resource management data and thread-local storage (TLS) data. On NT operating systems, the PE format is used for EXE, DLL, SYS (device driver), and other file types. The Extensible Firmware Interface (EFI) specification states that PE is the standard executable format in EFI environments.

<p align="center">
  <img width="360" height="400" src="/images/pe.png">
</p>

This post is to explain how Golang interacts with PE files in a generic example.

Here you Go:
```Golang
package main

import (
	"fmt"
	"debug/pe"
	"os"
	"io"
	"encoding/binary"
)

func check(e error) {
    if e != nil {
        panic(e)
    }
}

func ioReader(file string) io.ReaderAt {
	r, err := os.Open(file)
	check(err)
	return r
}

func main() {
	
if len(os.Args) < 2 {
		fmt.Println("Usage: petest pe_file")
		os.Exit(1)
	}

	file := ioReader(os.Args[1])
	f, err := pe.NewFile(file)
	check(err)
	
	var sizeofOptionalHeader32 = uint16(binary.Size(pe.OptionalHeader32{}))
	var sizeofOptionalHeader64 = uint16(binary.Size(pe.OptionalHeader64{}))
	
	var dosheader [96]byte	
	var sign [4]byte
	file.ReadAt(dosheader[0:], 0)
	var base int64
	if dosheader[0] == 'M' && dosheader[1] == 'Z' {
		signoff := int64(binary.LittleEndian.Uint32(dosheader[0x3c:]))
		//var sign [4]byte
		file.ReadAt(sign[:], signoff)
		if !(sign[0] == 'P' && sign[1] == 'E' && sign[2] == 0 && sign[3] == 0) {
			fmt.Printf("Invalid PE File Format.\n")
		}
		base = signoff + 4
	} else {
		base = int64(0)
	}

	sr := io.NewSectionReader(file, 0, 1<<63-1)
	sr.Seek(base, os.SEEK_SET)
	binary.Read(sr, binary.LittleEndian, &f.FileHeader)

	var oh32 pe.OptionalHeader32
	var oh64 pe.OptionalHeader64
	var x86_x64 string
	var magicNumber uint16
	
	switch f.FileHeader.SizeOfOptionalHeader {
		case sizeofOptionalHeader32:
			binary.Read(sr, binary.LittleEndian, &oh32)
			if oh32.Magic != 0x10b { // PE32
				fmt.Printf("pe32 optional header has unexpected Magic of 0x%x", oh32.Magic)
			}
			magicNumber = oh32.Magic
			x86_x64 = "x86"

		case sizeofOptionalHeader64:
			binary.Read(sr, binary.LittleEndian, &oh64)
			if oh64.Magic != 0x20b { // PE32+
				fmt.Printf("pe32+ optional header has unexpected Magic of 0x%x", oh64.Magic)
			}
			magicNumber = oh64.Magic
			x86_x64 = "x64"
	}
	
	var isDLL bool
	if (f.Characteristics & 0x2000) == 0x2000 {
		isDLL = true
	} else if (f.Characteristics & 0x2000) != 0x2000 {
		isDLL = false
	}
	
	var isSYS bool
	if (f.Characteristics & 0x1000) == 0x1000 {
		isSYS = true
	} else if (f.Characteristics & 0x1000) != 0x1000 {
		isSYS = false
	}
		
	f.Close() //close file handle
	
	
	fmt.Printf("OptionalHeader: %#x\n", f.OptionalHeader)
	fmt.Printf("DLL File: %t\n", isDLL)
	fmt.Printf("SYS File: %t\n", isSYS)
	fmt.Printf("Base: %d\n", base)
	fmt.Printf("File type: %c%c\n", sign[0],sign[1])
	fmt.Printf("dosheader[0]: %c\n", dosheader[0])
	fmt.Printf("dosheader[1]: %c\n", dosheader[1])
	fmt.Printf("MagicNumber: %#x (%s)\n", magicNumber, x86_x64)

}
```
Compile with: go build -i pe_test.go

Usage: petest file-to-analyze.ext

The expected output will be something like this:
```golang
OptionalHeader: &{0x20b 0xd7000 0x400 0x177200 0x10 c4000 0x9204}
DLL File: false
SYS File: false
Base: 252
File type: PE
dosheader[0]: M
dosheader[1]: Z
MagicNumber: 0x20b (x64)
```
