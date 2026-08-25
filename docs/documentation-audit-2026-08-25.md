# Audit documentazione Phoenix — 25 agosto 2026

Questa pagina è un checkpoint di revisione, non una guida operativa. Serve a distinguere la documentazione che appartiene alla baseline corrente da quella che conserva soltanto una fotografia storica. Nessun file candidato viene cancellato o spostato in questa revisione: i casi dubbi restano esplicitamente da decidere.

## Criterio

Classificazioni usate:

- **KEEP — baseline**: guida necessaria o riferimento corrente;
- **KEEP — history**: contenuto utile, ma deve essere letto come cronologia e non come configurazione corrente;
- **REVIEW / MOVE**: contenuto valido come archivio tecnico, ma probabilmente collocato meglio sotto `docs/migration-history/phoenix/`;
- **UPDATE**: file necessario, ma con parti superate dalla baseline corrente.

## KEEP — baseline

### `docs/getting-started.md`

Resta la porta di ingresso della migrazione. È coerente con la separazione tra baseline community e personalizzazioni Phoenix.

### `docs/backup-and-rollback.md`

Resta essenziale. Il progetto continua a richiedere backup verificati e possibilità di rollback prima di modifiche invasive.

### `docs/install-cb1-mainline.md`

Resta essenziale per il percorso CB1/Mainline.

### `docs/flash-mcus.md`

Resta essenziale per flashing e recovery.

### `docs/base-configuration.md`

Resta parte della baseline Mainline.

### `docs/native-eddy.md`

Resta il riferimento generale per Eddy nativo. I valori Phoenix specifici devono restare separati dalla baseline riutilizzabile.

### `docs/zero-toolhead-eddy-2026-08-17.md`

Resta necessario come documento specifico della Sovol Zero/Eddy usata sulla Phoenix, purché continui a essere presentato come configurazione macchina-specifica e non come preset universale.

### `docs/phoenix-macros.md`

Resta il riferimento corrente per il layer Phoenix Macros.

Il nuovo Phoenix Automatic Soak è già descritto qui. La pagina, però, necessita di un ultimo passaggio di consolidamento dopo la chiusura dei due punti ancora aperti del soak: compatibilità del credito con un `PHOENIX_START BED=...` diverso dalla temperatura persistente e semantica di `soak_total_seconds` quando `SOAK_SECONDS` differisce da 600.

### `docs/validation-and-calibration.md`

Resta necessaria come guida di metodo. Va mantenuta la distinzione fra baseline corrente e sezioni esplicitamente storiche.

### `docs/troubleshooting.md`

Resta necessaria. È coerente con l'approccio evidence-first adottato durante la migrazione.

### `docs/remote-access-tailscale.md`

Nuova guida validata il 25 agosto 2026. Da mantenere.

Stato validato:

- Tailscale installato sul CB1 come nodo singolo;
- nessun subnet routing;
- nessun exit node;
- `--accept-dns=false` per non modificare la gestione DNS locale;
- SSH via Tailscale funzionante;
- Mainsail/Moonraker via Tailscale funzionanti dopo aggiunta di `100.64.0.0/10` ai `trusted_clients` Moonraker.

Resta da completare il test esplicito con Phoenix e client su due reti Internet differenti.

## KEEP — history / REVIEW

### `docs/demon-integration.md`

Il file è già marcato chiaramente come fase storica e dichiara che DKEU non è più una dipendenza runtime Phoenix. Il contenuto conserva valore tecnico e di attribuzione.

**Da decidere domani:**

1. lasciarlo sotto `docs/` come appendice storica facilmente raggiungibile; oppure
2. spostarlo sotto `docs/migration-history/phoenix/` per rendere ancora più difficile confonderlo con la baseline corrente.

Raccomandazione preliminare: **MOVE**, mantenendo un piccolo redirect/indice se esistono link pubblici al percorso attuale.

## REVIEW / MOVE — snapshot datati oggi ancora in `docs/`

### `docs/phoenix-print-recovery-2026-08-18.md`

È una fotografia preziosa della transizione Phoenix, ma contiene configurazioni esplicitamente superate: dipendenze DKEU allora ancora presenti, `CLEAN_NOZZLE` Demon, vecchio stato di `PHOENIX_START`, decisioni temporanee su mesh e hook Orca.

