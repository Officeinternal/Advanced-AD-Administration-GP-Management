# Advanced Active Directory Administration & Group Policy Management Lab
## Overview
This lab is a deeper dive into a focus on domain administration, Group Policy engineering, workstation management, and troubleshooting. Instead of showing how to install AD, it documents the actual challenges involved in maintaining a Windows domain — including domain controller failures, secure channel issues, SYSVOL outages, GPO misapplication, drive‑mapping problems, Restricted Groups enforcement, and domain‑join instability.

What began as a basic AD deployment quickly turned into a into rigorous lesson on how Windows domains behave under stress. A straightforward setup escalated into rebuilding the domain controller, reinstalling the client workstation, diagnosing trust failures, and dealing with Windows Server 2025’s instability — ultimately teaching me far more about Active Directory than any guide or textbook could.

This lab builds on my previous project, where I deployed a Windows Server 2025 VM, installed AD DS, configured DNS, and created a clean OU structure modeled after a small corporate environment. From there, the environment expanded into a full enterprise simulation with users, groups, GPOs, drive mappings, folder redirection, AppLocker rules, Restricted Groups, and real troubleshooting incidents that mirror what administrators face in production networks.

## Table of Contents
- [Overview](#overview)
- [Lab Architecture](#lab-architecture)
- [Troubleshooting Incidents](#troubleshooting-incidents)
  - [Drive Mapping Failure](#drive-mapping-failure-root-cause-dc-rename)
  - [Restricted Groups Not Applying](#restricted-groups-not-applying)
- [Lessons Learned](#lessons-learned)



## Lab Architecture

A minimal, realistic Active Directory environment built to simulate a small corporate network:

- **Domain Controller:** Windows Server 2025  
  - AD DS, DNS, SYSVOL/Netlogon  
  - Hosts shared folders and GPOs  

- **Client Workstation:** Windows 11 Enterprise  
  - Domain‑joined  
  - Used for testing GPOs, drive mapping, Restricted Groups, AppLocker, and secure channel behavior  

- **Domain:** `office.internal`  
- **DNS:** AD‑integrated  
- **OU Structure:**  
  - Corp Users (Sales, HR, IT, Executives)  
  - Corp Computers (Workstations, Servers)



The project continued from my last lab (see my profile for that repository) where I deployed a Windows Server 2025 VM and installed Active Directory Domain Services. I created a domain, set up DNS, and built out a clean OU structure that resembled a small corporate environment. 

The first major task was enforcing password policies. I created a new GPO and configured minimum password length,

<img width="811" height="589" alt="Screenshot 2026-07-30 184949" src="https://github.com/user-attachments/assets/c6f97ffc-6ac9-40f8-9415-298c1f9f23c0" />

maximum password age, complexity requirements, and lockout thresholds. 

<img width="791" height="572" alt="Screenshot 2026-07-30 185416" src="https://github.com/user-attachments/assets/c9283af5-f7e3-43b0-bcaa-d0c04e0a9ae1" />

## Troubleshooting Incidents
At first, the policy wouldn’t apply, and I spent a while running gpupdate, checking link order, and verifying inheritance. Eventually, I realized the default domain policy was overriding mine. This was the first instance where I realized just how integral placement is in Active Directory. After adjusting the link order, the policy applied correctly, and running net accounts on the client confirmed the new settings. At the time this is where I had my Security Policy, which later I learned was not optimal and I would organize better.

<img width="878" height="588" alt="Screenshot 2026-07-30 191816" src="https://github.com/user-attachments/assets/a6bf62cf-1f74-43fb-9f4d-be2a07675ad1" />


Dwight immediately tested the lockout policy by entering a one-character password five times and locking himself out. It was a small moment, but it felt like the first real sign that the domain was behaving like an actual enterprise environment.

https://github.com/user-attachments/assets/b834c6c4-716f-4eef-a60a-94ecb90fe877

<img width="973" height="731" alt="Screenshot 2026-07-30 195130" src="https://github.com/user-attachments/assets/c4de6308-caae-4e05-9729-7307530ac55e" />

### Restricted Groups Not Applying
Next came Restricted Groups, and this is where everything started to unravel. My goal was simple: enforce who could be a local administrator on domain-joined machines. I created the GPO, added the Administrators group, and expected it to work. It didn’t. 

<img width="789" height="569" alt="Screenshot 2026-07-30 201201" src="https://github.com/user-attachments/assets/5af59f2b-0408-4a71-90ff-35638e099d95" />

<img width="1106" height="621" alt="Screenshot 2026-07-30 201928" src="https://github.com/user-attachments/assets/15863029-60d2-4be0-af6d-f2a7c3a5a27d" />


Running gpresult /r showed that the workstation was receiving domain GPOs. Since this was my first time, I think I got really mixed up here. Seeing all the different groups had me convinced that something deeper was wrong when I was only expecting the groups I had setup in the DC.

<img width="928" height="692" alt="Screenshot 2026-07-30 212228" src="https://github.com/user-attachments/assets/d06b0a6b-45bd-4c60-8816-bc92a597d6d1" />

This led me down a path of non-stop troubleshooting. I was double-checking Users, Groups, GPO placement, even going into my dns and ip configs to test if there was a mistake I had made. 

<img width="2375" height="1023" alt="Screenshot 2026-08-03 130038" src="https://github.com/user-attachments/assets/68ea1c28-c678-495a-a2b9-3a4371b88013" />

<img width="935" height="680" alt="Screenshot 2026-08-03 131207" src="https://github.com/user-attachments/assets/aa7eae9c-ce61-4822-917e-c497af76d662" />

<img width="900" height="641" alt="Screenshot 2026-08-03 131323" src="https://github.com/user-attachments/assets/86d7e8b8-6b02-4e10-8f83-047c5d862296" />

My troubleshooting ended up advancing me into systems that felt above my knowledge. To compare this to a Helpdesk equivalent, it was as if I escalated the issue to a tier 2/3, maybe even sys admin level. I eventually landed on trying to access SYSVOL, but it failed. 
Next, I checked the secure channel using nltest /sc_verify. It failed too. I checked DNS, Netlogon, Kerberos, and SMB connectivity. Everything pointed upstream — the domain controller itself was failing.
Then the domain controller completely froze. SYSVOL and Netlogon were unavailable. The workstation couldn’t authenticate. Group Policy processing collapsed entirely. This wasn’t a simple GPO issue — it was a full domain meltdown. The workstation lost trust with the domain, and nothing I tried could repair the secure channel.
At this point I was willing to try anything so I tried the most risky move - deleting the DC, not knowing what exactly would happen but hoping that I could simply force a total reset within the server. As you can probably guess, I was locked out of everything on the DC and Client side as the client was connected to a domain that no longer existed so I had no choice but to rebuild both the domain controller and the Windows 11 client from scratch. 

<img width="1012" height="796" alt="Screenshot 2026-08-03 140404" src="https://github.com/user-attachments/assets/1cba137c-cd8a-4cbd-9328-ee7930bd9b9b" />

<img width="1216" height="955" alt="Screenshot 2026-08-12 133406" src="https://github.com/user-attachments/assets/e11713ad-f630-43f3-8550-4a8854f481ca" />


I reinstalled Windows Server, reinstalled AD DS, rebuilt DNS, recreated SYSVOL and Netlogon, and rebuilt all users, groups, OUs, and GPOs. Then I reinstalled Windows 11 and rejoined it to the domain. It was a painful process, but it taught me how fragile domain trust can be and how deeply Group Policy depends on a healthy domain controller, also to create snapshots when settings are stable to avoid massive reinstall time.


[(video)](https://github.com/user-attachments/assets/7885babc-100e-421a-bd46-62e70f6f67c3)


Once everything was rebuilt, I revisited Restricted Groups and discovered the real issue: I had selected the wrong Administrators group. I chose a domain group instead of the local Administrators group, so the policy silently did nothing. After correcting it, the policy applied perfectly. Running net localgroup administrators showed the correct enforced membership, and Dwight was no longer a local admin. It felt like the payoff for all the troubleshooting.

<img width="853" height="274" alt="Screenshot 2026-08-05 155044" src="https://github.com/user-attachments/assets/54d5b599-df14-41a5-bbba-6b714504b060" />
[


(video dwight no longer admin)](https://github.com/user-attachments/assets/14269c7b-8630-4669-90fb-0c48add126ca)


With the core security policies in place, I moved on to user experience and workstation control. I deployed a custom corporate wallpaper through Group Policy 


[(video of wallpaper policy)](https://github.com/user-attachments/assets/91b66a73-dc01-4a27-b534-5335f6f67e48)


<img width="1020" height="769" alt="Screenshot 2026-08-05 175305" src="https://github.com/user-attachments/assets/b41d3b6a-034c-4b91-8ebc-57bd47b41a62" />

<img width="1017" height="731" alt="Screenshot 2026-08-05 175829" src="https://github.com/user-attachments/assets/369fb29d-557b-4e1e-84f9-8847d0c375a3" />


Than I locked down the Control Panel, and experimented with Software Restriction Policies. 

<img width="685" height="640" alt="Screenshot 2026-08-05 180819" src="https://github.com/user-attachments/assets/2a82a535-858b-45ec-b65b-bd15fe649a30" />

<img width="1187" height="718" alt="Screenshot 2026-08-05 190528" src="https://github.com/user-attachments/assets/a3d6a8d3-d225-49f3-a5dd-585508acac23" />


[(control video)](https://github.com/user-attachments/assets/2104de82-e81c-4f2b-9e6b-25c4054c265f)


[(event video)](https://github.com/user-attachments/assets/c29fcf8b-37f7-47d5-996c-675decf02987)


SRP worked somewhat, but that's when I learned of how it was outdated and inconsistent on Windows 11. This is also when I felt that keeping all my security settings under one policy was a rather messy procedure and decided to restructure and organize.

<img width="753" height="527" alt="Screenshot 2026-08-05 204605" src="https://github.com/user-attachments/assets/da0379b0-6082-45e6-ad33-848c16678fed" />

Cleaner but I would later organize the users policies to be in the officeusers folder to keep them separate from computer policies.

<img width="1262" height="867" alt="Screenshot 2026-08-05 221037" src="https://github.com/user-attachments/assets/91261dd2-4e35-4679-9ee6-6dca788dc569" />


With SRP's not working very well it was then where I sought out an alternative and found AppLocker — the modern way to block unauthorized applications. AppLocker required enabling the Application Identity service and creating publisher-based rules. 


[(applocker video)](https://github.com/user-attachments/assets/96d18a55-f310-4fde-9412-d08fd64bde0c)


At first, AppLocker didn’t work because I didn’t have any allow rules, I made use of the auto generate but learned real quick not to rely on that seemingly easier option. 

<img width="1168" height="766" alt="Screenshot 2026-08-05 225055" src="https://github.com/user-attachments/assets/90789f46-b14a-4c78-8130-9d570eb2d3da" />


[(too restrict video)](https://github.com/user-attachments/assets/c94e0aa8-b7ee-4594-8035-d192433d00b1)


A classic case where I had too many restrictions in place (some of those being the auto generated rules) where the client couldn't open anything. Once I added proper allow rules, AppLocker behaved exactly like a locked-down corporate machine, blocking PowerShell, MMC, Command Prompt, and other administrative tools for standard users.

<img width="1021" height="748" alt="Screenshot 2026-08-05 232107" src="https://github.com/user-attachments/assets/31f62199-6363-48eb-b92f-b629d8c7670f" />


[(allowapps vid)](https://github.com/user-attachments/assets/5013a8bd-053a-497c-bad2-4a83fa438c3e)


This was one of the most satisfying moments in the entire lab, having restrictions in place, custom wall paper, users and group permissions operating. I felt like it was a small taste of having a client almost enterprise ready.


[(almostenterprise video)](https://github.com/user-attachments/assets/9f7da3db-33a7-430e-abaf-525a3c74b402)


### Drive Mapping Failure (Root Cause: DC Rename)
This brings us to the next MAJOR issue I had run into with this home lab. Drive mapping. I went through the standard steps of creating a shared folder, shared users folder, and started setting permissions.


[(dripmapbegin vid)](https://github.com/user-attachments/assets/3b78ce69-59b6-4f05-aa97-8c063b1e8d40)


Created the sub-folders within my Shared Folders to separate between the different departments of the organization.


[(sharefolderperm vid)](https://github.com/user-attachments/assets/98ceb6e3-85c7-46ec-88a1-19a63b64f54a)

<img width="791" height="573" alt="Screenshot 2026-08-12 142413" src="https://github.com/user-attachments/assets/eca65069-eaf9-4dd7-8d74-0f7e579fa5ed" />


A simple enough task thinking I was finally going to have an easy part in the lab, but this is when I noticed the share path of my folders. I hadn't named the pc that the Domain Controller was on. Rather than being //DC01, it was //WIN-9MV... etc. Thus did I learn of the trials of trying to rename an established DC and how it would supply yet another instance where everything is locked out, even at an administrative level.

At first everything seemed fine, I renamed the pc to DC01, you can witness the correct folder path I had set in the video. On the client side, I didn't notice any flaring issues...yet.


[(everything seems fine vid)](https://github.com/user-attachments/assets/61ea8616-3952-468c-9586-e2a68e11a68c)

However, upon trying to update the client's policies I would very quickly assess that everything being seemingly fine on the surface does not mean that there isn't a disaster bellowing from below. After the rename, the workstation was stuck in a half-joined state. 

<img width="843" height="629" alt="Screenshot 2026-08-12 150015" src="https://github.com/user-attachments/assets/4efb24e7-a19f-43f4-90b5-7bd7bb15221c" />

I tried to many different commands such as ipconfig /flushdns /release /renew, net start and stop, diag:testdns, etc. Searching the dns management for old records of the a mix up in ip addresses and dns settings that was causing a collision. Dns was still looking at the old host name and I tried to delete and remake Cname and A records after a certain point.
<img width="1271" height="1085" alt="Screenshot 2026-08-12 161618" src="https://github.com/user-attachments/assets/c14d0e05-02e0-4b7c-be51-3d0897cbb744" />

<img width="1055" height="260" alt="Screenshot 2026-08-12 173019" src="https://github.com/user-attachments/assets/a74bdee4-c420-4a74-b217-c4e3d997eb5a" />

<img width="1043" height="526" alt="Screenshot 2026-08-12 173220" src="https://github.com/user-attachments/assets/24d8d237-349c-4265-aadf-a182e757ff6f" />

Eventually I had to make the difficult decision of reverting back to a saved snapshot before I renamed the server and undoing hours of work/trouble shooting. Ultimately as much as I wanted to solve the underlying issue, it was another case where I felt this had surpassed my level of knowledge and I would save much more time with a reset.

It wasn't too much longer after that I redid folders, permissions for sharing and security, and create a new gp policy without changing the DC's name that everything worked and flowed normally.


[(mapdriveend vid)](https://github.com/user-attachments/assets/1fc7d2b5-ac11-4d3b-8d9a-571a026c5572)


Folder redirection was the last thing I wanted to implement for this lab. I redirected Documents and Desktop to a server share with carefully configured NTFS and share permissions. 

<img width="814" height="709" alt="Screenshot 2026-08-12 185016" src="https://github.com/user-attachments/assets/255f1be8-902a-4a21-8b61-e3a551416951" />

<img width="790" height="573" alt="Screenshot 2026-08-12 185401" src="https://github.com/user-attachments/assets/ff08722f-2d99-4249-afc6-6a3abbd098e8" />


Some users, like Dwight and ITAdmin, redirected without issues. 


[(folderredirect vid)](https://github.com/user-attachments/assets/81d5b30a-0613-479e-a541-7e24991343a0)


Others, like Toby, ran into problems. His folder wouldn’t redirect at all, which led to more troubleshooting: checking inheritance, verifying ownership, deleting corrupted profiles, removing broken server folders, reapplying GPOs. I couldn't quite figure out why he lacked the redirect. An authorized user, within the same OU structure as the others but I even tried to log in as Jim and he too lacked the redirect. Jim is in the sales dept, same as Dwight. Folder redirection is extremely sensitive to NTFS permissions, but I triple checked that everything looked and was placed in the correct setting. Which is what sparks up my conclusion to this journey. 


## Lessons Learned
Throughout the entire lab, I ran into issues that forced me to dig deeper into how Active Directory actually works. I learned how to diagnose secure channel failures, how to interpret gpresult HTML reports, how SYSVOL availability affects GPO processing, how AppLocker behaves when allow rules are missing, and how folder redirection silently fails when permissions aren’t perfect. I also learned how easy it is for a domain controller to become unstable in on an administrative level — especially on Windows Server 2025.
And that leads to one of the biggest takeaways from this project: Active Directory I felt for some time was a weak structure, like it was fragile and needed to be treated delicate. Now I sit with the understanding that it's more strict than anything. It follows rules and paths to a T. Windows Server 2025 I feel a bit otherwise as I question if it was stable enough for this kind of lab. I ran into missing ADMX templates, broken File Explorer policies, inconsistent GPO processing, folder redirection bugs, secure channel failures, SYSVOL instability, documentation gaps, and unpredictable behavior. Which was great experience for me as I'm here to learn however I have done a little research and noticed that Server 2022 is widely considered the most stable and reliable version of Windows Server for enterprise environments, and after everything I experienced, I understand why.
This lab wasn’t perfect, and it wasn’t smooth — but that’s exactly why it was valuable. I learned how Active Directory really behaves, how Group Policy actually applies, and how fragile domain trust can be. I learned how to troubleshoot problems that real IT departments deal with every day. Most importantly, I learned that building a homelab isn’t about everything going right — it’s about learning how to fix things when they go wrong.
And this lab gave me plenty of opportunities to do exactly that.
