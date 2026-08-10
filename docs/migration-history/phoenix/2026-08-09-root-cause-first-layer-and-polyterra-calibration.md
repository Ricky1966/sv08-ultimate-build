# Phoenix — root cause first layer e calibrazione PolyTerra PLA — 2026-08-09

## Contesto

Dopo la migrazione della Sovol SV08 **Phoenix** a Klipper Mainline con Eddy nativo (`[probe_eddy_current eddy]`), homing, probing, QGL e mesh risultavano operativi, ma il primo vero test di stampa su area ampia aveva prodotto un first layer gravemente errato.

La diagnosi è stata condotta sui log reali della stampa e sullo snapshot della configurazione attiva, evitando modifiche meccaniche o correzioni casuali di QGL/Z.

## Root cause individuata

Nel `printer.cfg` attivo era rimasto un override legacy creato per il vecchio Klipper Sovol 0.12 / Eddy NG:

```ini
# Neutralizza _APPLY_EDDY_Z_OFFSET di DKEU: usa printer.probe.last_probe_position,
# campo assente nel Klipper Sovol 0.12. Rimuovere questo blocco se si passa a mainline.
[gcode_macro _APPLY_EDDY_Z_OFFSET]
gcode:
    RESPOND TYPE=COMMAND MSG="Eddy Z offset saltato (Klipper Sovol)"
```

Su Mainline questo override non era più corretto: sostituiva la macro Demon che deve riallineare la coordinata Z dopo `_PROBE_TAP` usando `printer.probe.last_probe_position.z`.

La macro Demon corretta è:

```jinja
[gcode_macro _APPLY_EDDY_Z_OFFSET]
gcode:
  {% set z_pos = printer.toolhead.position.z %}
  SET_KINEMATIC_POSITION Z={z_pos - printer.probe.last_probe_position.z}
  RESPOND TYPE=COMMAND MSG="Setting position to {z_pos - printer.probe.last_probe_position.z}"
```

Nel print fallito `_PROBE_TAP` aveva misurato:

```text
Result: at 152.070,195.220 estimate contact at z=-0.060846
```

ma il vecchio override impediva l'applicazione della correzione. Una differenza di circa `0.061 mm` è sufficiente a compromettere in modo marcato un primo layer da `0.20 mm`.

## Correzione applicata

Il blocco legacy `_APPLY_EDDY_Z_OFFSET` è stato rimosso dal `printer.cfg`, dopo backup del file, senza modificare QGL, meccanica o motori Z.

Dopo restart Klipper è stata verificata l'esecuzione della macro Demon attiva.

Test manuale:

```text
G28
_PROBE_TAP
```

Risultato:

```text
Setting position to 0.457101
Result: at 151.070,199.220 estimate contact at z=0.065696
probe: at 151.070,199.220 bed will contact at z=0.065696
```

Questo conferma che il percorso Mainline + Eddy nativo + Demon ora applica correttamente il riallineamento Z.

## Verifica in stampa

È stato ristampato lo stesso G-code usato nel test precedente (`Cube_PLA_0.2_43m55s.gcode`) per ottenere un confronto A/B significativo.

Esito:

- first layer radicalmente migliorato;
- tre quadranti sostanzialmente uniformi;
- zona posteriore/destra ancora leggermente meno schiacciata rispetto alle altre;
- nessuna evidenza di sottoestrusione generalizzata;
- un piccolo difetto isolato nella zona circa X100/Y40 compatibile con contaminazione locale.

Conclusione: il vecchio override `_APPLY_EDDY_Z_OFFSET` era la causa primaria del fallimento di stampa successivo alla migrazione Mainline.

## Geometria del piatto: stato congelato

Il piatto originale mostra ancora una non uniformità locale, soprattutto nel quadrante posteriore/destra, già osservata anche in test precedenti.

Non vengono effettuate ulteriori correzioni meccaniche, QGL, axis twist o regolazioni dei quattro Z prima dell'arrivo del nuovo piatto in grafite.

Dopo il montaggio del piatto in grafite dovranno essere rivalidati almeno:

- PID bed;
- QGL;
- mesh;
- riferimento Z / first layer;
- comportamento locale del quadrante posteriore/destra.

Se l'anomalia posteriore/destra resterà anche con il nuovo piatto, si aprirà una diagnosi meccanica/kinematica dedicata.

## Nota sulla Benchy e sul primo layer al centro

Una Benchy stampata al centro piatto con il profilo PolyTerra appena calibrato ha mostrato il fondo troppo schiacciato: la scritta inferiore `CT3D.xyz` non è risultata leggibile.