Non deve essere interpretato come guida corrente.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`**.

### `docs/phoenix-status-2026-08-21.md`

È uno snapshot macchina-specifico e include patch/configurazioni che descrivono precisamente lo stato del 21 agosto, comprese dipendenze e wrapper poi rimossi.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`**.

### `docs/phoenix-status-2026-08-22.md`

È uno snapshot importante per la nascita del thermal credit, ma la logica Automatic Soak del 25 agosto lo ha già superato in aspetti fondamentali: persistenza su disco, timestamp wall-clock, recovery dopo restart, guard di 60 s e credito accumulabile oltre 600 s.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`**.

### `docs/phoenix-macros-validation-2026-08-22.md`

È utile come prova della validazione fisica progressiva, ma registra conteggi e stati intermedi (`22` poi `21` macro) precedenti all'aggiunta delle nuove macro Automatic Soak.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`** e lasciare in `docs/phoenix-macros.md` soltanto lo stato consolidato corrente.

### `docs/phoenix-session-closeout-2026-08-22.md`

È per definizione un closeout di sessione. Contiene pending successivamente chiusi e uno snapshot del runtime poi evoluto.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`**.

### `docs/phoenix-input-shaper-validation-2026-08-22.md`

La misura resta utile e contiene anche il promemoria corretto di ripetere Input Shaper dopo il ripristino dei pannelli isolati. Tuttavia è una validazione macchina-specifica datata, non una guida community.

Raccomandazione preliminare: **MOVE in `docs/migration-history/phoenix/`**, mantenendo nella guida di calibrazione soltanto il principio della ricalibrazione dopo variazioni meccaniche significative.

## UPDATE — incongruenze da correggere senza cancellare storia

### README italiano e inglese

Il README pubblicato descrive ancora la baseline del 23 agosto con un conteggio fisso di `21` macro `PHOENIX_*` e afferma che nel layer Phoenix non viene usato `save_variables`.

Dopo l'introduzione di Phoenix Automatic Soak il conteggio fisso non rappresenta più bene la baseline, mentre `save_variables` viene ora usato intenzionalmente per lo stato termico persistente in `phoenix_variables.cfg`.

Raccomandazione: sostituire il conteggio rigido con una descrizione funzionale del set corrente e precisare che `save_variables` è usato esclusivamente dal sottosistema Phoenix persistente, non come residuo DKEU.

Il README deve inoltre aggiungere un link esplicito alla nuova guida `docs/remote-access-tailscale.md` senza trasformare Tailscale in requisito della migrazione: è una funzione opzionale di accesso remoto.

### `docs/phoenix-macros.md`

La pagina è corretta nell'impianto, ma alcune frasi parlano ancora di Automatic Soak "in validazione". Il recovery persistente, il timestamp wall-clock, il decadimento offline, il guard da 60 s e il ritorno autonomo a `VALID` sono stati realmente verificati il 25 agosto.

Resta invece corretto mantenere come aperti i due casi tecnici indicati sopra; non vanno dichiarati chiusi finché non vengono testati.

## Struttura consigliata del repository documentale

Obiettivo della prossima pulizia:

```text
docs/
  getting-started.md
  backup-and-rollback.md
  install-cb1-mainline.md
  flash-mcus.md
  base-configuration.md
  native-eddy.md
  zero-toolhead-eddy-2026-08-17.md
  phoenix-macros.md
  validation-and-calibration.md
  troubleshooting.md
  remote-access-tailscale.md
  migration-history/
    phoenix/
      ...snapshot e sessioni datate...
```

La regola proposta è semplice: **le guide in `docs/` devono dire cosa fare oggi; gli snapshot devono spiegare cosa è successo allora**.

## Decisioni rinviate

Nessuno spostamento o cancellazione viene eseguito in questo audit.

Domani decidere:

1. se spostare `demon-integration.md` nella cronologia;
2. quali snapshot datati spostare in blocco sotto `docs/migration-history/phoenix/`;
3. se mantenere redirect/compatibility stub per i vecchi percorsi pubblici;
4. aggiornare README IT/EN per Automatic Soak e Tailscale;
5. consolidare `phoenix-macros.md` dopo i due ultimi test di coerenza termica.
