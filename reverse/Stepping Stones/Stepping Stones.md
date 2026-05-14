# Description
Something keeps interrupting your flow. Can you step through and figure out where things go off track?

# Initial assessment
The task consists of analyzing the **chall** 64-bit ELF executable file. Preliminary static analysis suggests involvement of many dummy flags used in the process of authenticating the user's input.
<img width="915" height="292" alt="image" src="https://github.com/user-attachments/assets/567e6a22-22e8-499d-aa10-80eb40b6f59b" />

# Code analysis

## Ghidra
Examination in Ghidra revealed several key functions used in the program.
<img width="275" height="281" alt="image" src="https://github.com/user-attachments/assets/0df9ef95-2b23-48ce-bb39-496217132716" />

The main ones are:
- `FUN_00100460` is a "success" function.
- `FUN_001005f0` seems to be a flag checker, when run with invalid flag, it ends the program and extraction of the real one becomes impossible.
To bypass it, I decided to overwrite the `$RIP` wit an address of the next command.
- `FUN_00100820` is a key to understanding authentication process during the execution of a program: <img width="684" height="707" alt="image" src="https://github.com/user-attachments/assets/c0d952ed-38dd-4551-9efc-3626a993e78f" />

There is one caveat to the flag checking function (FUN_00100820): the loop runs 19 times (`if (((long)DAT_00103034 + 0xdU == uVar9) && (bVar2))`). There can be some false positives, during debugging and only
one flag can end up being correct.

After saving the memory address to the comparison of user input with real flag's characters, the dynamic analysis with `gdb` can be started.
<img width="913" height="138" alt="image" src="https://github.com/user-attachments/assets/604e1f82-a023-4891-b0ac-c195b896d905" />
<img width="915" height="292" alt="image" src="https://github.com/user-attachments/assets/f8d783c3-e22b-41b2-92ea-49329b194442" />

# Dynamic analysis

## dbg
Firstly, an address of the current process has to be known. To get it, I typed in `info proc mappings` and got the reference address `0x555555554000`.
<img width="1486" height="583" alt="image" src="https://github.com/user-attachments/assets/b8dfce42-538a-4c1d-a4c7-9bf9bd71ba5f" />

To deactivate flag checking I set the breakpoint `break *(0x555555554000 + 0x8b6)`, where 0x8b6 is the address of a flag checking function.

Next, I set the breakpoint on the address of an earlier discovered comparison `break *(0x555555554000 + 0x949)` (the added value is a result of 0x00100949 – 0x00100000,
where the first value is previously discovered address of comparison and the latter one is an address of a program's beginning in Ghidra). Later just run the program, I had to input some flag 
(it had to be exactly 32 characters) and when the program stopped on `0x5555555548b6` I simply went to the next instruction without executing the function by typing in `set $rip = 0x5555555548bb`.
During each step of the loop, I could see the values of dummy flags.
```
(gdb) starti
Starting program: /home/mujben/btsctf/rev_chall/chall 

Program stopped.
0x00007ffff7fe4540 in _start () from /lib64/ld-linux-x86-64.so.2
(gdb) info proc mappings
process 7563
Mapped address spaces:

          Start Addr           End Addr       Size     Offset  Perms  objfile
      0x555555554000     0x555555555000     0x1000        0x0  r-xp   /home/mujben/btsctf/rev_chall/chall
      0x555555555000     0x555555556000     0x1000     0x1000  r--p   /home/mujben/btsctf/rev_chall/chall
      0x555555556000     0x555555558000     0x2000     0x1000  rw-p   /home/mujben/btsctf/rev_chall/chall
      0x7ffff7fbf000     0x7ffff7fc1000     0x2000        0x0  r--p   [vvar]
      0x7ffff7fc1000     0x7ffff7fc3000     0x2000        0x0  r--p   [vvar_vclock]
      0x7ffff7fc3000     0x7ffff7fc5000     0x2000        0x0  r-xp   [vdso]
      0x7ffff7fc5000     0x7ffff7fc6000     0x1000        0x0  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7fc6000     0x7ffff7ff1000    0x2b000     0x1000  r-xp   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7ff1000     0x7ffff7ffb000     0xa000    0x2c000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7ffb000     0x7ffff7fff000     0x4000    0x36000  rw-p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffffffdd000     0x7ffffffff000    0x22000        0x0  rw-p   [stack]
  0xffffffffff600000 0xffffffffff601000     0x1000        0x0  --xp   [vsyscall]
(gdb) b *(0x555555554000 + 0x8b6)
Breakpoint 1 at 0x5555555548b6
(gdb) b *(0x555555554000 + 0x949)
Breakpoint 2 at 0x555555554949
(gdb) continue
Continuing.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Enter product key: BtSCTF{XXXX-YYYY-ZZZZ-WWWW-1A2B}

Breakpoint 1, 0x00005555555548b6 in ?? ()
(gdb) set $rip = 0x00005555555548bb
(gdb) continue
Continuing.

Breakpoint 2, 0x0000555555554949 in ?? ()
(gdb) x/c $rsi + $r14 + 0x10
0x555555555690: 66 'B'
```
To automate the task I made a small script:
```
(gdb) commands 2
Type commands for breakpoint(s) 2, one per line.
End with a line saying just "end".
>silent
>printf "Flag: %d\n", $r9
>x/c $rsi + $r14 + 0x10
>continue
>end
(gdb) set pagination off
(gdb) continue
```
<details>
  <summary>Finally, the flag showed up as the 13th one:</summary>
  
  ```
  Flag: 13
  0x5555555557f8: 66 'B'
  Flag: 13
  0x555555555803: 116 't'
  Flag: 13
  0x55555555580e: 83 'S'
  Flag: 13
  0x5555555557f9: 67 'C'
  Flag: 13
  0x555555555804: 84 'T'
  Flag: 13
  0x55555555580f: 70 'F'
  Flag: 13
  0x5555555557fa: 123 '{'
  Flag: 13
  0x555555555805: 53 '5'
  Flag: 13
  0x555555555810: 84 'T'
  Flag: 13
  0x5555555557fb: 51 '3'
  Flag: 13
  0x555555555806: 80 'P'
  Flag: 13
  0x555555555811: 45 '-'
  Flag: 13
  0x5555555557fc: 80 'P'
  Flag: 13
  0x555555555807: 49 '1'
  Flag: 13
  0x555555555812: 78 'N'
  Flag: 13
  0x5555555557fd: 71 'G'
  Flag: 13
  0x555555555808: 45 '-'
  Flag: 13
  0x555555555813: 83 'S'
  Flag: 13
  0x5555555557fe: 84 'T'
  Flag: 13
  0x555555555809: 79 'O'
  Flag: 13
  0x555555555814: 78 'N'
  Flag: 13
  0x5555555557ff: 45 '-'
  Flag: 13
  0x55555555580a: 69 'E'
  Flag: 13
  0x555555555815: 83 'S'
  Flag: 13
  0x555555555800: 48 '0'
  Flag: 13
  0x55555555580b: 48 '0'
  Flag: 13
  0x555555555816: 45 '-'
  Flag: 13
  0x555555555801: 56 '8'
  Flag: 13
  0x55555555580c: 68 'D'
  Flag: 13
  0x555555555817: 57 '9'
  Flag: 13
  0x555555555802: 70 'F'
  Flag: 13
  0x55555555580d: 125 '}'
  ```
</details>

# Conclusion
The flag is: **BtSCTF{5T3P-P1NG-STON-ES00-8D9F}**
