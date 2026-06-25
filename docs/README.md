# RetroPie su Raspberry Pi 5 con Display TFT SPI

Manuale tecnico di installazione e troubleshooting per far girare **RetroPie / EmulationStation**
su un **Raspberry Pi 5** con sistema operativo basato su **Debian Trixie** (kernel 6.x),
usando un **display TFT SPI da 3.5"** (controller `tft35a`) collegato via GPIO.

---

## Indice

- [Il problema architetturale](#il-problema-architetturale)
- [Requisiti](#requisiti)
- [Installazione: il "Ponte X11"](#installazione-il-ponte-x11)
  - [1. Pacchetti X11 e legacy](#1-pacchetti-x11-e-legacy)
  - [2. Permessi e routing sul framebuffer](#2-permessi-e-routing-sul-framebuffer)
  - [3. Disabilitazione Display Manager e boot testuale](#3-disabilitazione-display-manager-e-boot-testuale)
  - [4. Autostart (il "grilletto")](#4-autostart-il-grilletto)
- [Configurazione hardware: bus SPI (`tft35a`)](#configurazione-hardware-bus-spi-tft35a)
- [Utilizzo: ROM e controlli](#utilizzo-rom-e-controlli)
- [Aprire un terminale sullo schermino](#aprire-un-terminale-sullo-schermino)
- [Troubleshooting](#troubleshooting)

---

## Il problema architetturale

Sui Raspberry Pi 5 con Debian Trixie (kernel 6.x), il driver grafico moderno
**KMS/DRM** (`vc4-kms-v3d`) indirizza l'accelerazione hardware **unicamente alle porte HDMI**.

Conseguenze:

- I display **SPI** (collegati via GPIO, es. TFT 3.5" con driver `tft35a`) **non vengono rilevati**
  dal sistema DRM.
- Il vecchio software **`fbcp`** (Framebuffer Copy) **non è più supportato**: Broadcom ha rimosso
  le API hardware legacy (**Dispmanx**).
- Risultato: all'avvio di EmulationStation si ottengono **schermate nere** o **blocchi**
  (deadlock su LLVMpipe).

**La soluzione:** installare un server **X11 minimale** (senza Desktop Environment) e obbligarlo
a renderizzare direttamente sul framebuffer **`/dev/fb0`**.

---

## Requisiti

- Raspberry Pi 5
- OS basato su Debian Trixie (kernel 6.x) con RetroPie installato
- Display TFT SPI 3.5" con controller compatibile `tft35a`
- Accesso a un terminale (locale o via SSH)

> Gli esempi usano l'utente `noya`. Sostituiscilo con il tuo username dove necessario.

---

## Installazione: il "Ponte X11"

### 1. Pacchetti X11 e legacy

```bash
sudo apt install -y xorg xserver-xorg-video-fbdev xinit xserver-xorg-legacy
```

### 2. Permessi e routing sul framebuffer

Permetti l'avvio di X da console a un utente non-root:

```bash
sudo sed -i 's/allowed_users=console/allowed_users=anybody/' /etc/X11/Xwrapper.config
sudo sed -i 's/needs_root_rights=no/needs_root_rights=yes/' /etc/X11/Xwrapper.config
```

Forza X11 sul framebuffer locale, ignorando l'HDMI mancante:

```bash
sudo bash -c 'cat > /usr/share/X11/xorg.conf.d/99-fbdev.conf << EOF
Section "Device"
    Identifier "Schermo_SPI"
    Driver "fbdev"
    Option "fbdev" "/dev/fb0"
EndSection
EOF'
```

### 3. Disabilitazione Display Manager e boot testuale

Evita l'avvio di interfacce desktop non richieste (es. LightDM) e imposta il boot in modalità testuale:

```bash
sudo systemctl stop lightdm
sudo systemctl disable lightdm
sudo systemctl set-default multi-user.target
```

### 4. Autostart (il "grilletto")

Abilita il **Console Autologin**:

```bash
sudo raspi-config
# System Options -> Boot / Auto Login -> Console Autologin
```

Aggiungi l'avvio del server X nel profilo utente:

```bash
nano /home/noya/.bash_profile
```

In fondo al file inserisci:

```bash
if [ -z "$DISPLAY" ] && [ -n "$XDG_VTNR" ] && [ "$XDG_VTNR" -eq 1 ]; then
  exec startx /usr/bin/emulationstation -- -nocursor
fi
```

> **Nota:** rimuovi eventuali vecchi file di autostart in
> `/opt/retropie/configs/all/autostart.sh`, altrimenti possono entrare in conflitto.

---

## Configurazione hardware: bus SPI (`tft35a`)

Per migliorare gli **FPS** e gestire l'orientamento del TFT, configura il bus SPI nel file
**`/boot/firmware/config.txt`**.

> **Attenzione alla sintassi:** i parametri dell'overlay vanno separati da **virgole**,
> non da due punti.

Sintassi corretta e sicura per il modulo `tft35a` (16 MHz):

```ini
dtoverlay=tft35a,rotate=90,speed=16000000,fps=60
```

| Parametro          | Significato |
|--------------------|-------------|
| `tft35a`           | Carica i driver kernel specifici per il controller di questo TFT. |
| `rotate=90`        | Ruota l'output di 90° (modalità orizzontale / Landscape). Valori possibili: `0`, `90`, `180`, `270` in base al montaggio fisico. |
| `speed=16000000`   | Frequenza del bus SPI a 16 MHz. |
| `fps=60`           | Frame per secondo target. |

> Se imposti una frequenza troppo alta (es. 64 MHz) e lo schermo mostra **artefatti**
> ("spazzatura" a video) o il kernel si **blocca**: estrai la MicroSD, inseriscila in un PC,
> apri la partizione **`bootfs`** e riporta `speed` a `16000000` nel `config.txt`.

---

## Utilizzo: ROM e controlli

### Trasferimento ROM (via `scp`)

Dal PC verso il Raspberry:

```bash
scp "Nome_Gioco.zip" noya@retropi:/home/noya/RetroPie/roms/NOME_CONSOLE/
```

Dopo il trasferimento, **riavvia EmulationStation** dal menu principale per aggiornare la lista giochi.

### Chiusura sicura dell'emulatore

| Metodo                  | Comando |
|-------------------------|---------|
| Controller              | Hotkey + Start (spesso **Select + Start**) |
| Tastiera                | **ESC** |
| Terminale di emergenza (SSH) | `killall retroarch` |

> **Mai scollegare brutalmente l'alimentazione** durante un blocco: rischi la corruzione
> della partizione FAT32 di avvio.

---

## Aprire un terminale sullo schermino

Dato che EmulationStation gira **dentro X11** (`startx`), il modo più comodo per avere un
terminale **direttamente sul display SPI** — senza passare per le console virtuali (`Alt+F2`,
`Alt+F3`, ...) — è lanciare un emulatore di terminale **nella stessa sessione X**,
comandabile da EmulationStation.

### Metodo consigliato: terminale come "Port"

1. Installa `xterm` (leggerissimo, non richiede un Desktop Environment):

   ```bash
   sudo apt install -y xterm
   ```

2. Crea un launcher nella sezione **Ports** di RetroPie:

   ```bash
   cat > /home/noya/RetroPie/roms/ports/Terminal.sh << 'EOF'
   #!/bin/bash
   # Apre xterm a tutto schermo sul TFT; chiudendo la shell (exit) si torna a EmulationStation
   xterm -fa "Monospace" -fs 14 -geometry 9999x9999+0+0 -e bash
   EOF
   chmod +x /home/noya/RetroPie/roms/ports/Terminal.sh
   ```

   > La `-geometry 9999x9999+0+0` viene ridimensionata automaticamente da xterm alla
   > risoluzione reale del display: in pratica equivale a "tutto schermo".

3. **Riavvia EmulationStation.** Nella sezione **Ports** comparirà la voce **Terminal**:
   selezionandola con il controller o con `Invio` si apre il terminale sul TFT. Digiti i
   comandi con una tastiera collegata e, una volta finito, con `exit` (o `Ctrl+D`) chiudi
   la shell e torni alla lista giochi.

### Alternativa da tastiera (console di emergenza)

La guardia in `.bash_profile` lancia X **solo sulla VT1**. Premendo `Alt+F2` ottieni quindi
una normale shell di login su un'altra console, che **non** riavvia EmulationStation; con
`Alt+F1` torni al display di gioco. È il metodo "di emergenza", utile se X si blocca, ma
richiede comunque la tastiera fisica e i tasti `Alt+F#`.

---

## Troubleshooting

### Schermo nero o blocco all'avvio di EmulationStation
Il display SPI non viene gestito dal driver DRM moderno. Verifica di aver completato il
[Ponte X11](#installazione-il-ponte-x11): server X minimale + `99-fbdev.conf` puntato su `/dev/fb0`.

### RetroArch crasha all'avvio di un gioco (video/audio)
Dentro il Ponte X11, RetroArch cerca accelerazione 3D hardware e scheda audio HDMI, andando in crash
(**ALSA Error 524**, **Segmentation Fault**).

Forza il rendering software e l'audio dummy nel config globale di RetroArch:

```bash
sudo sed -i '/video_driver/d' /opt/retropie/configs/all/retroarch.cfg
sudo sed -i '/audio_driver/d' /opt/retropie/configs/all/retroarch.cfg
echo 'video_driver = "sdl2"'   >> /opt/retropie/configs/all/retroarch.cfg
echo 'audio_driver = "dummy"'  >> /opt/retropie/configs/all/retroarch.cfg
```

### Artefatti grafici / kernel bloccato dopo modifica SPI
Frequenza `speed` troppo alta nel `config.txt`. Riporta `speed=16000000` editando la partizione
`bootfs` della MicroSD da un altro PC (vedi
[Configurazione hardware](#configurazione-hardware-bus-spi-tft35a)).

### Si avvia un desktop invece di EmulationStation
LightDM non è stato disabilitato o esiste un vecchio autostart. Riesegui i passi
[3](#3-disabilitazione-display-manager-e-boot-testuale) e
[4](#4-autostart-il-grilletto), e rimuovi `/opt/retropie/configs/all/autostart.sh`.

### Errore di permessi all'avvio di `startx`
Controlla `/etc/X11/Xwrapper.config`: `allowed_users=anybody` e `needs_root_rights=yes`
(vedi [passo 2](#2-permessi-e-routing-sul-framebuffer)).