Questo dato non viene corretto in modo permanente sul piatto attuale. Eventuali prove con live Z-offset positivo saranno esclusivamente diagnostiche e non verranno salvate.

## Calibrazione PolyTerra PLA in OrcaSlicer

Profilo macchina:

```text
Phoenix 0.4 nozzle
```

Profilo filamento:

```text
Polymaker PolyTerra PLA @Phoenix 0.4
```

Temperatura usata per la calibrazione: `200 °C`.

### Flow Rate

Pass 1 coarse da flow base `1.00`: campione migliore circa `+15`.

Valore intermedio:

```text
1.15
```

Pass 2 fine: campione migliore `-9`.

Calcolo:

```text
1.15 × 0.91 = 1.0465
```

Valore finale:

```text
Flow ratio = 1.0465
```

### Pressure Advance

Orca PA Pattern, direct drive:

```text
Start: 0
End:   0.08
Step:  0.002
```

Zona migliore osservata: circa `0.034`.

Valore finale:

```text
Pressure Advance = 0.034
```

### Retraction

Test Orca:

```text
Start: 0 mm
End:   2 mm
Step:  0.1 mm
```

Le torri sono risultate sostanzialmente pulite, senza stringing significativo. Non è stato necessario aumentare la retraction.

La retraction corrente viene mantenuta come valore validato.

### Max Volumetric Speed

Test Orca Max flowrate:

```text
Start: 5 mm³/s
End:   30 mm³/s
Step:  0.5 mm³/s
```

La stampa è stata interrotta a 145/163 layer perché la parte alta mostrava già degrado evidente.

Altezza misurata prima del chiaro inizio dei difetti:

```text
45.38 mm
```

Formula Orca:

```text
MVS = start + height × step
    = 5 + 45.38 × 0.5
    = 27.69 mm³/s
```

Il valore di profilo viene mantenuto con margine rispetto al limite visivo:

```text
Max volumetric speed = 24 mm³/s
```

## Profilo PolyTerra PLA consolidato

Valori da considerare attualmente validati:

```text
Flow ratio:            1.0465
Pressure Advance:      0.034
Retraction:            corrente, validata dal test
Max volumetric speed:  24 mm³/s
Nozzle calibration:    200 °C
```

Questi parametri non devono essere ricalibrati da zero al cambio piatto; il nuovo piatto richiederà soprattutto una nuova validazione del sistema Z/bed/first layer.

## Altri punti emersi durante la diagnosi

### Demon `_MESH_HANDLING`

È stato notato un possibile errore logico nella condizione di rilevamento probe di `_MESH_HANDLING`: la condizione usa `or` fra controlli di assenza e quindi può risultare vera anche quando uno dei probe supportati è presente.

Non è stata stabilita come causa del print fallito e **non viene modificata in questa fase**.

### Rilevamento Eddy in Demon start

È stata osservata anche una condizione che considera `probe_eddy_current btt_eddy` ma non esplicitamente `probe_eddy_current eddy`.

Anche questo punto resta documentato ma non modificato finché non esiste evidenza concreta di un effetto negativo.

## Umbilical / USB toolhead

Il fascio nero della toolhead continua a poter sfiorare il telaio posteriore. Il PTFE non è il problema principale.

Il percorso attuale usa filo armonico come supporto, ma il cavo USB sembra avere margine limitato con la geometria originale della catena. In passato è già stato rimosso un elemento della catena per recuperare lunghezza utile.

Sequenza prevista per il futuro redesign:

1. misurare la lunghezza e il margine reale dell'umbilical attuale;
2. valutare se recuperare ulteriori centimetri rimuovendo un altro elemento della catena;
3. ridisegnare prima l'ancoraggio posteriore;
4. solo dopo sistemare l'attacco lato toolhead;
5. verificare il percorso naturale ai quattro estremi XY.

Non irrigidire semplicemente il filo armonico: una forza elastica variabile applicata alla toolhead può introdurre un nuovo disturbo meccanico.

## Stato a fine sessione

Phoenix è nuovamente stampabile e la root cause principale del first layer fallito dopo la migrazione Mainline è stata rimossa.

Restano volutamente congelati fino al nuovo piatto in grafite:

- QGL;
- meccanica Z;
- axis twist;
- correzioni permanenti del live Z-offset;
- interventi sul quadrante posteriore/destra.

Prossimi passi:

1. installare e calibrare il piatto in grafite quando disponibile;
2. se il piatto non è ancora arrivato, calibrare il PETG con release agent/colla sul PEI;
3. riprendere successivamente il redesign dell'umbilical e dell'ancoraggio posteriore.
