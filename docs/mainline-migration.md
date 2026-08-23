# Klipper Mainline migration

Questa pagina è mantenuta come punto di compatibilità per i vecchi link.

La procedura di migrazione Sovol SV08 a Klipper Mainline è ora suddivisa nelle guide community seguenti.

## Percorso consigliato

1. [Getting started](getting-started.md)
2. [Backup and rollback](backup-and-rollback.md)
3. [CB1 and Mainline installation](install-cb1-mainline.md)
4. [MCU flashing and recovery](flash-mcus.md)
5. [Base Mainline configuration](base-configuration.md)
6. [Sovol Zero toolhead, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)
7. [Phoenix Macros](phoenix-macros.md)
8. [Validation and calibration](validation-and-calibration.md)
9. [Troubleshooting](troubleshooting.md)

Le pagine dedicate alla vecchia Eddy NG e all'integrazione DKEU restano disponibili come documentazione **storica** del percorso di sviluppo Phoenix, ma non fanno parte della baseline operativa corrente.

## Importante

Non iniziare dal flash delle MCU.

Prima di modificare firmware, eMMC o configurazione, completare almeno:

- inventario della macchina;
- backup delle configurazioni;
- backup/rollback del sistema;
- dump personale delle MCU quando previsto;
- verifica della procedura di recupero.

## Phoenix

La cronologia dettagliata della macchina Phoenix è conservata separatamente in:

`docs/migration-history/phoenix/`

Quella cronologia documenta il percorso reale di sviluppo e debugging, ma non deve essere usata come procedura community lineare.

---

## Navigazione

← **Pagina precedente:** [README](../README.md)
→ **Pagina successiva:** [Getting started](getting-started.md)
