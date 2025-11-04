## Environment and Setup
I'm using a FLAREVM installation on a Windows 11 Hyper-V virtual machine. 

Download file from hackthebox, then disable any internet connection in the lab environment and verify with `ipconfig`
<img width="1380" height="653" alt="heartbreaker1" src="https://github.com/user-attachments/assets/fbf64b1d-3482-41e9-9470-b5dad907d3a7" />
After extracting the file with the provided password, append '.vir' to the file extension to prevent accidental execution.

<img width="132" height="136" alt="heartbreaker3" src="https://github.com/user-attachments/assets/c0954ca6-b3a9-4ede-a669-c5f8b01da857" />

## Task 1 - To accurately reference and identify the suspicious binary, please provide its SHA256 hash.

Load the file into Detect It Easy and click hash, ensuring the method is set to SHA265. 
<img width="719" height="530" alt="heartbreaker4" src="https://github.com/user-attachments/assets/37dd84a8-587c-425c-8af2-b2474233944a" />
<img width="1037" height="515" alt="heartbreaker5" src="https://github.com/user-attachments/assets/3d4f5757-8895-4163-8290-dfc5a02595b3" />

Solution
<img width="1182" height="186" alt="heartbreaker6" src="https://github.com/user-attachments/assets/30bd0bc9-6067-426f-8a4a-b940b6ebc1cf" />

## Task 2 - When was the binary file originally created, according to its metadata (UTC)?

Load the file into PE-bear and navigate to "File Hdr" section.
<img width="1276" height="756" alt="heartbreaker7" src="https://github.com/user-attachments/assets/58a2880d-b0a3-4f53-8369-bd58655625ce" />

The "Time Date Stamp" field shows a date of Wednesday, March 13th 2024. 10:38:06 UTC. Formatting according to HTB syntax provides the solution.
<img width="1127" height="173" alt="heartbreaker8" src="https://github.com/user-attachments/assets/05324787-bfa7-479d-808b-a6f974cc8078" />

## Task 3 - Examining the code size in a binary file can give indications about its functionality. Could you specify the byte size of the code in this binary?

The PE-bear analysis shows the majority of the file's content is in the .text section which likely contains the code. The raw byte size of that section is 9600.
<img width="1273" height="757" alt="heartbreaker9" src="https://github.com/user-attachments/assets/791d55d1-de12-4837-b7b6-55004a397211" />
The Optional Hdr section confirms 9600 as the "Size of Code".
<img width="1275" height="754" alt="heartbreaker10" src="https://github.com/user-attachments/assets/5f066224-33de-4143-81fd-7101529c771d" />

Since 9600 is a hexidecimal number, the decimal conversion is the solution.
To convert 9600 from hex to decimal:

0 x 16^0 + 0 x 16^1 + 6 x 16^2 + 9 x 16^3 = 1536 + 36864 = 38400 bytes

<img width="1120" height="166" alt="heartbreaker11" src="https://github.com/user-attachments/assets/51457558-f184-44d4-b8f1-9f369489e493" />

## Task 4 - It appears that the binary may have undergone a file conversion process. Could you determine its original filename?

A common starting point for static analysis is analyzing the strings from the malicious file. We could run strings.exe, but PE-bear has the strings already visible. I filter on `.` assuming that any legacy file name will include a period notating the file extension
<img width="1555" height="944" alt="heartbreaker12" src="https://github.com/user-attachments/assets/2aa2ba15-992a-41d7-b3f3-8ab1f155a713" />
Looking at the few matching strings, newILY.ps1 sticks out as a .ps1 file extension indicates a powershell script.
<img width="1127" height="168" alt="heartbreaker13" src="https://github.com/user-attachments/assets/b108dee9-78d7-4696-985f-07bde9a914bb" />

## Task 5 - Specify the hexadecimal offset where the obfuscated code of the identified original file begins in the binary.

Scolling through the .text section in a hex editor shows 0x2C74 contains '$' which sppears to be the start of a PowerShell variable '$sCrt' followed by readable ASCII.

<img width="659" height="746" alt="heartbreaker14" src="https://github.com/user-attachments/assets/223f9ca7-e493-425f-b6ef-e27c4318166a" />

<img width="1123" height="172" alt="heartbreaker16" src="https://github.com/user-attachments/assets/8262f692-9238-4106-874d-394e641da297" />

## Task 6 - The threat actor concealed the plaintext script within the binary. Can you provide the encoding method used for this obfuscation?

