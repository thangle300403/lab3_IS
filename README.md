# LAB 03: RETURN-TO-LIBC  
**LE QUOC THANG - 21110799**

In this lab, we exploit a vulnerability using the return-to-libc technique. The `exit()` function address is located right after the `system()` function, allowing the program to exit gracefully once `system()` executes. We pass a memory reference containing the string `/bin/sh` as a parameter to `system()` to execute a command like `"rm tmp/dummyfile"`.

### Vulnerable Program (`vuln.c`) Details
- The stack frame consists of a 64-byte buffer (`buf`), a 4-byte saved frame pointer (`ebp`), and a return address (`eip`).
- To overwrite the return address, we need to fill the 64-byte buffer and the 4-byte `ebp`, totaling 68 bytes.

![Stack Frame Diagram](https://github.com/user-attachments/assets/7c61cb97-fc2f-4037-a1dd-a88031aebf5d)

### Performing the Attack

1. **Create the Necessary Files**: Set up a folder `tmp` and create a file `tmp/dummyfile`.

   ![Create Files](https://github.com/user-attachments/assets/a1d0f999-1420-42b9-92af-87ad58a58114)

2. **Construct an Environment Variable**: This variable holds the command `"rm tmp/dummyfile"`.

   ![Environment Variable](https://github.com/user-attachments/assets/409281eb-4118-49d9-997a-0a83096dd979)

3. **Open `vuln.out` in GDB**: Set a breakpoint at `main` and execute the program to obtain the addresses of `system()` and `exit()`.

   ![GDB Breakpoint](https://github.com/user-attachments/assets/ad936c2e-6096-4b25-9cc4-262a3beaba2a)

4. **Check the Environment Variable Address**: Use the command `x/300s $esp` to find the address, which is `0xffffd995`.

   ![Check Address](https://github.com/user-attachments/assets/36b012ae-1a5a-43cc-8aaf-7f4941b9460c)

5. **Craft the Exploit**: The exploit consists of:
   - 68 bytes of padding
   - The address of `system()`
   - The address of `exit()`
   - The address of the environment variable

   ![Exploit Crafting](https://github.com/user-attachments/assets/282b2f32-44da-4364-9ae0-f593e36aa60f)

### Final Script
The payload for the exploit is constructed using the following Python command:

```bash
python -c "print('a' * 68 + '\xb0\x0d\xe5\xf7' + '\xe0\x49\xe4\xf7' + '\x99\xd9\xff\xff')"
```

![Picture6](https://github.com/user-attachments/assets/18a4d655-6a94-45c5-97f7-b34e2b1ad101)
![Picture7](https://github.com/user-attachments/assets/675b7b92-4413-4b2a-b2f7-6800e2e6c31e)
![Picture8](https://github.com/user-attachments/assets/c6e833ff-ddfc-4efe-a0e6-64cf05d361e3)

