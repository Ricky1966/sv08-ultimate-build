# OrcaSlicer presets — Phoenix

Snapshot dei preset personali OrcaSlicer della Sovol SV08 "Phoenix".

OrcaSlicer:
- installazione Flatpak user
- versione al momento dello snapshot: 2.4.2

## Machine

- Phoenix 0.2 nozzle
- Phoenix 0.4 nozzle
- Phoenix 0.6 nozzle
- Phoenix 0.8 nozzle

I profili ereditano dai corrispondenti preset di sistema Sovol SV08.

Il campo `print_host` è volutamente escluso dallo snapshot perché dipende
dalla rete locale e non fa parte della configurazione macchina versionata.

## Process

- Phoenix 0.20 Strong 30% Gyroid + Support

Base:
- 0.20mm Standard @Sovol SV08 0.4 nozzle

## Filament

### Operativo

- Polymaker PolyTerra PLA @Phoenix 0.4
  - base: PolyTerra PLA @System
  - flow ratio: 1.00
  - pressure advance: 0.032
  - nozzle: 210 °C
  - bed: 65 °C
  - first layer bed: 70 °C
  - max volumetric speed: ereditato dalla base Orca (22 mm³/s)
    fino a nuova calibrazione sulla Phoenix

### Da calibrare

- Polymaker PolyLite PETG @Phoenix 0.4 - TO CALIBRATE
  - base: PolyLite PETG @System

- Polymaker PolyLite ASA @Phoenix 0.4 - TO CALIBRATE
  - base: PolyLite ASA @System

PETG e ASA non contengono ancora override di calibrazione Phoenix.

## Hardware rilevante

Configurazione attuale di riferimento:

- Sovol SV08 Phoenix
- Klipper Mainline
- Micro Swiss FlowTech hotend
- FlowTech Brass Plated nozzle 0.4 mm presumibilmente standard
- Eddy NG

Il Max Volumetric Speed del PLA dovrà essere ricalibrato sulla configurazione
FlowTech reale prima di aumentarlo rispetto al valore prudente ereditato.
