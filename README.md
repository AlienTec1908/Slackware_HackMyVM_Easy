# Slackware - HackMyVM (Easy)

![Slackware.png](Slackware.png)

## Übersicht

*   **VM:** Slackware
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Slackware)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2024-04-25
*   **Original-Writeup:** https://alientec1908.github.io/Slackware_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Slackware" zu erlangen. Der Weg dorthin begann mit der Entdeckung eines Webservers auf dem nicht standardmäßigen Port 2, der eine PHP-Anwendung (Version 5.6.40) und einen Hinweis auf den Benutzer `mozes` hostete. Im Webverzeichnis `/getslack/` wurde ein mehrteiliges 7z-Archiv (`twitter.7z.00*`) gefunden. Nach dem Herunterladen und Extrahieren dieses Archivs wurde eine Bilddatei (`twitter.png`) erhalten, aus der mittels `strings` das Passwort `trYth1sPasS1993` extrahiert wurde. Dies ermöglichte den SSH-Login (auf dem nicht standardmäßigen Port 1) als Benutzer `patrick`. Eine Kette von Fehlkonfigurationen bei Gruppenberechtigungen und Klartextpasswörtern in `mypass.txt`-Dateien in den Home-Verzeichnissen erlaubte Lateral Movement durch mehrere Benutzer (`patrick` -> `claor` -> `alienum` -> ... -> `rpj7`). Die User-Flag wurde im Home-Verzeichnis von `rpj7` gefunden. Das Root-Passwort (`To_Jest_Bardzo_Trudne_Haslo`) war mittels Whitespace-Steganographie in der `user.txt`-Datei von `rpj7` versteckt und konnte mit `stegsnow` extrahiert werden, was den `su root`-Zugriff ermöglichte. Die Root-Flag wurde dann aus `/home/0xh3rshel/.screenrc` gelesen.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi`
*   `nmap`
*   `nikto`
*   `dirb`
*   `gobuster`
*   `ssh`
*   `curl` (impliziert)
*   `openssl` (für s_client)
*   `wfuzz`
*   `7z`
*   `strings`
*   `grep`
*   `sudo` (versucht)
*   `find`
*   `su`
*   `scp`
*   `file`
*   `stegsnow`
*   `stegsolve` (impliziert für Bildanalyse)
*   Standard Linux-Befehle (`cat`, `ls`, `id`, `cd`, `chmod`, `wget`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Slackware" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.107) mit `arp-scan` identifiziert. Hostname `slackware` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 1 (SSH, OpenSSH 9.3) und Port 2 (HTTP, Apache 2.4.58 mit PHP 5.6.40).
    *   `nikto` und `gobuster` auf Port 2 fanden `index.html`, `robots.txt`, `background.jpg` und das Verzeichnis `/getslack/`. Ein Kommentar deutete auf den Benutzer `mozes` hin.
    *   `wfuzz` auf `/getslack/` fand `twitter.7z.001`, was auf ein mehrteiliges 7z-Archiv hindeutete.

2.  **Steganography & Initial Access (SSH als `patrick`):**
    *   Mittels eines Bash-Skripts wurden die Teile `twitter.7z.001` bis `twitter.7z.014` von `http://192.168.2.107:2/getslack/` heruntergeladen.
    *   Das Archiv wurde mit `7z x twitter.7z.001` extrahiert und enthielt die Datei `twitter.png`.
    *   `strings twitter.png -n 12` extrahierte das Passwort `trYth1sPasS1993` aus dem Bild.
    *   Erfolgreicher SSH-Login auf Port 1 als Benutzer `patrick` mit dem gefundenen Passwort.

3.  **Privilege Escalation (Lateral Movement Kette `patrick` -> `rpj7`):**
    *   Als `patrick` (Mitglied der Gruppe `kretinga`) konnte `/home/claor/mypass.txt` (Gruppe `kretinga`) gelesen werden, was das Passwort für `claor` (`JRksNe5rWgis`) enthüllte.
    *   Mit `su claor` wurde zu `claor` gewechselt.
    *   Dieser Prozess wurde wiederholt, da jeder Benutzer Mitglied der Gruppe des nächsten Benutzers in der Kette war und das Passwort des nächsten Benutzers in `mypass.txt` im jeweiligen Home-Verzeichnis gespeichert war.
    *   Die Kette führte über mehrere Benutzer (impliziert: `claor` -> `alienum` -> `mrmidnight` -> ... -> `b4el7d`) schließlich zum Benutzer `rpj7` (Passwort `wP26CtkDby6J` aus `b4el7d`'s `mypass.txt`).
    *   Die User-Flag (`HMV{Th1s1s1Us3rFlag}`) wurde in `/home/rpj7/user.txt` gefunden.

4.  **Root Access (Steganographie in User-Flag & `su root`):**
    *   Die Datei `/home/rpj7/user.txt` wurde auf das Angreifer-System kopiert (`scp`).
    *   Mittels `stegsnow -C user.txt` wurde ein in der User-Flag-Datei durch Whitespace-Steganographie verstecktes Passwort extrahiert: `To_Jest_Bardzo_Trudne_Haslo`.
    *   Mit `su root` und dem Passwort `To_Jest_Bardzo_Trudne_Haslo` wurde Root-Zugriff erlangt.
    *   Die Root-Flag (`HMV{SlackwareStillAlive}`) wurde in `/home/0xh3rshel/.screenrc` gefunden (lesbar als Root).

## Wichtige Schwachstellen und Konzepte

*   **Dienste auf Nicht-Standard-Ports:** SSH auf Port 1, HTTP auf Port 2.
*   **Veraltete PHP-Version:** PHP 5.6.40 ist End-of-Life und potenziell anfällig.
*   **Exponiertes Archiv mit Steganographie:** Ein 7z-Archiv auf dem Webserver enthielt ein Bild (`twitter.png`), in dem ein Passwort mittels `strings` gefunden wurde.
*   **Fehlkonfigurierte Gruppenberechtigungen & Klartextpasswörter:** Eine Kette von Benutzern, bei der jeder Benutzer das Home-Verzeichnis des nächsten lesen und dessen Passwort aus einer `mypass.txt`-Datei extrahieren konnte.
*   **Steganographie (Whitespace):** Das Root-Passwort war mittels Whitespace-Steganographie in der User-Flag-Datei versteckt und konnte mit `stegsnow` extrahiert werden.
*   **Information Disclosure:** Hinweise auf Benutzernamen in Kommentaren. Root-Flag in einer ungewöhnlichen Datei (`.screenrc`) im Home eines anderen Benutzers gespeichert.

## Flags

*   **User Flag (`/home/rpj7/user.txt`):** `HMV{Th1s1s1Us3rFlag}`
*   **Root Flag (`/home/0xh3rshel/.screenrc`):** `HMV{SlackwareStillAlive}`

## Tags

`HackMyVM`, `Slackware`, `Easy`, `Non-Standard Ports`, `Steganography`, `7z Archive`, `Strings Exploit`, `Group Permission Exploit`, `Cleartext Passwords`, `Lateral Movement`, `Whitespace Steganography`, `stegsnow`, `Linux`, `Web`, `Privilege Escalation`, `Apache`, `OpenSSH`
