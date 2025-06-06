---
title: Trigger RCE in Microsoft Skype
description: >-
  Bug Bounty Writeup
author: Cronus
date: 2024-06-23 18:26:00 
categories: [Bug bounty]
tags: [Bug Bounty]
pin: true
image:
  path: /assets/img/bugbounty/skype.png
  width: 300
  height: 300
---

## Executive Summary
I have recently discovered a critical Remote Code Execution (RCE) vulnerability in Skype. This bug presents a severe security risk as it enables an attacker to potentially take control of a victim's computer. Although this finding did not result in a reward or a CVE identification, it has been acknowledged as valid by Microsoft's security team, and I wish to document it for reference and awareness.

![email](https://github.com/Cr0nu3/RCE_Exploit_in_Microsoft_Skype/assets/68406739/72cbca5f-2509-4349-9227-9d49e9add64b)


## Introduction
The inspiration for investigating this vulnerability arose from a well- known incident where an RCE attack was successfully executed against Telegram using a .pyzw file. A member of my team introduced me to this case, sparking my curiosity about the potential applicability of similar exploits to other platforms. Motivated by this, I conducted tests on Skype, albeit with no initial expectations of discovering a parallel vulnerability.

## Exploitation Methodology
The next step is very simple. Upload a malicious file to a Skype chat that is designed to work in the Windows operating system environment. If victim clicks the file, then RCE attack succeeds. To exploit this vulnerability, I crafted a malicious .pyzw file designed to alter the Windows registry. This modification causes the infected system to attempt to connect to an attacker-defined server upon every startup, effectively persisting the malicious activity indefinitely.

The exploitation process is quite straightforward:

The crafted malicious file is uploaded to a Skype conversation, designed to operate within the Windows environment.
The attack is executed successfully when the victim unwittingly clicks and downloads the file.


## Analysis
There have been many reports of successful RCE attacks on other software using pyzw files as above ( e.g. Telegram, KakaoTalk etc.. ). 

![blacklist_file_in_telegram](https://github.com/Cr0nu3/RCE_Exploit_in_Microsoft_Skype/assets/68406739/c8f5afb7-e72a-4a3b-9b3d-801cb08a2f5a)

In Telegram case, there was a typo in blacklist so flitering didn't work appropriately.

Developers usually use codes from other open sources. In my opinnion, same RCE attack can be successful against multiple software, not just Telegram, because the blacklist is often implemented into the software without verification. (as long as the file upload structure is the same)

So, when developers import other source code, they must verify that the code is safe.

## Result
![email_reply](https://github.com/Cr0nu3/RCE_Exploit_in_Microsoft_Skype/assets/68406739/705cd263-88a0-4ffb-b154-66c95cc48483)

Following the report, skype developers added more executable extensions to the existing list in Skype and patched them in a way that warned users when the files were down. 

I thought it was so close but I didn't get a CVE or bounty, but it was a great experience.

<iframe src="https://drive.google.com/file/d/1hfGizshRLFlvvZxEKozupn1GTpq1ODII/preview" width="640" height="480" allow="autoplay"></iframe>