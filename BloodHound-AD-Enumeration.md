# BloodHound - AD Attack Path Enumeration (Hinglish Notes)

**Category:** AD Enumeration
**CPTS Module:** Active Directory Enumeration & Attacks

## Concept samjho

Yaar jab bhi tumhe koi bhi domain credential mil jaye - chahe kितna hi low-priv kyun na ho - sabse pehla kaam ye karo: **BloodHound chalao**. Ye tool poori AD environment ka ek graph bana deta hai - users, groups, computers, sessions, aur sabse important, **kisme kisme permissions (ACLs) hain** ye sab. Manually ACL dhoondhna scale hi nahi karta, isliye ye tool literal lifesaver hai.

Bas ek query maaro aur BloodHound khud bata dega "shortest path to Domain Admin kya hai" ya "ye account kya-kya control kar sakta hai". CPTS ke AD-heavy machines mein ye tool chalana basically compulsory habit hai.

## Tools ka roster

| Tool | Kaam | Type |
|---|---|---|
| `bloodhound-python` | Python collector - Linux attack box se bhi chal jaata hai | Collector |
| `SharpHound` | Native C# collector, Windows/domain context se chalta hai | Collector |
| `BloodHound` (GUI) | Collected data visualize/query karne ke liye | Analysis |
| `BloodHound CE` | Newer containerized version | Analysis |

## Commands - cheat sheet

```bash
# Data collect karo (Linux se)
bloodhound-python -u '<user>' -p '<pass>' -d <domain> -ns <dc_ip> -c All --zip

# SharpHound se (agar Windows foothold hai)
SharpHound.exe -c All

# Fir zip file ko BloodHound GUI mein import karo, aur:
#  - Apna controlled user "Owned" mark karo
#  - "Outbound Object Control" dekho ki wo kya control kar sakta hai
#  - Built-in queries use karo: "Shortest Paths to Domain Admins" wagera
```

**Ye edges dhoondhte raho:** `GenericAll`, `GenericWrite`, `WriteOwner`, `WriteDacl`, `ForceChangePassword`, `AddMember`, `AllowedToDelegate`, `AddAllowedToAct`. Ye sab red flags hain, jab bhi dikhein, samajh jao attack path hai.

## Practical Example - Fluffy (HTB) se seedha

`p.agila` ka cracked password mil chuka tha (NTLM leak se, dekho `NTLM-Hash-Leak-Coercion.md`). Pura collection chalaya:

```bash
bloodhound-python -u 'p.agila' -p 'promXXXXXXXXXXX' -d fluffy.htb -ns 10.10.11.69 -c All --zip
```

GUI mein analysis kiya to pata chala `p.agila` **SERVICE ACCOUNTS** group ka member tha.

**Sample BloodHound query result (Cypher-style summary):**
```
Node: p.agila (User)
MemberOf -> SERVICE ACCOUNTS (Group)
```

Us group ke paas kai service accounts pe `GenericWrite` tha, jisme `winrm_svc` bhi tha.

**Sample "Outbound Object Control" panel output:**
```
SERVICE ACCOUNTS (Group) --GenericWrite--> winrm_svc (User)
SERVICE ACCOUNTS (Group) --GenericWrite--> ca_svc (User)
```

Aur bhi deep dekha to pata chala group ke paas `ca_svc` pe bhi write permission cascade ho rahi thi (same panel output jaisa upar dikhaya).

**Yaad rakhna:** BloodHound hi wo cheez tha jisne ek single cracked password ko poore escalation path mein badal diya. Isne jo `GenericWrite` edges dikhaye, wahi seedha Shadow Credentials attack mein use hue (dekho `Shadow-Credentials-Attack.md`).
