# Minerva-microwake-word

microWakeWord ébresztőszó Timi asszisztensének (Minerva, Gemini) a dev lapon (Waveshare ESP32-S3-RLCD-4.2).
Tanítás Google Colabban, magyar Piper-hangokkal (hu_HU, medium).

| Fájl | Mi ez |
|---|---|
| `custom_microwakeword_training.ipynb` | az első „Minerva” modell notebookja (egy szó, cutoff 0,85) |
| `minerva.tflite` | a régi „Minerva” modell – **túl érzékeny volt**, a dev lapon kikapcsolva (`select.dev_lap_ebreszto_szo_2` = no_wake_word) |
| `he_minerva_training.ipynb` | **új: „Hé Minerva”** (2026-11-21) – Colab-változat |
| `he_minerva.tflite`, `he_minerva.json` | **a kész „Hé Minerva” modell** (2026-11-21, GitHub Actions: dronesys/devagent-server `wakeword` workflow, kiadás `wakeword-he-minerva-10`, 20 000 lépés, angol TTS kiejtés-közelítéssel) |
| `teszt/he_minerva_roc.txt` | tesztkimenet: téves elutasítás (frr) és téves ébredés/óra (faph) küszöbönként |

## Hé Minerva – mi változott a régihez képest
- kétszavas kifejezés, `CLIP_DURATION_MS` 1800;
- **kemény negatívok** (Cell 8b): „minerva” egyedül, „hé”, „hé mi van”, „minden”, „hey jarvis” stb. – ezekre nem ébredhet;
- `NEGATIVE_CLASS_WEIGHT` 25 (régi 20), `TRAINING_STEPS` 20000 (régi 10000), `target_minimization` 0,5 téves ébredés/óra (régi 0,9);
- `PROBABILITY_CUTOFF` 0,97 (régi 0,85); a végén `he_minerva.tflite` + `he_minerva.json` (ESPHome manifest) töltődik le.

## Futtatás
1. Megnyitás Colabban (a notebook tetején lévő gomb), Futtatási környezet → **T4 GPU**.
2. Cellák sorban; Cell 2 után session-újraindítás kell (a notebook kiírja). Cell 5-nél meghallgatod a változatokat,
   Cell 5b-ben beírod, melyek maradjanak (pl. `0,1,2`).
3. Cell 8 után **Cell 8b** (kemény negatívok), majd tovább a tanításig. A végén letöltődik a két fájl.
4. A két fájl ide a tárolóba (`he_minerva.tflite`, `he_minerva.json`), utána a dev lapon csere.

## Beépítés a dev lapra
A dev lap ESPHome YAML-ja a HA-n van (`/config/esphome/`, Jarvis is innen éri el), nem a ha-config tárolóban.
A régi Minerva modell helyett:

```yaml
micro_wake_word:
  models:
    - model: https://github.com/dronesys/Minerva-microwake-word/raw/main/he_minerva.json   # vagy helyi: wakeword/he_minerva.json
      id: he_minerva
```
Privát tárolónál az URL-t az ESPHome nem éri el → helyi másolat a `/config/esphome/wakeword/` alá.
Fordítás + OTA, majd HA: `select.dev_lap_ebreszto_szo_2` = „Hé Minerva”. Entitást nem nevezünk át.
Ha még mindig érzékeny: `probability_cutoff` 0,98–0,99, `sliding_window_size` 8–10.

## Tesztkimenet (wakeword-he-minerva-10)
AUC 0,704. Küszöb 0,90: frr 22,9 %, faph 0,75/óra; 0,85: frr 20,7 %, faph 0,94/óra. A manifestben 0,97 (szigorúbb, kevesebb téves ébredés,
több kihagyott kimondás). Ha sokszor nem ismeri fel: 0,93–0,95; ha még mindig ok nélkül ébred: 0,98–0,99 vagy sliding_window_size 8.
Jobb modellhez: saját felvételek (`wakeword/sajat/` a devagent-serverben) és újrafuttatás.
