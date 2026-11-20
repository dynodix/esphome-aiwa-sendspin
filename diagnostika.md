# Diagnostika AIWA Sendspin – Music Assistant / ESPHome

Datum diagnostike in popravka: 20. november 2026  
Način dela: najprej diagnostika samo za branje, nato z izrecnim dovoljenjem uporabnika sprememba ESPHome YAML in OTA namestitev prek Wi-Fi. Home Assistant in Music Assistant nista bila ponovno zagnana.

## Povzetek

Vzrok napake je potrjen. Music Assistant predvajalniku ne pošlje MP3 ali WAV, temveč neprekinjeni HTTP flow stream v formatu **FLAC, 48 kHz, 16-bit, stereo**. Trenutni ESPHome firmware ima glasbeni cevovod preveden samo za **MP3** (`media_pipeline.format: MP3`), obvestilni cevovod pa za WAV. Ker podpora FLAC ni vključena v firmware, ESPHome ne prepozna niti `Content-Type: audio/flac` niti pripone `.flac`. `AudioReader` zato vrne natančno `ESP_ERR_NOT_SUPPORTED` še pred dekodiranjem.

Napaka ni posledica omrežne nedosegljivosti, HTTP preusmeritve ali napačnega MIME tipa. MIME, pripona in dejanski podpis vsebine se vsi ujemajo: FLAC.

## Končni rezultat – potrjeno delovanje

Predlagani popravek je bil izveden in preverjen:

- `media_pipeline.format` je bil spremenjen iz `MP3` v `FLAC`;
- glasbeni cevovod in fizični I²S izhod sta bila nastavljena na 48 kHz, 16-bit, stereo;
- MP3-specifični `bitrate: 128` je bil odstranjen;
- `buffer_size: 32768` je ostal nespremenjen;
- firmware se je uspešno prevedel s podporo `microFLAC` in je bil OTA nameščen prek Wi-Fi;
- ESPHome je nato zabeležil `Reading FLAC file type`, dekodiranje 2 kanalov pri 48.000 Hz/16 bit in zagon I²S zvočnika brez `ESP_ERR_NOT_SUPPORTED`;
- prvotni mali vijolični PCM5102A modul kljub pravilnemu digitalnemu toku ni dal uporabnega analognega izhoda;
- po zamenjavi z modulom **PCM5102MK2.0** (napajanje 5 V, 3,3-V I²S logika; LRCK=GPIO4, BCK=GPIO5, DATA=GPIO6, skupna masa) je uporabnik potrdil delujoč stereo linijski izhod.

Končni test iz Music Assistanta je ostal v stanju `PLAYING`, glasnost je bila 94 %, dekoder in I²S sta tekla brez napake, uporabnik pa je potrdil: **zvok deluje**.

## 1. Dostop, namestitev in uporabljeni viri

- Music Assistant instanca je dostopna prek HA ingressa `https://<HA_HOST>:8123/<MA_INGRESS>`.
- Ingress pot `/api/hassio_ingress/...` in HA Action history z vzrokom `Supervisor` potrjujeta, da Music Assistant teče kot dodatek v Home Assistantu.
- Music Assistant uporablja provider **Home Assistant MediaPlayers** za fizični predvajalnik.
- Pravilni ID fizičnega predvajalnika je `media_player.aiwa_sendspin_aiwa_sendspin`.
- HA vsebuje tudi MA-izpostavljeno zrcalno entiteto `media_player.aiwa_sendspin_aiwa_sendspin_2`; njen `media_content_id` je knjižnični URI (`library://track/...`) in ni URL, poslan ESP32.
- ESPHome Builder je potrdil ESPHome 2026.11.0 in povezavo z napravo `<ESP_IP>` prek ESPHome API.
- SSH strežnik HA na `<HA_IP>:22` je bil dosegljiv, toda ta Codex seja ni imela uporabnikovega zasebnega ključa ali delujočega SSH agenta, zato SSH prijava ni bila uporabljena. Diagnostika je bila izvedena prek že prijavljenih HA/MA/ESPHome spletnih vmesnikov.
- Povezani MA diagnostični/MCP vmesnik je kazal drugo instanco in ni poznal pravilnega player ID-ja, zato njegovi podatki niso bili uporabljeni za zaključke.

