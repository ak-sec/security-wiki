## Environment and Setup
I'm using a FLAREVM installation on a Windows 11 Hyper-V virtual machine.

Download file from hackthebox, then disable any internet connection in the lab environment and verify with `ipconfig`.

## Task 1 - What is the process name of malicious NodeJS application?

The file is compressed so I began with 7-zip to unpack the file, revealing an executable and .pcap file.
<img width="711" height="404" alt="superstarpic1" src="https://github.com/user-attachments/assets/85cadbce-0bef-4124-acd3-46ce0d819bc4" />

Detect It Easy identifies the file as a NSIS installer, and looking at the file in PE-bear there is very little code but excessive memory reserved at runtime. This suggests the malware unpacks its payload at runtime.

<img width="1023" height="424" alt="superstarpic3" src="https://github.com/user-attachments/assets/5cbe8e43-e587-4ae6-81d3-def4db5b86f2" />

Decompressing Electron-Coupon.exe reveals Coupon.exe as the main executable of an Electron application
<img width="1430" height="570" alt="superstarpic4" src="https://github.com/user-attachments/assets/78c521e3-ff66-43ea-8671-2ecdbde66649" />

<img width="453" height="168" alt="superstarpic5" src="https://github.com/user-attachments/assets/520b9455-4346-4c59-93bd-b680b2d73734" />

## Task 2 - Which option has the attacker enabled in the script to run the malicious Node.js application?

The resources directory contains an asar archive which likely contains source code, and elevate.exe which is used to elevate privileges.
<img width="624" height="91" alt="superstarpic6" src="https://github.com/user-attachments/assets/b72a2c77-c216-43ab-8eb2-fe3148ab38c5" />
<img width="537" height="139" alt="superstarpic7" src="https://github.com/user-attachments/assets/65365352-64b5-4f77-bcd2-56879c42898c" />