Knowing where the obfuscated code begins, I copied everything to what looks like the end of the script, marked by a closing set of `"`, and saved it to a text file as `script.txt`.
<img width="1223" height="758" alt="heartbreaker17" src="https://github.com/user-attachments/assets/6dd00c26-a5ac-45cc-b951-5697baa0a50b" />

It looks like the character set include A-Z, a-z, 0-9, +, and the script begins with `==`. If you didn't have prior knowledge that this is base64, AI is a helpful tool to identify the encoding method based on those features:
<img width="707" height="542" alt="heartbreaker18" src="https://github.com/user-attachments/assets/80f8a121-ad9c-4845-aff2-b338b4cc22f7" />

Using CyberChef to decode from base64 doesn't return a readable output. 
<img width="1413" height="918" alt="heartbreaker19" src="https://github.com/user-attachments/assets/e69b9295-58c7-45ce-a396-a50bf147d6dd" />

Notably '==' appears at the front of the script, however '=' padding should always occur at the end of a Base64 string. We can reverse the script so that the = appears at the end, then decode with base64. This produces a readable output confirming Base64 as the encoding method.
<img width="1415" height="919" alt="heartbreaker20" src="https://github.com/user-attachments/assets/21468f43-3137-48aa-b060-e6c6f9b64482" />

<img width="1116" height="159" alt="heartbreaker21" src="https://github.com/user-attachments/assets/3e9ef708-e470-4d79-8364-aa4ab88b80ad" />

## Task 7 - What is the specific cmdlet utilized that was used to initiate file downloads?

Looking at the decoded script from task 6, the code defines a url and img variable, then calls `Invoke-WebRequest` to initiate the download.
<img width="1005" height="689" alt="heartbreaker22" src="https://github.com/user-attachments/assets/59df632d-4642-44d2-bcf7-507aa1642132" />
<img width="1121" height="166" alt="heartbreaker23" src="https://github.com/user-attachments/assets/db0813ae-1cbc-4f2f-9653-ae366edf4aff" />

## Task 8 - Could you identify any possible network-related Indicators of Compromise (IoCs) after examining the code? Separate IPs by comma and in ascending order.

The script contains two IP addresses, 44.206.187.144 on line 3, and 35.169.66.138 on line 64. 
<img width="1120" height="167" alt="heartbreaker24" src="https://github.com/user-attachments/assets/4e646d53-89dc-4534-bbb5-8d98ee64d2f2" />

## Task 9 - The binary created a staging directory. Can you specify the location of this directory where the harvested files are stored?

Line 10 of the code specifies a C drive directory: `$targetDir = "C:\Users\Public\Public Files"`. This line is followed by code that check if that directory exists, and creates it if not.

<img width="608" height="107" alt="heartbreaker25" src="https://github.com/user-attachments/assets/1ce396c3-4d10-4014-acf7-8d77d23aacd9" />
<img width="1122" height="165" alt="heartbreaker26" src="https://github.com/user-attachments/assets/1a6248f9-f481-4935-a420-44d18e14c356" />

## Task 10 - What MITRE ID corresponds to the technique used by the malicious binary to autonomously gather data?

Searching through https://attack.mitre.org/techniques/enterprise/, I clicked into the collection category, then chose automated collection as it best matches the functionality of the script.
<img width="1405" height="532" alt="heartbreaker28" src="https://github.com/user-attachments/assets/95568fe2-14a5-420c-9870-e384138d20a2" />
<img width="1128" height="168" alt="heartbreaker29" src="https://github.com/user-attachments/assets/c035ce72-8c61-4e9c-8732-5eddbaffb8e7" />

## Task 11 - What is the password utilized to exfiltrate the collected files through the file transfer program within the binary?

The script contains an SFTP (secure file transfer protocol) request which is likely the code exfiltrating files.

<img width="549" height="118" alt="heartbreaker30" src="https://github.com/user-attachments/assets/54f4c232-de85-4fe2-b024-edf3c0f136ae" />

Using AI to research SFTP syntax, I identified the password component of the command as the portion following `service:`, in this case `M8&C!i6KkmGL1-#`

<img width="708" height="419" alt="heartbreaker31" src="https://github.com/user-attachments/assets/10ad25fe-cf7b-4afe-a566-525d26294870" />
<img width="1128" height="165" alt="heartbreaker32" src="https://github.com/user-attachments/assets/3b6a0b6e-4c8e-4c0c-ab3f-23cc8038403f" />

Solved!

<img width="567" height="125" alt="heartbreaker33" src="https://github.com/user-attachments/assets/ca4522bf-21a5-45e9-a9ff-1826b1e55e57" />
