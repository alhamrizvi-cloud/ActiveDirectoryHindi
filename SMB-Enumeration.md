# SMB Enumeration (Hinglish Notes)

**Category:** Enumeration
**CPTS Module:** Attacking Common Services / Nmap Enumeration

## Concept samjho

Dekho yaar, jab bhi tumhe kisi bhi AD machine pe koi bhi valid credential mil jaye — chahe wo random low-priv user hi kyun na ho — sabse pehle **SMB shares check karo**. SMB (port 445/139) basically Windows ka file-sharing protocol hai, aur yahi jagah pe log apni sabse badi galtiyan chhod dete hain — jaise ek share pe WRITE access de dena jo dena hi nahi chahiye tha.

Bas ye samajh lo: **agar koi share tumhe WRITE bhi de raha hai, na sirf READ** — to wo tumhara agla foothold delivery point ban sakta hai. Wahan pe tum malicious file daal sakte ho, exploit upload kar sakte ho, waiting game khel sakte ho.

## Tools ka roster

| Tool | Kaam kya karta hai | Type |
|---|---|---|
| `nmap` | SMB service/version detect karta hai, NSE scripts se enum karta hai | Scanner |
| `smbmap` | Sabse fast — shares + unke permission level ek saath dikhata hai | Enumeration |
| `smbclient` | FTP jaisa interactive client — browse/upload/download | Interaction |
| `netexec` (nxc) / `crackmapexec` | Bulk creds test karo, ek saath multiple hosts pe shares dekho | Enumeration/Attack |
| `enum4linux-ng` | Null session se bhi info nikal leta hai (users, groups, policies) | Enumeration |

## Commands — cheat sheet

```bash
# Nmap se SMB enum
nmap -p 139,445 --script smb-enum-shares,smb-enum-users,smb-os-discovery <target_ip>

# smbmap — shares + permissions ek nazar mein
smbmap -H <target_ip> -u '<user>' -p '<pass>'

# smbclient — shares list karo
smbclient -L //<target_ip>/ -U '<user>'

# smbclient — kisi share pe ghuso
smbclient //<target_ip>/<share> -U '<user>'
# andar jaake:
smb: \> ls
smb: \> get <file>
smb: \> put <file>

# enum4linux-ng — full sweep
enum4linux-ng -A <target_ip>

# netexec — quick shares check
nxc smb <target_ip> -u '<user>' -p '<pass>' --shares
```

## Practical Example — Fluffy (HTB) se seedha

Humein `j.fleischman` ke creds mile the (given tha challenge mein). Sabse pehle yahi kiya:

```bash
smbmap -H 10.10.11.69 -u 'j.fleischman' -p 'J0elTHEM4n1990!'
```

*Screenshot reference: `htb_fluffy_smb_sharedir.png` — output mein `IT` naam ka share dikha jisme READ, WRITE dono mile the, baaki sab sirf READ the.*

Fir us share pe connect kiya taaki dekh sake andar kya hai (aage chalke yahi share exploit deliver karne ke kaam aaya — dekho `NTLM-Hash-Leak-Coercion.md` file mein):

```bash
smbclient //10.10.11.69/IT -U j.fleischman
```

**Yaad rakhna:** Is poori chain ka starting point yehi tha — writable share dhoondhna. Agar ye miss kar dete, to aage kuch nahi hota. Isliye har share ka **permission level** zaroor check karo, sirf naam dekh ke aage mat badho.