## 2. Identificiran izvor zvočnega toka

HA Action history je ob svežem pritisku Play zabeležil klic:

```yaml
action: media_player.play_media
target:
  entity_id: media_player.aiwa_sendspin_aiwa_sendspin
data:
  media_content_id: http://<HA_IP>:8097/flow/<SESSION>/media_player.aiwa_sendspin_aiwa_sendspin/<STREAM-ID>/media_player.aiwa_sendspin_aiwa_sendspin.flac
  media_content_type: music
  enqueue: replace
  extra:
    bypass_proxy: true
```

`<SESSION>` in `<STREAM-ID>` sta anonimizirana. URL je začasen; po koncu seje isti URL pravilno vrne `404 Unknown (or invalid) session`.

Music Assistantov prikaz avdio cevovoda je za isti poskus pokazal:

- vhod: MP3, 44,1 kHz, 16-bit, stereo, 128 kb/s;
- obdelava: 32-bit float PCM, 48 kHz, stereo, z normalizacijo glasnosti;
- izhod proti AIWA: FLAC, 48 kHz, 16-bit, stereo, približno 1.147 kb/s.

To pojasni, zakaj izvorna datoteka MP3 ne pomeni, da MA predvajalniku pošlje MP3. Zaradi flow načina in obdelave MA ustvari nov FLAC tok.

## 3. HTTP analiza aktivnega URL-ja

Aktivni URL je bil preverjen takoj po svežem Play. Prenos je bil omejen na 65.536 bajtov.

```text
HTTP/1.1 200 OK
Server: Music Assistant
Content-Type: audio/flac
Transfer-Encoding: chunked
Accept-Ranges: none
Connection: close
Cache-Control: no-cache
Pragma: no-cache
```

Rezultati zahtevanih preverjanj:

| Lastnost | Rezultat |
|---|---|
| HTTP status | `200 OK` med aktivno sejo; `404` po poteku seje |
| Content-Type | `audio/flac` |
| Content-Length | ni prisoten |
| Transfer-Encoding | `chunked` |
| Content-Encoding | ni prisoten |
| Accept-Ranges | `none` |
| Preusmeritve | ni preusmeritve |
| Končni naslov | `<HA_IP>:8097`, neposredno v istem lokalnem omrežju |
| Podpis vsebine | prvi štirje bajti `66 4C 61 43`, ASCII `fLaC` |

Range zahteva je bila ignorirana skladno z `Accept-Ranges: none` in je vrnila `200`, ne `206`. Tok je zato namenoma neprekinjen/chunked. To samo po sebi ni problem za ESPHomeov `esp_http_client`.

Dostop iz ESP32 je praktično potrjen tudi s samo vrsto napake: povezovalna napaka bi vrnila napako odpiranja/HTTP povezave, medtem ko se `ESP_ERR_NOT_SUPPORTED` v tej kodi vrne po prejemu glav, ko tip datoteke ostane neprepoznan.

## 4. ESPHome 2026.11.0 – zakaj vrne ESP_ERR_NOT_SUPPORTED

Pregledan je bil lokalni uradni vir taga `2026.11.0` (commit `1feda1bcaa861a5f2e933f2c65d599e44f4b132b`).

Relevantna logika:

