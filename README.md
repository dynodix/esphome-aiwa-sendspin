# ESPHome AIWA Sendspin

An ESP32-S3 network stereo line-output player for Home Assistant and Music Assistant, using an external PCM5102MK2.0 I²S DAC.

## Verified configuration

- ESPHome 2026.11.0
- ESP32-S3-WROOM-1
- PCM5102MK2.0 powered from 5 V with 3.3 V I²S logic
- Music Assistant output: FLAC, 48 kHz, 16-bit stereo
- Play, Pause, and Stop through the Home Assistant MediaPlayers provider

## Wiring

| ESP32-S3 | PCM5102MK2.0 |
|---|---|
| 5V | VCC |
| GND | GND |
| GPIO4 | LRCK |
| GPIO5 | BCK |
| GPIO6 | DATA |

Leave `FLT` unconnected and keep jumper `J1` in the I2S position. The analog output is line level; connect it to the AUX/LINE-IN input of powered speakers or an amplifier.

## Secrets

This repository intentionally does not contain `secrets.yaml`. Create it locally in ESPHome with at least:

```yaml
wifi_ssid: "YOUR_NETWORK_NAME"
wifi_password: "YOUR_NETWORK_PASSWORD"
```

`secrets.yaml` is excluded by `.gitignore` and must never be committed.

## Files

- `aiwa-sendspin.yaml` — verified configuration without credentials.
- `diagnostika.md` — detailed HTTP stream analysis, the cause of `ESP_ERR_NOT_SUPPORTED`, and the verified fix (Slovenian).


