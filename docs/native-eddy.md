# Native Eddy — nota storica e percorso Phoenix corrente

Ultima revisione strutturale: **2026-08-23**.

## Stato di questa pagina

Questa pagina viene mantenuta per conservare validi i vecchi link e spiegare l'evoluzione del sistema Eddy sulla Sovol SV08 “Phoenix”.

La configurazione Phoenix attuale **non utilizza più il precedente hardware Sovol Eddy NG collegato alla toolboard originale**.

La baseline corrente utilizza:

- **Sovol Zero Extruder Kit**;
- toolhead collegata via **CAN**;
- probe Eddy integrato nella Zero;
- supporto Eddy nativo di Klipper Mainline;
- Phoenix Macros.

La guida operativa corrente è:

[Sovol Zero toolhead, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)

## Percorso storico

Durante una fase precedente della migrazione Phoenix venne utilizzato il probe Sovol Eddy NG / LDC1612 sulla toolboard originale.

Quella configurazione comprendeva:

- MCU toolboard `extra_mcu`;
- collegamento I2C sulla vecchia toolboard;
- utilizzo iniziale del plugin `probe_eddy_ng.py`;
- successiva migrazione a `[probe_eddy_current eddy]`;
- calibrazioni e workaround specifici di quella configurazione;
- integrazione con DKEU.

Questi dati restano utili per ricostruire il debugging e le decisioni che hanno portato alla configurazione attuale, ma **non devono essere utilizzati come preset per la Sovol Zero**.

La cronologia tecnica è conservata sotto:

[migration-history/phoenix](migration-history/phoenix/)

## Cosa non copiare dalla vecchia configurazione

Non trasferire automaticamente sulla Sovol Zero valori appartenenti alla precedente Eddy NG, compresi:

- nomi MCU della vecchia toolboard;
- pin I2C;
- offset X/Y;
- `max_sensor_hz`;
- `reg_drive_current`;
- curve di calibrazione;
- workaround per `probe_eddy_ng.py`;
- macro DKEU create per il vecchio percorso.

La Zero utilizza una configurazione hardware e una calibrazione proprie.

## Percorso operativo corrente

Per la Phoenix attuale:

1. verificare mainboard e sistema Mainline;
2. configurare la Sovol Zero e il bus CAN;
3. configurare il probe Eddy integrato;
4. calibrare Eddy sulla propria macchina;
5. verificare homing;
6. verificare QGL;
7. verificare rapid bed mesh;
8. validare Z e first layer;
9. integrare le Phoenix Macros.

I parametri e i risultati realmente validati sulla Zero sono riportati nella pagina dedicata.

---

## Navigazione

← **Pagina precedente:** [Configurazione base Mainline](base-configuration.md)
→ **Pagina successiva:** [Sovol Zero, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)
