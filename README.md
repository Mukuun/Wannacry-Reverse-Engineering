Of course. Here is your WannaCry analysis formatted as a professional report suitable for a GitHub README. I've structured it with clear headings, organized the content for readability, and highlighted key technical details.

-----

# WannaCry Reverse Engineering Analysis

This report provides a static analysis of the WannaCry ransomware, focusing on its use of Windows API functions and the famous "kill switch" mechanism that was instrumental in stopping its initial spread.

-----

## Introduction

### The Shadow Brokers and EternalBlue

**The Shadow Brokers (TSB)** is a hacker group that emerged in the summer of 2016. The group's name is taken from a character in the *Mass Effect* video game series. TSB gained notoriety by leaking a trove of powerful hacking tools, zero-day exploits, and vulnerabilities stolen from the U.S. National Security Agency (NSA).

Among the leaked tools was the **EternalBlue** exploit, which targets a vulnerability in Microsoft's Server Message Block (SMB) protocol. In a notable turn of events, Microsoft released a security patch for this vulnerability (MS17-010) on March 14, 2017, exactly one month before TSB publicly leaked the exploit. This timing raised speculation about whether the NSA had privately disclosed the flaw to Microsoft in advance.

-----

## Static Analysis Overview

The initial phase of analysis involved examining the strings within the WannaCry executable. This **string analysis** provides a baseline of key indicators—such as function names, URLs, and file names—to look for during the more detailed disassembly and code analysis phase.

-----

## Windows API Functions Exploited

WannaCry, like most sophisticated malware, leverages the operating system's core functionalities by calling legitimate Windows API functions. These calls are grouped below by their role in the attack chain.

### 1\. Environment & User Interaction Analysis

Before executing its main payload, the malware checks its surroundings to detect sandboxes and identify the best way to display the ransom note.

  * **Functions Used**: `GetProcessWindowStation`, `GetUserObjectInformationW`, `GetActiveWindow`, `GetLastActivePopup`
  * **Malicious Purpose**:
      * **Sandbox Detection**: To determine if it's running in an isolated analysis environment versus a genuine user session.
      * **UI Hijacking**: To find the active window to ensure the ransom note is displayed prominently to the victim.

### 2\. File Encryption & Ransom Note Delivery

These functions are central to the ransomware's primary goal: encrypting files and notifying the victim.

  * **Functions Used**: `CreateFileA`, `WriteFile`, `CloseHandle`, `MessageBoxW`
  * **Malicious Purpose**:
      * **File Access**: To open target files for encryption and to create ransom note files (e.g., `@Please_Read_Me@.txt`).
      * **Data Manipulation**: To write the encrypted data back to the files and to write the ransom message into the note files.
      * **User Notification**: To display the ransom demand in a pop-up dialog box.

### 3\. Persistence & Privilege Escalation

To ensure it survives a reboot and operates with the necessary permissions, WannaCry installs itself as a Windows service.

  * **Functions Used**: `OpenSCManagerA`, `CreateServiceA`, `StartServiceA`, `ChangeServiceConfig2A`, `SetServiceStatus`
  * **Malicious Purpose**:
      * **Service Creation**: To create a new Windows service to run the main payload and ensure it launches automatically on system startup.
      * **Execution Control**: To manage the service's lifecycle, making the malware resilient and persistent.

### 4\. Cryptographic Operations

WannaCry uses the built-in Windows cryptographic library (CryptoAPI) to generate strong encryption keys.

  * **Functions Used**: `CryptAcquireContextA`, `CryptGenRandom`
  * **Malicious Purpose**:
      * **Key Generation**: To securely generate the random AES session keys used to encrypt the victim's files. Using the OS's own tools makes the process more reliable and harder to detect.

### 5\. Payload & Resource Management

The malware hides its malicious components within its own executable as "resources" to evade detection.

  * **Functions Used**: `FindResourceA`, `LoadResource`, `LockResource`, `SizeofResource`
  * **Malicious Purpose**:
      * **Self-Extraction**: To unpack embedded payloads (like DLLs) from the main malware file directly into memory at runtime. This stealthy technique avoids dropping malicious files onto the disk.

### 6\. Process Creation & Evasion

To execute different parts of its attack, the malware launches new processes.

  * **Functions Used**: `CreateProcessA`, Dynamic resolution of functions from **`KERNEL32.dll`**
  * **Malicious Purpose**:
      * **Launch Payloads**: To run network scanners, propagation tools, or other malicious components.
      * **Evasion**: By dynamically calling functions from core libraries at runtime, the malware hides its intentions from static analysis tools.

-----

## The Kill Switch: An Accidental Savior

A critical component discovered within WannaCry was a "kill switch," a mechanism that could halt the malware's propagation. This was likely an unintentional flaw in its design that was ultimately used to neutralize the global attack.

### Mechanism of Action

Upon execution, WannaCry performed a crucial check:

1.  **URL Obfuscation**: The malware contained a hardcoded, nonsensical URL: `http://www.iuqerfsodp9ifjaposdfj.com`. Instead of using a standard `strcpy` function, the code copied this URL character by character in a loop, a mild form of obfuscation to evade detection.
2.  **HTTP Request**: Using the **WinINet API**, the ransomware attempted to connect to this URL.

The result of this connection attempt determined the malware's next action:

  * **If the connection failed**, WannaCry proceeded with its attack.
  * **If the connection succeeded**, the program terminated itself.

### Code Analysis: The WinINet Connection

The code responsible for the kill switch initializes the WinINet library and attempts to open the hardcoded URL.

```c
// A loop copies the URL http://www.iuqerfsodp9ifjaposdfj.com into a local buffer.

// 1. Initialize the WinINet API
uVar1 = InternetOpenA(0, 1); // 1 = INTERNET_OPEN_TYPE_DIRECT

// 2. Attempt to open the URL
InternetOpenUrlA(uVar1, &url_buffer, 0, 0, 0x84000000, 0);
```

The flags used in `InternetOpenUrlA` (`0x84000000`) are a combination of `INTERNET_FLAG_RELOAD` and `INTERNET_FLAG_NO_CACHE_WRITE`. These flags ensure the malware gets a fresh response directly from the server, bypassing any local cache.

### Intended Purpose: Anti-Sandbox Technique

The kill switch was likely a crude **anti-sandbox technique**. Many automated analysis environments simulate an internet connection and will respond positively to any network request. The creators likely assumed:

  * A **failed connection** meant it was on a real computer.
  * A **successful connection** meant it was in a sandbox, prompting it to shut down to avoid analysis.

### Discovery and Activation

On May 12, 2017, UK-based security researcher **Marcus Hutchins** (MalwareTech) discovered the domain check while analyzing the malware. He found the domain was unregistered and, realizing its purpose, immediately registered it for $10.69.

By registering the domain and pointing it to a server, Hutchins activated the kill switch globally. Newly infected machines would now successfully connect to the domain and terminate, a single action that is credited with halting the initial, devastating wave of the WannaCry attack.
