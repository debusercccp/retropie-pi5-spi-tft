# RetroPie su Raspberry Pi 5 con Display TFT SPI

Manuale tecnico di installazione e troubleshooting per far girare **RetroPie / EmulationStation**
su un **Raspberry Pi 5** con sistema operativo basato su **Debian Trixie** (kernel 6.x),
usando un **display TFT SPI da 3.5"** (controller `tft35a`) collegato via GPIO.

La configurazione finale è **auto-commutante**: all'accensione il Pi rileva se c'è un cavo HDMI
e manda EmulationStation sulla **TV** (se collegata) oppure sul **TFT SPI** (se assente).

---

## Indice

- [Il problema architetturale](#il-problema-architetturale)
- [Requisiti](#requisiti)
- [Installazione: il "Ponte X11" con auto-switch](#installazione-il-ponte-x11-con-auto-switch)
  - [1. Pacchetti X11 e legacy](#1-pacchetti-x11-e-legacy)
  - [2. Permessi X e file di configurazione del TFT](#2-permessi-x-e-file-di-configurazione-del-tft)
  - [3. Disabilitazione Display Manager e boot testuale](#3-disabilitazione-display-manager-e-boot-testuale)
  - [4. Autostart con auto-switch TV/TFT](#4-autostart-con-auto-switch-tvtft)
- [Configurazione hardware: bus SPI (`tft35a`)](#configurazione-hardware-bus-spi-tft35a)
- [Utilizzo: ROM e controlli](#utilizzo-rom-e-controlli)
- [Aprire un terminale sullo schermino](#aprire-un-terminale-sullo-schermino)
- [Troubleshooting](#troubleshooting)

---

## Il problema architetturale

Sui Raspberry Pi 5 con Debian Trixie (kernel 6.x), il driver grafico moderno
**KMS/DRM** (`vc4-kms-v3d`) indirizza l'accelerazione hardware **unicamente alle porte HDMI**.

Conseguenze:

- Le uscite **HDMI** funzionano nativamente: EmulationStation gira senza problemi su una TV/monitor.
- I display **SPI** (collegati via GPIO, es. TFT 3.5" con driver `tft35a`) **non vengono gestiti**
  dal sistema DRM: restano un framebuffer legacy `fbdev` (`/dev/fbN`).
- Il vecchio software **`fbcp`** (Framebuffer Copy) **non è più supportato**: Broadcom ha rimosso
  le API hardware legacy (**Dispmanx**).
- Risultato: avviando EmulationStation in modo nativo sul TFT si ottengono **schermate nere** o
  **blocchi** (deadlock su LLVMpipe).

**La soluzione:** un server **X11 minimale** (senza Desktop Environment) che renderizza
EmulationStation sul framebuffer del TFT (`fbdev`). Questo Ponte X11 serve **solo per il TFT**.
Poiché l'HDMI funziona da solo, lo stesso script di avvio **rileva all'accensione se c'è un cavo
HDMI collegato** e sceglie dove (e come) mandare EmulationStation:

| All'accensione | Come parte EmulationStation |
|---|---|
| **HDMI collegato** | sulla **TV**, **diretto su KMS** (senza X) |
| **Nessun HDMI** | sul **TFT SPI**, tramite **Ponte X11** (`fbdev`) |

> **Perché sull'HDMI niente X.** Su Pi, il RetroArch di RetroPie è compilato **senza driver video
> X11**: il suo unico contesto video è **KMS**, quindi all'avvio di un gioco vuole prendere il
> controllo diretto del display. Se EmulationStation girasse dentro X sulla TV, X terrebbe occupato
> il "master" DRM dell'HDMI e RetroArch fallirebbe il modeset (`[KMS]: Error when switching mode`),
> uscendo subito: schermo nero di qualche secondo e ritorno a EmulationStation. Lanciando ES
> **diretto su KMS** (come nel RetroPie standard), il display resta libero e i giochi partono.
> Il Ponte X11 si usa solo sul TFT, dove non c'è KMS e RetroArch ripiega su `sdl2`.

> **Doppio schermo simultaneo — non supportato con `tft35a`.** Non è possibile avere
> *contemporaneamente* ES sulla TV **e** un secondo output (es. un terminale) sul TFT con il driver
> `tft35a`/`fbdev`: X enumera come schede video solo i device **DRM**, e il TFT fbtft non lo è,
> quindi non può essere agganciato come secondo schermo accanto alla GPU HDMI. Per avere un
> terminale mentre giochi in TV si usa **SSH** (vedi
> [Aprire un terminale](#aprire-un-terminale-sullo-schermino)). Esiste una via "DRM" sperimentale
> (`dtoverlay=piscreen,drm`, driver kernel `ili9486`) che la renderebbe possibile, ma richiede di
> tarare l'init del pannello — vedi [Troubleshooting](#doppio-schermo-tv--tft-simultaneo).

---

## Requisiti

- Raspberry Pi 5
- OS basato su Debian Trixie (kernel 6.x) con RetroPie installato
- Display TFT SPI 3.5" con controller compatibile `tft35a`
- Accesso a un terminale (locale o via SSH)

> Gli esempi usano l'utente `noya`. Sostituiscilo con il tuo username dove necessario.

---

## Installazione: il "Ponte X11" con auto-switch

### 1. Pacchetti X11 e legacy

```bash
sudo apt install -y xorg xserver-xorg-video-fbdev xinit xserver-xorg-legacy xterm
```

### 2. Permessi X e file di configurazione del TFT

Permetti l'avvio di X e lascia che giri con i privilegi necessari a gestire il framebuffer:

```bash
sudo sed -i 's/allowed_users=console/allowed_users=anybody/' /etc/X11/Xwrapper.config
grep -q '^needs_root_rights=' /etc/X11/Xwrapper.config \
  && sudo sed -i 's/^needs_root_rights=.*/needs_root_rights=yes/' /etc/X11/Xwrapper.config \
  || echo 'needs_root_rights=yes' | sudo tee -a /etc/X11/Xwrapper.config
```

> X avviato **da root** accetta `-config` **solo con percorso relativo** dentro `/etc/X11/` (non
> percorsi assoluti). Per questo il file di configurazione del TFT va creato lì.

Crea il file di config del TFT in `/etc/X11/` e rendilo scrivibile dal tuo utente, così
l'autostart può rigenerarlo a ogni avvio (il numero del framebuffer `/dev/fbN` può cambiare):

```bash
sudo touch /etc/X11/spi.conf
sudo chown noya:noya /etc/X11/spi.conf
```

### 3. Disabilitazione Display Manager e boot testuale

Evita l'avvio di interfacce desktop non richieste (es. LightDM) e imposta il boot in modalità testuale:

```bash
sudo systemctl stop lightdm
sudo systemctl disable lightdm
sudo systemctl set-default multi-user.target
```

### 4. Autostart con auto-switch TV/TFT

Abilita il **Console Autologin**:

```bash
sudo raspi-config
# System Options -> Boot / Auto Login -> Console Autologin
```

Aggiungi in fondo a `/home/noya/.bash_profile` la logica che, all'accensione, rileva l'HDMI e
avvia EmulationStation sullo schermo giusto:

```bash
if [ -z "$DISPLAY" ] && [ -n "$XDG_VTNR" ] && [ "$XDG_VTNR" -eq 1 ]; then

    HDMI_STATUS=$(cat /sys/class/drm/card*-HDMI-A-*/status 2>/dev/null | grep -m 1 "^connected")

    if [ -n "$HDMI_STATUS" ]; then
        # HDMI collegato: EmulationStation diretto su KMS (niente X).
        # RetroArch su Pi usa solo il contesto video KMS e ha bisogno del
        # display libero: dentro X non puo' disegnare (X11 driver assente).
        sed -i '/video_driver/d; /audio_driver/d' /opt/retropie/configs/all/retroarch.cfg
        printf 'video_driver = "gl"\naudio_driver = "alsa"\n' >> /opt/retropie/configs/all/retroarch.cfg
        exec emulationstation
    else
        # Nessun HDMI: EmulationStation sul TFT SPI tramite Ponte X11 (fbdev).

        # Rileva il framebuffer del TFT SPI (fbtft) per nome: il numero /dev/fbN può cambiare
        TFT_DEV="/dev/fb1"
        for f in /sys/class/graphics/fb*; do
            if grep -qs "fb_ili9486" "$f/name"; then
                TFT_DEV="/dev/$(basename "$f")"
            fi
        done

        # Config X "solo TFT" (fbdev), rigenerata a ogni avvio sul framebuffer corretto
        {
            printf 'Section "ServerFlags"\n    Option "AutoAddGPU" "false"\nEndSection\n'
            printf 'Section "Device"\n    Identifier "TFT_SPI"\n    Driver "fbdev"\n    Option "fbdev" "%s"\nEndSection\n' "$TFT_DEV"
            printf 'Section "Screen"\n    Identifier "Screen_TFT"\n    Device "TFT_SPI"\nEndSection\n'
            printf 'Section "ServerLayout"\n    Identifier "Solo_TFT"\n    Screen "Screen_TFT"\nEndSection\n'
        } > /etc/X11/spi.conf

        sed -i '/video_driver/d; /audio_driver/d' /opt/retropie/configs/all/retroarch.cfg
        printf 'video_driver = "sdl2"\naudio_driver = "dummy"\n' >> /opt/retropie/configs/all/retroarch.cfg
        exec startx /usr/bin/emulationstation -- -config spi.conf -nocursor
    fi
fi
```

Come funziona:

- **HDMI collegato** — `exec emulationstation` lancia ES **diretto su KMS** (come nel RetroPie
  standard), senza X: il display HDMI resta libero per RetroArch (`video_driver = "gl"`, audio
  `alsa`). Avviarlo dentro X farebbe fallire il modeset KMS di RetroArch e i giochi non partirebbero.
- **Nessun HDMI** — il Ponte X11 entra in gioco solo qui: ES sul TFT con `video_driver = "sdl2"`
  (rendering software) e audio `dummy`, che evita i crash di RetroArch dentro X.
- **`fb_ili9486`** — individua il framebuffer del TFT anche se l'ordine `fb0`/`fb1` cambia tra un
  avvio e l'altro (con l'HDMI collegato la GPU di solito prende `fb0` e il TFT `fb1`).
- **`spi.conf` con `printf`** — il file di config X viene generato riga per riga con `printf`
  (niente heredoc), così è robusto anche se l'autostart viene incollato in un terminale.

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

Ci sono due situazioni:

- **Mentre giochi sulla TV (HDMI collegato):** usa semplicemente **SSH** dal PC/telefono —
  `ssh noya@retropi`. È il modo più pratico, e l'unico possibile col TFT che resta spento (il
  doppio schermo simultaneo non è supportato, vedi
  [il problema architetturale](#il-problema-architetturale)).
- **Quando usi il Pi in portatile (ES sul TFT, niente HDMI):** apri un terminale **direttamente
  sullo schermino** con il metodo qui sotto, senza passare per le console virtuali (`Alt+F2`...).

In modalità TFT EmulationStation gira **dentro X11** (`startx`), quindi basta lanciare un emulatore
di terminale **nella stessa sessione X**, comandabile da EmulationStation. (Sulla TV invece ES gira
diretto su KMS senza X: lì il terminale si ottiene via SSH, vedi sopra.)

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
[Ponte X11](#installazione-il-ponte-x11-con-auto-switch): server X minimale + `/etc/X11/spi.conf`
puntato sul framebuffer giusto. Controlla quale `/dev/fbN` è il TFT con:

```bash
for f in /sys/class/graphics/fb*; do echo "$f -> $(cat $f/name 2>/dev/null)"; done
```

Il TFT è quello con nome **`fb_ili9486`**: se non è `/dev/fb1`, è proprio per questo che lo script
di avvio lo rileva per nome invece di usare un numero fisso.

### Sulla TV il gioco parte, schermo nero per qualche secondo e torna a EmulationStation
RetroArch sta partendo ma muore subito. Lancialo a mano da SSH per vedere l'errore:

```bash
DISPLAY=:0 XAUTHORITY=/home/noya/.Xauthority \
/opt/retropie/emulators/retroarch/bin/retroarch \
  -L /opt/retropie/libretrocores/lr-mgba/mgba_libretro.so \
  --config /opt/retropie/configs/gba/retroarch.cfg --verbose \
  "/percorso/del/gioco.gba" 2>&1 | tail -30
```

Se nel log vedi:

```
[INFO] [GL]: Found GL context: "kms".
[ERROR] [KMS]: Error when switching mode.
[ERROR] [Video]: Cannot open threaded video driver.. Exiting..
```

allora EmulationStation sta girando **dentro X** sulla TV. Su Pi il RetroArch di RetroPie ha solo
il contesto video **KMS** (nessun driver X11: verificalo con
`retroarch --features | grep -E 'X11|KMS'`), quindi vuole prendere il display direttamente, ma X
tiene occupato il master DRM dell'HDMI e il modeset fallisce. La soluzione **non** è cambiare il
`video_driver`: è avviare ES **diretto su KMS** sulla TV (`exec emulationstation`, senza `startx`),
come fa il [`.bash_profile` con auto-switch](#4-autostart-con-auto-switch-tvtft).

### RetroArch crasha all'avvio di un gioco sul TFT (video/audio)
Dentro il Ponte X11 (modalità TFT), RetroArch cerca accelerazione 3D hardware e scheda audio HDMI,
andando in crash (**ALSA Error 524**, **Segmentation Fault**).

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
(vedi [passo 2](#2-permessi-x-e-file-di-configurazione-del-tft)). Se nel log di X
(`/var/log/Xorg.0.log`) trovi `Invalid argument for -config ... must specify a relative path`,
significa che stai passando un percorso **assoluto** a un X che gira da root: tieni `spi.conf`
in `/etc/X11/` e richiamalo come `-config spi.conf` (relativo).

### Doppio schermo TV + TFT simultaneo
**Non supportato con il driver `tft35a`/`fbdev`.** Avviando un solo server X con due schermi
(`modesetting` per l'HDMI + `fbdev` per il TFT), X carica `fbdev` ma poi lo **scarta**: enumera
come schede video solo i device **DRM** (`/dev/dri/cardN`) e il TFT fbtft non lo è, quindi non
viene agganciato come secondo schermo. Nel log compare `Falling back to old probe method for fbdev`
seguito da `UnloadModule: "fbdev"`.

Per il terminale mentre giochi sulla TV usa **SSH**. Se vuoi davvero ES sulla TV **e** un secondo
output sul TFT in contemporanea, l'unica strada è rendere il TFT un device **DRM**:

```ini
# in /boot/firmware/config.txt, al posto di dtoverlay=tft35a,...
dtoverlay=piscreen,drm,rotate=90,speed=16000000
```

Questo carica il driver kernel `ili9486` e fa comparire un `/dev/dri/card2` (`ili9486drmfb`),
gestibile da `modesetting` come secondo schermo o tramite multiseat di `logind`. Attenzione: l'init
del pannello va tarato (può restare **bianco** se i parametri non combaciano), e con questo display
non è banale — per questo la configurazione di riferimento resta `tft35a` con l'auto-switch.