1. `audio_reader.cpp` odpre HTTP URL, sprejme statuse 200–308 in podpira do pet preusmeritev.
2. Ob glavi `Content-Type` pokliče `detect_audio_file_type()`.
3. `audio.cpp` prepozna `audio/flac`, `audio/x-flac` in pripono `.flac` **samo**, če je firmware preveden z `USE_AUDIO_FLAC_SUPPORT`.
4. Če tip po preverjanju Content-Type in URL-ja ostane `AudioFileType::NONE`, `audio_reader.cpp` vrne `ESP_ERR_NOT_SUPPORTED`.
5. `audio_pipeline.cpp` to napako izpiše kot `Media reader encountered an error: ESP_ERR_NOT_SUPPORTED`.

Trenutni YAML:

```yaml
speaker:
  - platform: i2s_audio
    channel: stereo
    sample_rate: 16000
    bits_per_sample: 16bit

media_player:
  - platform: speaker
    buffer_size: 32768
    media_pipeline:
      format: MP3
      sample_rate: 16000
      num_channels: 2
      bitrate: 128
    announcement_pipeline:
      format: WAV
      sample_rate: 16000
      num_channels: 2
```

Ta konfiguracija vključi MP3 in WAV, ne pa FLAC. Music Assistant pa pošlje `audio/flac` z `.flac` URL-jem. Ker sta obe FLAC prepoznavni veji pogojno odstranjeni pri prevajanju, tip ostane `NONE` in bralnik vrne napako na točno opaženi vrstici.

### Kaj ni vzrok

- **MIME ni napačen:** `audio/flac` je pravilen.
- **Vsebina ni preoblečen MP3/WAV:** podpis je `fLaC`.
- **Ni preusmeritve:** aktivni URL neposredno vrne 200.
- **Ni 403/avtentikacijske napake:** aktivni URL je dostopen brez dodatnega HTTP žetona.
- **Chunked prenos ni potrjen vzrok:** ESPHome uporablja `esp_http_client`; napaka se zgodi pri prepoznavi formata, preden pride do dekodiranja telesa.
- **`format: WAV` ali `format: MP3` v ESPHome ne naroči Music Assistantu, naj transkodira v ta format.** Pri HA MediaPlayers providerju MA še vedno pošlje svoj FLAC flow output.

## 5. Predlagani najmanj invazivni popravek

Najbolj neposreden in robusten popravek je uskladitev ESPHome z dejanskim formatom MA, brez sprememb Music Assistanta ali načina upravljanja:

```yaml
speaker:
  - platform: i2s_audio
    id: pcm5102a_speaker
    i2s_audio_id: i2s_bus
    dac_type: external
    i2s_dout_pin: GPIO6
    channel: stereo
    sample_rate: 48000       # prej 16000
    bits_per_sample: 16bit
    buffer_duration: 100ms

media_player:
  - platform: speaker
    id: aiwa_player
    name: "AIWA Sendspin"
    buffer_size: 32768       # za prvi preizkus ostane nespremenjeno

    media_pipeline:
      speaker: music_input
      format: FLAC           # prej MP3
      sample_rate: 48000     # usklajeno z dejanskim MA tokom
      num_channels: 2
      # odstrani: bitrate: 128 (MP3-specifično)

    announcement_pipeline:
      speaker: announcement_input
      format: WAV
      sample_rate: 16000
      num_channels: 2
```

Zakaj ta različica:

- `format: FLAC` zahteva in vključi FLAC podporo, zato ESPHome prepozna `audio/flac` in `.flac`.
- 48 kHz ohrani dejanski MA stereo tok brez nepotrebnega pretvarjanja na 16 kHz.
- 16-bit stereo je skladen z MA tokom in PCM5102A.
- `buffer_size: 32768` ostane nespremenjen, da se RAM ne poveča brez dokazane potrebe.
- Play/Pause/Stop ostanejo prek istega HA MediaPlayers providerja; Action history je že potrdil pravilne klice `play_media`, `media_pause` in `media_stop`.

### RAM in PSRAM

V trenutnem YAML ni komponente `psram:`. Iz oznake `ESP32-S3-WROOM-1` ni mogoče zanesljivo sklepati, ali je fizični modul različica z vgrajenim PSRAM (odvisno od polne končnice modula, npr. R2/R8). Zato PSRAM ni del osnovnega popravka.

