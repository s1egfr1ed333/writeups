# Symfonos 2
___
### Phase 1: Reconnaissance

Start with an nmap port scan to identify available services:

<figure style="text-align: center;">
    <img src="screenshots/Captura de pantalla 2025-09-10 105917.png">
</figure>

<figure style="text-align: center;">
    <img src="screenshots/Captura de pantalla 2025-09-10 110118.png">
</figure>

Since SMB is available, we enumerate shared resources with the ``anonymous`` user:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 110220.png">
  <img src="screenshots/Captura de pantalla 2025-09-10 110347.png">
</figure>

<figure style="text-align: center;">
    <img src="screenshots/Captura de pantalla 2025-09-10 110509.png">
    <figcaption>First line of the <strong>log.txt</strong> file</figcaption>
</figure>

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 110835.png">
  <figcaption>Another section of the <strong>log.txt </strong>file</figcaption>
</figure>

**Key findings::**

- Accessible ``anonymous`` shared resource
- ``log.txt`` file revealing the existence of ``/var/backups/shadow.bak``
- The shared directory corresponds to ``/home/aeolus/share``

Also identify that the FTP service has a "File Copy" vulnerability that allows moving files using ``site cpfr`` and ``site cpto``.

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 111128.png">
</figure>

---

### Phase 2: Initial Access

##### FTP vulnerability exploitation:

Using the ``File Copy`` vulnerability to move the ``shadow`` file backup to the SMB-accessible directory:

- `site cpfr` (copy from)
- `site cpto` (copy to)

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 111449.png">
  <img src="screenshots/Captura de pantalla 2025-09-10 111911.png">
  <img src="screenshots/Captura de pantalla 2025-09-10 112049.png">
</figure>

Download ``shadow.bak`` via SMB and crack it with ``john``:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 112945.png">
</figure>

Obtained credentials:

- User: `aeolus`
- Password: `sergioteamo`

With these credentials, we gain SSH access to the system:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 113557.png">
</figure>

---

### Phase 3: Post-Exploitation Enumeration

Once inside the system, we enumerate internal services:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 113942.png">
  <img src="screenshots/Captura de pantalla 2025-09-10 114607.png">
</figure>

Port 8080 is running ``LibreNMS``, a web-based network monitoring system. Since it's only accessible locally, we set up ``port forwarding``:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 114758.png">
</figure>

Accessing ``localhost:8080`` shows a LibreNMS login panel. We test credential reuse with ``aeolus:sergioteamo`` and gain successful access.

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 114855.png">
  <img src="screenshots/Captura de pantalla 2025-09-10 115001.png">
</figure>

After identifying an RCE vulnerability in LibreNMS and analysing its code, we proceeded with manual exploitation as the automatic exploit was unreliable:

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 115123.png">
</figure>

Let's go through it step by step:

1. Navigate to `/addhost`
2. Enter the ``Hostname`` (In my case, RCE)
3. Insert payload in the ``Community`` field and click `Add Device`
4. Navigate to `/devices`, select the device just created
5. Go to settings ⚙️ → ``Capture`` → ``SNMP`` → ``Run``

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 121057.png">
  <figcaption>http://localhost:8080/addhost</figcaption>
</figure>

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 121142.png">
  <figcaption>⚙️ -> Capture</figcaption>
</figure>

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 121130.png">
  <figcaption>SNMP -> Capture -> Run</figcaption>
</figure>

With our listener configured ``(nc -lvp 4646)``, we obtain a reverse shell.

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 122421.png">
</figure>

---

### Phase 4: Privilege Escalation

Check sudo permissions for the current user (cronus).
**Finding**: The user can execute ``mysql`` as root without a password.

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 122707.png">
  <figcapture>whoami: root</figcapture>
</figure>

This gives us root privileges and access to the final flag.

<figure style="text-align: center;">
  <img src="screenshots/Captura de pantalla 2025-09-10 122807.png">
</figure>

---

### Attack Vector Summary

##### Exploitation chain:

**SMB Anonymous Access** → Information about shadow backup
**FTP File Copy Vulnerability** → Access to shadow.bak file
**Password Cracking** → Valid credentials
**SSH Access** → Foothold in the system
**LibreNMS RCE** → Reverse shell
**MySQL Sudo Privileges** → Escalation to root

---

### Remediation Recommendations

**SMB Access**: Disable anonymous access or restrict permissions
**FTP Service**: Update to version without File Copy vulnerability
**Credential Management**: Implement robust password policies
**Internal Services**: Isolate critical web applications through network segmentation, firewalls, or containerization to limit blast radius
**Sudo Permissions**: Apply principle of least privilege, avoid direct MySQL access as root