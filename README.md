# RetroPie su Raspberry Pi 5 con Display TFT SPI

Guida tecnica di **installazione** e **troubleshooting** per far girare RetroPie / EmulationStation
su **Raspberry Pi 5** (Debian Trixie, kernel 6.x) con un **display TFT SPI da 3.5"** (`tft35a`)
collegato via GPIO.

Su Pi 5 il driver grafico moderno (KMS/DRM) gestisce solo l'HDMI: questa guida spiega come usare
un **"Ponte X11"** minimale per renderizzare EmulationStation sul framebuffer del display SPI,
oltre a configurare il bus SPI e risolvere i crash di RetroArch.

## Guida completa

**[docs/README.md](docs/README.md)**

## Contenuti

- Il problema architetturale (KMS/DRM, Dispmanx, `fbcp`)
- Installazione del Ponte X11
- Configurazione hardware del bus SPI (`tft35a`)
- Trasferimento ROM e controlli
- Troubleshooting (schermo nero, crash RetroArch, artefatti SPI, permessi)

## Licenza

[MIT](LICENSE)