Če je na modulu potrjen PSRAM, je smiselno pozneje dodati pravilno `psram:` konfiguracijo in po potrebi usmeriti FLAC medpomnilnike v PSRAM. Tega ne naredi brez preverjanja polne oznake modula. Če po osnovnem popravku pride do `ESP_ERR_NO_MEM`, je to ločen problem in takrat je naslednji korak PSRAM oziroma previdno povečanje/razmestitev medpomnilnikov.

### Alternativa v Music Assistantu

Napredna možnost `Prefer low-latency WAV for live sources` je trenutno izklopljena, vendar njen opis izrecno pravi, da velja za žive vire, kot sta Spotify Connect in AirPlay Receiver. Ne rešuje zanesljivo običajnega predvajanja skladb iz MA čakalne vrste. V pregledanem UI za HA MediaPlayers ni bilo nastavitve, ki bi običajni flow output prisilila v MP3. Zato je sprememba ESPHome na FLAC manj invazivna in bolj predvidljiva.

## 6. Postopek preverjanja po ročni spremembi

1. Pred spremembo shrani kopijo trenutnega YAML.
2. Uporabi zgornje tri spremembe: fizični izhod 48 kHz, `format: FLAC`, media pipeline 48 kHz; odstrani MP3 `bitrate`.
3. V ESPHome najprej izvedi samo Validate/Compile in preveri, da ni napak; šele nato po lastni odločitvi namesti firmware.
4. Po zagonu odpri ESPHome logs in v Music Assistantu izberi AIWA Sendspin.
5. Predvajaj navadno skladbo. Pričakovano:
   - ni več `ESP_ERR_NOT_SUPPORTED`;
   - log pokaže prepoznan/bran FLAC in dekodiran stereo 48 kHz/16-bit tok;
   - stanje ostane `PLAYING` in ne preide takoj v `IDLE`;
   - na PCM5102A je stereo zvok.
6. Preizkusi Pause, Resume in Stop iz MA. HA Action history mora pokazati ustrezne klice, ESPHome pa stanja `PAUSED`, `PLAYING`, `IDLE` brez reader napak.
7. Preizkusi še WAV obvestilo; 16 kHz announcement pipeline naj ostane delujoč prek mešalnika/resamplerja.
8. Spremljaj opozorila `ESP_ERR_NO_MEM`, underrun in reset. Če jih ni, obdrži 32 KiB buffer. Buffer povečuj samo ob dejanskih underrunih.

## Potrjeno in še nepotrjeno

### Potrjeno

- MA pošilja začasni HTTP flow URL na `<HA_IP>:8097`.
- Aktivni odgovor je 200, `audio/flac`, chunked, brez preusmeritve in brez Content-Encoding/Content-Length.
- Dejanska vsebina ima FLAC podpis.
- MA izhod je 48 kHz, 16-bit, stereo FLAC.
- Prvotni ESPHome media pipeline je bil MP3/16 kHz, fizični izhod pa 16 kHz.
- Koda prvotnega firmwarea je vrnila `ESP_ERR_NOT_SUPPORTED`, ker FLAC ni bil v prevedenem naboru formatov.
- Popravljeni firmware vključuje FLAC in uspešno dekodira stereo 48 kHz/16-bit tok.
- Novi PCM5102MK2.0 modul na 5 V daje delujoč stereo linijski izhod; delovanje je uporabnik potrdil pri dejanskem predvajanju iz Music Assistanta.

### Še nepotrjeno

- Ali konkretni modul vsebuje in pravilno podpira PSRAM.
- Dolgoročna optimalnost 32-KiB medpomnilnika za vse vire; pri opravljenih testih povečanje ni bilo potrebno.
- Dolgoročna stabilnost po večurnem predvajanju.


