# KN01: Virtualisierung & erste automatisierte VM

## A) Repository & Multipass-Test

### A1) Repository

Für alle Abgaben habe ich dieses Repo erstellt. Pro KN gibt es einen Ordner (`KN01/`, `KN02/`, ...). Code-Dateien liegen immer als eigene Datei im Ordner, Screenshots kommen in `img/`.

### A2) Multipass-Test

- Windows-Edition: Windows 11 Pro
- Multipass-Version: _TODO_
- Backend (`local.driver`): _TODO_

Befehle, die ich ausgeführt habe:

```powershell
multipass version
multipass get local.driver
(Get-CimInstance Win32_OperatingSystem).Caption
multipass launch --name smoke -c 1 -m 1G -d 5G 26.04
multipass list
multipass exec smoke -- lsb_release -a
multipass delete --purge smoke
```

![Multipass-Test](img/a2-smoke.png)

## B) Hypervisor-Experiment

### Backends in der Gruppe

| Person | Betriebssystem | Backend | Vermutung | Begründung |
|---|---|---|---|---|
| Nico | Windows 11 Pro | hyperv | Typ 1 | Hyper-V ist fest in Windows eingebaut und läuft direkt auf der Hardware. Windows selber läuft eigentlich auch nur als eine Art VM darüber. |
| _TODO_ | | | | |

### Host-Ressourcen

Mein Rechner hat 22 logische Prozessoren und ca. 63 GB RAM.

![Host-Ressourcen](img/b-host.png)

### Test-VM mit 4 CPUs und 4 GB RAM

```powershell
multipass launch --name test -c 4 -m 4G
multipass exec test -- lscpu
multipass exec test -- free -h
```

![Test-VM](img/b-test.png)

### Mehr CPUs als vorhanden

```powershell
multipass launch --name cpu-test -c 26
multipass exec cpu-test -- lscpu | grep "CPU(s)"
```

![CPU-Test](img/b-cpu-test.png)

### Mehr RAM als vorhanden

```powershell
multipass launch --name ram-test -m 71G
multipass exec ram-test -- free -h
```

![RAM-Test](img/b-ram-test.png)

### Erklärung

_TODO: kommt, wenn die Resultate da sind._

Was man dafür wissen muss:

- **CPU:** Eine virtuelle CPU ist für den Host einfach ein Thread. Der Scheduler verteilt die Threads in kleinen Zeitscheiben auf die echten Kerne. Darum kann man theoretisch mehr vCPUs geben, als es Kerne gibt. Die laufen dann aber nicht wirklich gleichzeitig.
- **RAM:** Wenn der echte RAM nicht reicht, lagert das System Teile auf die Disk aus (Swap). So kann man mehr RAM versprechen, als da ist. Wenn aber alles gleichzeitig gebraucht wird, wird es extrem langsam.

## C) Erste automatisierte VM mit cloud-init

### C1) Basis-Konfiguration

Datei: [cloud-init-basis.yaml](cloud-init-basis.yaml)

```powershell
multipass launch --name kn01 --cloud-init cloud-init-basis.yaml 26.04
multipass shell kn01
```

`neofetch` und `tree` sind installiert, obwohl ich nichts von Hand installiert habe:

![neofetch und tree](img/c1-pakete.png)

Inhalt von `/home/ubuntu/begruessung.txt`:

![begruessung.txt](img/c1-begruessung.png)

- Die ersten zwei Zeilen ("Diese Datei wurde von cloud-init erstellt." und "Niemand hat sie von Hand angelegt.") kommen aus `write_files`. Dort wird die Datei mit diesem Inhalt erstellt.
- Die letzte Zeile "Installation abgeschlossen am ..." kommt aus `runcmd`. Der `echo`-Befehl hängt sie mit `>>` am Schluss an. Weil `runcmd` erst am Ende läuft, steht sie zuunterst.

cloud-init-Log (`cat /var/log/cloud-init-output.log`):

![cloud-init-Log](img/c1-log.png)

Danach habe ich die VM gelöscht: `multipass delete --purge kn01`

### C2) Eigene Konfiguration

Datei: [cloud-init-kn01.yaml](cloud-init-kn01.yaml)

```yaml
#cloud-config
users: # Liste der Benutzer, die beim ersten Start erstellt werden
  - default # Standardbenutzer ubuntu bleibt, sonst geht multipass shell nicht mehr
  - name: nico # mein eigener Benutzer
    sudo: ALL=(ALL) NOPASSWD:ALL # darf sudo ohne Passwort benutzen
    shell: /bin/bash # bash als Shell

package_update: true # Paketliste vor der Installation aktualisieren (apt update)

packages: # diese Pakete werden installiert
  - neofetch # zeigt Systeminfos an (aus der Basis-Datei)
  - tree # zeigt Ordner als Baum an (aus der Basis-Datei)
  - htop # mein zusätzliches Paket, Taskmanager fürs Terminal

write_files: # Dateien, die cloud-init erstellt
  - path: /home/ubuntu/begruessung.txt # Datei aus der Basis-Datei
    content: | # Inhalt der Datei, | heisst mehrere Zeilen
      Diese Datei wurde von cloud-init erstellt.
      Niemand hat sie von Hand angelegt.
  - path: /home/ubuntu/info-nico.txt # meine eigene Datei
    content: | # Inhalt der Datei
      Name: Nico Schult
      Schule: TBZ
      Modul: M346
      Diese VM wurde mit cloud-init aufgesetzt.

runcmd: # Befehle, die am Schluss vom ersten Start ausgeführt werden
  - echo "Installation abgeschlossen am $(date)" >> /home/ubuntu/begruessung.txt # Datum an die Datei anhängen
  - mkdir -p /home/nico/projekte # mein eigener Befehl: Ordner projekte erstellen
  - chown nico:nico /home/nico/projekte # Ordner gehört nico und nicht root
```

Die Syntax habe ich vor dem Start auf yamllint.com geprüft.

```powershell
multipass launch --name kn01v2 --cloud-init cloud-init-kn01.yaml 26.04
multipass shell kn01v2
```

**1. Benutzer nico** (`sudo su - nico`, `whoami`, `echo $SHELL`, `sudo -l`):

![Benutzer nico](img/c2-user.png)

**2. Paket htop** (`htop --version`):

![htop](img/c2-htop.png)

**3. Eigene Datei** (`cat /home/ubuntu/info-nico.txt`):

![info-nico.txt](img/c2-datei.png)

**4. Eigener Befehl** (`ls -l /home/nico`):

![Ordner projekte](img/c2-runcmd.png)

## D) Reflexion: von Hand vs. automatisiert

| Variante | Meine aktive Arbeitszeit |
|---|---|
| Von Hand (VM `manuell`) | _TODO_ |
| Mit cloud-init (ein Befehl) | _TODO_ |

_TODO: Reflexion kommt, wenn die Zeiten gemessen sind._