Using a 7-zip plugin (https://www.tc4shell.com/en/7zip/asar/) I extracted app.asar and found index.js, the entry point for the app. Within index.js, `nodeIntegration` is set to true. Enabling `nodeIntegration` exposes full Node.js APIs to renderer content, so any XSS in the page becomes a straightforward path to remote code execution.
<img width="812" height="623" alt="superstarpic8" src="https://github.com/user-attachments/assets/461ed510-60c4-49a2-914b-f438a00f6032" />
<img width="660" height="165" alt="superstarpic9" src="https://github.com/user-attachments/assets/4a16a87f-f188-4cf5-9343-e14741b1aeb0" />

## Task 3 - What protocol and port number is the attacker using to transmit the victim's keystrokes?

The public directory contains keylogger.js, where a socket variable is defined to use websocket protocol on port 44500.
<img width="629" height="362" alt="superstarpic10" src="https://github.com/user-attachments/assets/77ee5214-5abe-4bcf-940c-50737f9c610d" />
<img width="612" height="165" alt="superstarpic11" src="https://github.com/user-attachments/assets/6656c36a-f2fc-46ba-8c3f-449a97aca8a4" />

## Task 4 - What XOR key is the attacker using to decode the encoded shellcode?

Continuing to look through available files, there is preload.js in the extraResources directory. At line 29 of preload.js, the attacker decodes a base64 string by XOR with a key from an http request.
<img width="784" height="879" alt="superstarpic13" src="https://github.com/user-attachments/assets/dceac7a0-d6f5-4e79-b8c8-681caeabd133" />

The problem includes network traffic logs from Wireshark. Filtering those logs to http requests, there are two packets from an unknown IP to the host machine. One of these packets, log 126, contains a 16-byte hex payload.

<img width="1110" height="522" alt="superstarpic14" src="https://github.com/user-attachments/assets/172ccf31-8087-40af-906d-f7ec08b49f60" />
<img width="515" height="168" alt="superstarpic15" src="https://github.com/user-attachments/assets/c5de2bec-7fbd-4727-99d0-883aa2b87320" />

## Task 5 - What is the IP address, port number and process name encoded in the attacker payload?

Looking back at the code, the attacker decodes the base64 string then applies XOR decryption.
<img width="726" height="339" alt="superstarpic16" src="https://github.com/user-attachments/assets/8dc3b4d4-56d9-48c4-a4c1-ec7183088a8b" />

We can reproduce the same operations in cyberchef to achieve the embedded script. The script spawns a cmd.exe subprocess, then establishes a TCP connection with `15[.]206[.]13[.]31:4444`
<img width="897" height="555" alt="superstarpic17" src="https://github.com/user-attachments/assets/dae65e55-fe58-4cca-873b-7ef79ef340da" />
<img width="643" height="169" alt="superstarpic18" src="https://github.com/user-attachments/assets/a920065d-362a-4ba6-ac34-6f581877e03a" />

## Task 6 - What are the two commands the attacker executed after gaining the reverse shell?

From Task 5, we know the attacker established a TCP connection with the victim machine and routed input/output through command prompt. Wireshark shows two TCP sessions between the attacker and the victim. You can right-click each packet to follow the TCP stream.
<img width="234" height="113" alt="superstarpic19" src="https://github.com/user-attachments/assets/20f2dd6f-524f-4d64-8dde-a917464484f8" />

Session 10 is a websocket handshake.

<img width="729" height="501" alt="superstarpic20" src="https://github.com/user-attachments/assets/fc605b2e-ab44-4958-b705-f01e26f412cb" />

Session 11 is the reverse shell session. The attacker ran two commands: `whoami` and `ipconfig`.
<img width="741" height="591" alt="superstarpic21" src="https://github.com/user-attachments/assets/509f0093-52a0-4e68-a6d8-d39ea1e1e13b" />
<img width="580" height="165" alt="superstarpic22" src="https://github.com/user-attachments/assets/3ed21614-fd14-4d3f-b6b6-c6207769f4ad" />

## Task 7 - Which Node.js module and its associated function is the attacker using to execute the shellcode within V8 Virtual Machine contexts?

In preload.js, lines 33-37 show the decoded string made into a script object using the `vm` module. The script is executed using the `runInNewContext()` function.

<img width="782" height="115" alt="preloadJS" src="https://github.com/user-attachments/assets/ac16c6e6-5edd-4fd6-8363-07b084de37fb" />
<img width="904" height="170" alt="superstarpic24" src="https://github.com/user-attachments/assets/59d3d207-8577-40fb-a24b-00d7cc986073" />

## Task 8 - Decompile the bytecode file included in the package and identify the Win32 API used to execute the shellcode.

Task 8 and 9 took up 80% of my time on this challenge. It took me a long time to figure out how to decompile the .jsc file. I tried a few different programs or methods, with View8 (https://github.com/suleram/View8) eventually working after some version and dependency troubleshooting. At first I gave up on View8 after a couple attempts, and proceeded to try and find solutions without decompiling the file.

Win32 APIs used to execute shellcode likely involve allocating memory, writing code to the memory, then executing the code. Analyzing strings from the script.jsc file shows 3 Win32 APIs:
- VirtualAlloc (allocate memory)
- RtlMoveMemory (write code to the memory)
- CreateThread (execute the code)

Of these three processes, CreateThread would execute the shellcode.
<img width="919" height="622" alt="superstarpic25" src="https://github.com/user-attachments/assets/9cc1d539-71e7-46ba-8d88-567d6bb25904" />
<img width="762" height="169" alt="superstarpic26" src="https://github.com/user-attachments/assets/4574d3d5-050b-4911-af23-f997fa509d77" />

I felt a real sense of guilt arriving at this answer without actually decompiling the file, but moved on to task 9 anyway (decompilation at the end).

## Task 9 - Submit the fake discount coupon that the attacker intended to present to the victim.

Without being able to decompile script.jsc, I figured I'd try dynamic analysis and run the malware file in my secure environment. I wrote a short script `loader.js` to load the script.jsc using the bytenode module and log it to the console, then placed it in the same directory as script.jsc

<img width="461" height="138" alt="superstarpic27" src="https://github.com/user-attachments/assets/106b1d28-9b80-4464-bbe8-665f3d7a9bae" />
<img width="650" height="156" alt="superstarpic28" src="https://github.com/user-attachments/assets/d8aa7b4b-76ce-4d0c-b2f9-8e3851a3b725" />


Running my script with `node .\loader.js` returned an error
<img width="1198" height="305" alt="superstarpic29" src="https://github.com/user-attachments/assets/0c7a6a34-c099-4b55-a6eb-697f8819fd79" />

I was using Node V20.7. Detect It Easy identifies `script.jsc` as compiled using bytenode with V8 version `v9.4.146.26`. The documentation for View8 lists `v9.4.146.26` used in Node V16.x
<img width="719" height="401" alt="superstarpic30" src="https://github.com/user-attachments/assets/1bb4d578-1138-41a4-966f-6ceb0792c4db" />

After correcting the node version with `nvm use 16`, script.jsc ran successfully and displayed the solution.
<img width="1371" height="408" alt="superstarpic32" src="https://github.com/user-attachments/assets/4fb67827-1f15-4dfd-839b-3e8293ab07b5" />

## Script.jsc decompilation

After completing the challenge, I gave decompilation another try using View8. I had two issues getting this tool to work initially:

First, the parse module was not installed on my VM. I fixed this with `pip install parse`.
Second, the script was trying to execute with V8 version `v9.4.146.26` as described above while the disassembler binary from View8 was defaulting to another version. Specifying the disassembler path resulted in successful decompilation.
<img width="1198" height="194" alt="superstarpic34" src="https://github.com/user-attachments/assets/055e3408-6bfd-413d-b7a1-304afd985659" />

The resulting `output.txt` contained a long array of numbers and a call to CreateThread (solution to Task 8).
<img width="714" height="284" alt="superstarpic35" src="https://github.com/user-attachments/assets/f4772303-c6b4-4b1d-9f62-d779bfcdc0cd" />

I used cyberchef to convert the decimal to ascii and the output contained the solution `COUPON1337`.
<img width="970" height="629" alt="superstarpic36" src="https://github.com/user-attachments/assets/c8dbd323-7d9a-40dd-9b2f-352daf15897f" />
<img width="605" height="165" alt="superstarpic37" src="https://github.com/user-attachments/assets/44f9d443-9d0e-4488-98f9-10b3036e76bc" />

