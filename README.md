# Dhrake

Dhrake is a collection of scripts for reverse engineering Delphi binaries with [Ghidra] and [IDR]. 

## TL/DR How to Use Dhrake

1. Use [IDR] to load the Delphi binary and generate an IDC script from the symbols that [IDR] extracts.
2. Load the sample into [Ghidra] and let it analyze the file.
3. Run [DhrakeInit](DhrakeInit.java), giving it the path to the IDC file you generated in step 1.
4. Reverse engineer until you see something like:
   ```c
   iVar2 = TStringList.Create(VMT_472694_TStringList, CONCAT31((int3)((uint)extraout_EDX >> 8),1));
   ```
5. Navigate to `VMT_472694_TStringList` in the listing view and run [DhrakeParseClass](DhrakeParseClass.java).
6. Profit.

## Private IDR Build

I have uploaded my [private build of IDR][IDR-BUILD] because building it can be a little time consuming. I compiled this myself from the [IDR] repository in a virtual machine. I take no responsibility what so ever for this software or this build. I run this on my private machine host, I think it is a great piece of software, but if you want to be on the safe side, use a virtual machine.

## What Dhrake Does

There are three major issues when reverse engineering Delphi binaries in Ghidra:

1. Symbol names are missing.
2. A number of function signatures are broken.
3. Creating structs for Delphi classes and their virtual method tables is cumbersome.

Dhrake addresses all of these issues, although it heavily relies on [IDR] to solve the first problem, and having the symbol names is a necessary requirement to solve the other two. The prerequisite to using Dhrake with Ghidra is to generate an IDC file from the symbols extracted by [IDR].

The first thing that Dhrake does is to extract and apply the Delphi symbols from the IDC file which was generated by [IDR]. After that, a number of function signatures are fixed. Currently, the string comparison operators like `@LStrCmp` and the string concatenation functions like `@LStrCat3` and `@LStrCatN` are the only ones that have been implemented, but please open an issue or provide a PR if you have other Delphi library functions that need some love. Most of the repairs are straightforward, but some effort was required to fix calls to the multi string concatenation functions like `@LStrCatN`, the details of which you can find below. In a final step, Dhrake attempts to find cases where Ghidra has incorrectly determined the entry point of a function. This typically looks like this:
```
 ***************************************************************************
 *                                FUNCTION                                 *
 ***************************************************************************
 undefined FUN_00425e48()
      undefined            <RETURN>                   AL:1
  FUN_00425e48
  00425e48   0  PUSH         EBP
  00425e49 004  MOV          EBP, ESP
  00425e4b 004  ADD          ESP, 0xfffff99c
 ***************************************************************************
 *                                FUNCTION                                 *
 ***************************************************************************
 undefined FUN_00425e51()
      undefined            <RETURN>                   AL:1
  FUN_00425e51
  00425e51   0  PUSH         EBX
  00425e52 004  PUSH         ESI
  00425e53 008  PUSH         EDI
  00425e54 00c  XOR          EBX, EBX
  00425e56 00c  MOV          dword ptr [EBP + 0xfffff99c], EBX
  00425e5c 00c  MOV          dword ptr [EBP + -0x4], ECX
  00425e5f 00c  MOV          EBX, EDX
  00425e61 00c  MOV          ESI, EAX
  00425e63 00c  XOR          EAX, EAX
```
The heuristic used to this end is to find functions whose name begins with `FUN_` (Ghidra's default name) whose entry point is less than 25 bytes after another function that is not a thunk. In such a case, Dhrake undefines both functions and creates a new one at the first of the two entry points. If you want to avoid this, you can simply comment out the call to `repairWrongFunctionEntries` in the main `run` method of the script.

## Auto Creating Delphi Structs

To automatically create structs and virtual method tables, there is a separate script [DhrakeParseClass](DhrakeParseClass.java). After importing symbols from [IDR], you can search the **Symbol Table** in Ghidra for symbols that begin with `VMT_`. If you place the cursor on the address of such a symbol, running [DhrakeParseClass](DhrakeParseClass.java) will automatically create a structure for the vtable and for the class body itself. The class structure will have only one entry named `vt`, which will point to the vtable structure. This structure is automatically filled with the function pointers to the methods of this class.

## Reparing LStrCatN

Dhrake sets the function signature of `@LStrCatN` to 
```
void @LStrCatN (char * * Result, uint Count, ...)
```
but it is hard for Ghidra to propertly identify the number of additional arguments to `@LStrCatN`, and it often fails. A call to `@LStrCatN` passes the value `Result` in `EAX`, the value `Count` ind `EDX`, and then `Count` many `char` pointers on the stack. Dhrake enumerates all calls to `@LStrCatN`, determines the `Count` parameter for this call and overrides the call signature to match the number of arguments that are passed on the stack. Unfortunately, since the Delphi register calling convention expects the third parameter to be passed in `ECX` rather than the stack, Dhrake also has to create a third dummy `ECX` parameter, which you can ignore.


[Ghidra]: https://github.com/NationalSecurityAgency/ghidra
[IDR]: https://github.com/crypto2011/IDR
[IDR-BUILD]: https://github.com/huettenhain/dhrake/releases/download/INITIAL/IDR.7z