# Decode - HackMyVM (Easy)

![Decode.png](Decode.png)

## Übersicht

*   **VM:** Decode
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Decode)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 07. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Decode_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Easy"-Challenge war es, Root-Zugriff auf der Maschine "Decode" zu erlangen. Der Angriff begann mit der Enumeration eines Webservers (nginx), auf dem durch LFI und XXE (in `magic.php`) ein Certificate Signing Request (`decode.csr`) ausgelesen wurde. Aus diesem CSR konnte ein "challengePassword" (`i4mD3c0d3r`) extrahiert werden, das als SSH-Passwort für den Benutzer `steve` diente. Als `steve` wurde eine `sudo`-Regel entdeckt, die es erlaubte, `tee` als Benutzer `decoder` auszuführen. Dies wurde (vermutlich) genutzt, um einen eigenen SSH-Public-Key in die `authorized_keys`-Datei des Benutzers `ajneya` zu schreiben. Nach erfolgreichem SSH-Login als `ajneya` wurde eine weitere `sudo`-Regel gefunden: `ajneya` durfte `/usr/bin/ssh-keygen` als `root` ohne Passwort ausführen. Diese Schwachstelle wurde mittels Library Preloading ausgenutzt (`ssh-keygen -D /pfad/zur/boesartigen.so`), um eine Root-Shell zu erlangen und die Flags zu lesen.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `dirsearch`
*   `curl`
*   `wget` (implizit)
*   `nikto` (nicht explizit im Log, aber gängig)
*   Burp Suite (oder manuelles POST für XXE)
*   `CertLogik Decoder` (externes Tool)
*   `ssh`
*   `ssh-keygen` (Client-Seite und als Exploit-Vektor)
*   `scp`
*   `tee` (als Exploit-Vektor)
*   `msfvenom`
*   `nc` (netcat)
*   `sudo` (auf Zielsystem, auch `doas` erwähnt, aber `sudo` ist der Exploit-Pfad)
*   Standard Linux-Befehle (`vi`, `cat`, `ls`, `grep`, `id`, `cd`, `chmod`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Decode" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration (XXE & LFI):**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.113`, Hostname `decode.hmv`/`system.hmv`).
    *   `nmap`-Scan identifizierte SSH (22/tcp) und HTTP (80/tcp, nginx). `robots.txt` verwies auf `/encode/` und enthielt Hinweise auf LFI (`Allow: ../`, `Allow: /lfi../`).
    *   `gobuster` und `dirsearch` fanden u.a. `/decode/` und `magic.php`.
    *   LFI über `/decode../` wurde bestätigt und erlaubte das Auflisten von Verzeichnissen und das Lesen von Dateien wie `/etc/passwd`. Benutzer `steve` und `ajneya` wurden identifiziert.
    *   Parallel wurde (vermutlich) eine XXE-Schwachstelle in `magic.php` ausgenutzt, um eine Zertifikatsdatei `/usr/share/ssl-cert/decode.csr` auszulesen.

2.  **Initial Access (SSH als steve):**
    *   Der Inhalt des `decode.csr` wurde mit einem Online-Decoder analysiert. Daraus wurde das "challengePassword" `i4mD3c0d3r` extrahiert.
    *   Mit diesen Credentials (`steve:i4mD3c0d3r`) wurde ein erfolgreicher SSH-Login als Benutzer `steve` durchgeführt.

3.  **Lateral Movement (von steve zu ajneya):**
    *   Als `steve` wurde mittels `sudo -l` entdeckt, dass Befehle (`/usr/bin/openssl enc`, `/usr/bin/tee`) als Benutzer `decoder` ohne Passwort ausgeführt werden dürfen.
    *   Ein neues SSH-Schlüsselpaar wurde auf dem Angreifer-System generiert.
    *   Der öffentliche SSH-Schlüssel wurde (vermutlich mittels `sudo -u decoder /usr/bin/tee /home/ajneya/.ssh/authorized_keys`) in die `authorized_keys`-Datei des Benutzers `ajneya` geschrieben.
    *   Ein SSH-Login als `ajneya` mit dem privaten Schlüssel war erfolgreich.

4.  **Privilege Escalation (von ajneya zu root):**
    *   Als `ajneya` wurde mittels `sudo -l` entdeckt, dass `/usr/bin/ssh-keygen` als `root` ohne Passwort ausgeführt werden darf.
    *   Eine bösartige Shared Library (`lib.so`) mit einer Reverse-Shell-Payload wurde mit `msfvenom` erstellt und via `scp` nach `/tmp/lib.so` auf das Zielsystem übertragen.
    *   Ein Netcat-Listener wurde auf dem Angreifer-System gestartet.
    *   Auf dem Zielsystem wurde `sudo /usr/bin/ssh-keygen -D /tmp/lib.so` ausgeführt.
    *   Dies lud die bösartige Library als `root` und etablierte eine Reverse Shell zum Listener des Angreifers, wodurch Root-Zugriff erlangt wurde.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI):** Ermöglichte das Lesen von Dateien und das Auflisten von Verzeichnissen.
*   **XML External Entity (XXE) Injection:** (Vermutlich verwendet) zum Auslesen der CSR-Datei.
*   **Information Disclosure in CSR:** Das "challengePassword" im CSR führte direkt zu Benutzer-Credentials.
*   **Unsichere `sudo`-Regeln:**
    *   `steve` durfte `tee` als `decoder` ausführen, was (vermutlich) für Lateral Movement genutzt wurde.
    *   `ajneya` durfte `ssh-keygen` als `root` ausführen, was durch Library Preloading (`-D` Option) zur Root-Eskalation führte.
*   **Library Preloading / Hijacking:** Ausnutzung der `-D` Option von `ssh-keygen` in einer `sudo`-Regel.
*   **Unsichere SSH-Konfiguration / Berechtigungen:** Ermöglichte das Schreiben in die `authorized_keys`-Datei eines anderen Benutzers.

## Flags

*   **User Flag (`/home/ajneya/user.txt`):** `ee11cbb19052e40b07aac0ca060c23ee`
*   **Root Flag (`/root/root.txt`):** `63a9f0ea7bb98050796b649e85481845`

## Tags

`HackMyVM`, `Decode`, `Easy`, `Web`, `nginx`, `LFI`, `XXE`, `CSR`, `Information Disclosure`, `SSH`, `sudo`, `tee`, `ssh-keygen`, `Library Preloading`, `msfvenom`, `Privilege Escalation`, `Linux`
