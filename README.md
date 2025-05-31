# Icecream - HackMyVM (Easy)

![icecream.png](icecream.png)

## Übersicht

*   **VM:** Icecream
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Icecream)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 09. Oktober 2024
*   **Original-Writeup:** https://alientec1908.github.io/Icecream_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die Challenge "Icecream" auf der Plattform HackMyVM ist eine als "Easy" eingestufte virtuelle Maschine. Ziel war es, Root-Rechte auf dem System zu erlangen. Der Lösungsweg umfasste die Enumeration von SMB-Diensten, das Ausnutzen eines schreibbaren SMB-Shares, der auf das `/tmp`-Verzeichnis des Webservers gemappt war, um eine Webshell hochzuladen und RCE zu erlangen. Anschließend wurde eine ungesicherte Nginx Unit Control API für weitere Enumeration genutzt und schließlich eine fehlerhafte `sudoers`-Konfiguration ausgenutzt, um Root-Rechte zu erlangen.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `curl`
*   `nikto`
*   `enum4linux`
*   `smbclient`
*   `wget`
*   `nc` (Netcat)
*   `ss`
*   `chisel`
*   `python3 -m http.server`
*   `gobuster`
*   Standard Linux-Befehle (`ls`, `cat`, `find`, `chmod`, `echo`, `sudo`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Icecream" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   Identifizierung der Ziel-IP (`192.168.2.131`) und des Hostnamens (`icecream.hmv`).
    *   Portscans mit Nmap enthüllten offene Ports: UDP 137 (NetBIOS-NS) und TCP 22 (SSH - OpenSSH 9.2p1), 80 (HTTP - Nginx 1.22.1), 139 (NetBIOS-SSN), 445 (Microsoft-DS), 9000 (unbekannter Dienst, später als Nginx Unit Control API identifiziert).

2.  **Web & SMB Enumeration:**
    *   Der Webserver auf Port 80 zeigte eine `403 Forbidden`-Seite. `nikto` fand fehlende Sicherheitsheader.
    *   `enum4linux` und `smbclient` identifizierten einen SMB-Share namens `icecream` (Kommentar "tmp Folder"), der anonym les- und schreibbar war und auf das `/tmp`-Verzeichnis des Servers zeigte. Benutzer `ice` und `nobody` wurden enumeriert.

3.  **Initial Access (RCE via SMB Upload & Webserver Misconfiguration):**
    *   Eine PHP-Webshell (`rev.php`) wurde über den `icecream`-SMB-Share in das `/tmp`-Verzeichnis des Servers hochgeladen.
    *   Durch direkten Aufruf von `http://icecream.hmv/rev.php` konnte die Webshell ausgeführt werden (RCE als `www-data`), da der DocumentRoot des Nginx-Webservers auf `/tmp` konfiguriert war.
    *   Eine interaktive Reverse Shell als Benutzer `www-data` wurde etabliert.

4.  **Interne Service Discovery & Port Forwarding:**
    *   Der Dienst auf Port 9000 wurde als Nginx Unit Control API identifiziert.
    *   `chisel` wurde verwendet, um Port 9000 der Zielmaschine auf die Angreifer-Maschine weiterzuleiten (`localhost:9000`), um die API genauer zu untersuchen.
    *   Die API war ungesichert und lieferte Informationen über geladene Sprachmodule (Python, PHP, etc.).

5.  **Privilege Escalation (von `www-data` zu `ice` und dann zu `root`):**
    *   Der genaue Übergang von `www-data` zu `ice` wurde im Detail nicht explizit im Writeup gezeigt, erfolgte aber vermutlich durch Ausnutzung der Nginx Unit API oder einer anderen Schwachstelle/Fehlkonfiguration, um Code als `ice` auszuführen oder dessen Credentials zu erlangen.
    *   Als Benutzer `ice` wurde mittels `sudo -l` festgestellt, dass der Befehl `/usr/sbin/ums2net` mit `NOPASSWD` als `root` ausgeführt werden darf.
    *   Das Programm `ums2net` wurde missbraucht, um die Datei `/etc/sudoers` zu modifizieren. Eine `config`-Datei wurde erstellt, die `ums2net` anwies, `/etc/sudoers` über einen Netzwerkport (8889) beschreibbar zu machen.
    *   Mittels `nc` wurde die Zeile `ice ALL=(ALL) NOPASSWD: ALL` in die `/etc/sudoers`-Datei auf dem Zielsystem geschrieben.
    *   Anschließend konnte mit `sudo su -` eine Root-Shell erlangt werden.

## Wichtige Schwachstellen und Konzepte

*   **Unsicherer SMB-Share:** Anonymer Lese- und Schreibzugriff auf einen SMB-Share (`icecream`), der auf das systemweite `/tmp`-Verzeichnis gemappt war.
*   **Webserver-Fehlkonfiguration (DocumentRoot):** Der DocumentRoot des Nginx-Webservers zeigte auf `/tmp`, was die Ausführung der über SMB hochgeladenen Webshell ermöglichte.
*   **Ungesicherte Nginx Unit Control API:** Die Control API auf Port 9000 war ohne Authentifizierung zugänglich und lieferte detaillierte Informationen über den Server und seine Module, was potenziell für weitere Angriffe hätte genutzt werden können (im Writeup primär für Enumeration genutzt, bevor der `sudo`-Weg gefunden wurde).
*   **Fehlkonfiguration in `sudoers`:** Eine unsichere `sudo`-Regel erlaubte dem Benutzer `ice` die Ausführung von `/usr/sbin/ums2net` als `root` ohne Passwort. Dieses Programm konnte wiederum dazu missbraucht werden, beliebige Dateien (hier `/etc/sudoers`) zu überschreiben.
*   **Port Forwarding (Chisel):** Verwendung von `chisel` zur Weiterleitung eines internen Dienstes (Nginx Unit API) auf die Angreifer-Maschine zur genaueren Untersuchung.

## Flags

*   **User Flag (`/home/ice/user.txt` - Annahme basierend auf typischen Pfaden):** `USER_FLAG_WERT_PLATZHALTER` (Bitte im Original-Writeup nachsehen oder ergänzen)
*   **Root Flag (`/root/root.txt` - Annahme basierend auf typischen Pfaden):** `ROOT_FLAG_WERT_PLATZHALTER` (Bitte im Original-Writeup nachsehen oder ergänzen)

## Tags

`HackMyVM`, `Icecream`, `Easy`, `SMB`, `Webserver Misconfiguration`, `RCE`, `Nginx Unit`, `Chisel`, `Sudo Misconfiguration`, `Privilege Escalation`, `Linux`
