# Phoenix — sensore ambiente camera BME280

Validazione eseguita sulla Sovol SV08 **Phoenix** il 6 settembre 2026.

Questa nota documenta l'installazione realmente validata di un breakout a 4 pin serigrafato `BME/BMP280` (`SDA`, `SCL`, `GND`, `VIN`) usato come sensore ambiente della camera.

> [!IMPORTANT]
> Le indicazioni sotto descrivono la macchina Phoenix realmente verificata. Prima di replicare il cablaggio su un'altra SV08, controllare sempre pinout, tensioni e configurazione della propria scheda.

## Collegamento validato

Il sensore è collegato al connettore **EXP2** della mainboard.

Mappatura finale funzionante:

| Segnale BME280 | Pin MCU | Alias Klipper Phoenix |
|---|---|---|
| SCL | `PC6` | `EXP2_5` |
| SDA | `PC7` | `EXP2_3` |
| GND | GND EXP2 | — |
| VIN | alimentazione EXP2 misurata a circa `3.26 V` | — |

Sulla macchina Phoenix l'alimentazione misurata direttamente ai pin `VIN`/`GND` del breakout è risultata circa **3.26 V**.

Il `printer.cfg` Sovol usa storicamente la label `<5V>` per `EXP2_10`; questa label non deve essere assunta come misura elettrica reale. Sulla Phoenix la tensione è stata verificata con multimetro prima del collegamento del sensore.

## Conflitto con il display stock

Nella configurazione stock la sezione `[display]` usa:

```ini
encoder_pins: ^EXP2_5, ^EXP2_3
```

Questi sono gli stessi due pin riutilizzati dal bus I2C software del BME280. Sulla Phoenix il display UC1701 stock non è più usato, quindi la relativa sezione `[display]` è stata disabilitata prima di assegnare `EXP2_5` e `EXP2_3` al sensore.

La sezione separata `[output_pin beeper]` è rimasta attiva.

## Configurazione Klipper validata

Configurazione finale:

```ini
[temperature_sensor Chamber_Temp]
sensor_type: BME280
i2c_address: 118
i2c_mcu: mcu
i2c_software_scl_pin: EXP2_5
i2c_software_sda_pin: EXP2_3
i2c_speed: 100000
```

Dettagli:

- indirizzo I2C: `118` decimale = `0x76`;
- SCL: `PC6` / `EXP2_5`;
- SDA: `PC7` / `EXP2_3`;
- velocità I2C: `100000` Hz.

## Nota sul debug

Durante l'installazione, l'associazione iniziale SCL/SDA era invertita e Klipper riportava:

```text
MCU 'mcu' I2C request to addr 118 reports error START_NACK
```

Lo stesso errore compariva anche provando l'indirizzo `0x77`. La configurazione ha iniziato a funzionare immediatamente dopo aver corretto l'associazione delle due linee:

```text
SCL = PC6 = EXP2_5
SDA = PC7 = EXP2_3
```

Questa verifica è importante perché un `START_NACK` non implica necessariamente un indirizzo I2C errato: può anche indicare che il dispositivo non sta vedendo correttamente SDA/SCL.

## Validazione runtime

Moonraker ha esposto correttamente l'oggetto:

```text
temperature_sensor Chamber_Temp
```

con lettura iniziale, ad esempio:

```text
temperature: 28.25 °C
```

Dopo alcuni minuti Mainsail ha mostrato contemporaneamente:

- temperatura camera: circa **32–33 °C**;
- pressione: circa **980.7 hPa**;
- umidità relativa: circa **33–36 %**.

Il sensore è risultato visibile sia da **Mainsail in Chrome** sia dall'interfaccia Mainsail integrata in **OrcaSlicer**.

## Stato

**VALIDATO SU HARDWARE REALE — Phoenix, 2026-09-06.**
