**LAB 03: RETURN-TO-LIBC**  
LE QUOC THANG - 21110799
In this lab, we exploit a vulnerability using the return-to-libc technique. The `exit()` function address is located right after the `system()` function, allowing the program to exit gracefully once `system()` executes. We pass a memory reference containing the string `/bin/sh` as a parameter to `system()` to execute a command like `"rm tmp/dummyfile"`.

### Vulnerable program (`vuln.c`) details:
- The stack frame consists of a 64-byte buffer (`buf`), a 4-byte saved frame pointer (`ebp`), and a return address (`eip`).
- To reach and overwrite the return address, we need to fill up the 64-byte buffer and the 4-byte `ebp`, which totals 68 bytes.
| Buf – 64 bytes |
| -------------- |
| ebp – 4 bytes  |
| eip            |
| ergc           |
| argv           |
### Performing the attack:
1. **Create the necessary files**: Set up a folder `tmp` and create a file `tmp/dummyfile`.
 
2. **Construct an environment variable** that holds the command `"rm tmp/dummyfile"`.
 
3. **Open `vuln.out` in GDB**: Set a breakpoint at `main` and execute the program to obtain the addresses of `system()` and `exit()`.
 
4. **Check the environment variable address**: Use the command `x/300s $esp` to find the address, which is `0xffffd995`.
 
5. **Craft the exploit**: Based on the stack frame, the exploit consists of:
   - 68 bytes of padding,
   - The address of `system()`,
   - The address of `exit()`,
   - The address of the environment variable.
 

### Final script:
The payload for the exploit is constructed with a Python command:
python -c "print('a' * 68 + '\xb0\x0d\xe5\xf7' + '\xe0\x49\xe4\xf7' + '\x99\xd9\xff\xff')"

This completes the return-to-libc attack.
