# ForgE — Fix navigazione sessione (chiusura manuale, salti, superset)

**Data:** 2026-07-08
**Contesto:** bug emersi usando la scheda B con macchinari occupati (salto esercizi, ripresa, superset sbilanciato).

## Problemi

1. **Chiusura automatica** all'ultimo blocco → va al feedback senza volerlo.
2. **Esercizi saltati "in avanti"** (salto dalla lista a un altro esercizio) non riproposti.
3. **Ripresa incasinata**: rifà male il calcolo di cosa manca; marca "fatto" cose non fatte; sessione resta incompleta.
4. **Superset sbilanciato** (2/4, 3/2) anche a giro regolare → causa: `showUndo` (annulla serie) muove `si/bi` in modo lineare ignorando il superset.

## Comportamento deciso

### A. Nessuna chiusura automatica
- La sessione va al feedback **solo** premendo TERMINA (`confirmEnd`), mai da `advance`.
- Finito l'ultimo blocco, se restano esercizi **non completati**, l'app **salta automaticamente al primo esercizio non completato** (ordine scheda). Se sono tutti completati, resta sull'ultimo (non chiude).

### B. Definizione "esercizio completato"
- Un blocco è **completato** quando ha registrato (done non-skipped) un numero di serie ≥ alle serie di lavoro previste per la settimana corrente. (I set `skipped` NON contano come completamento; contano come "toccato" ma non "fatto".)
- Helper: `isBlockComplete(bi)`.

### C. Salti e "prossimo"
- Dopo aver completato l'ultima serie di un blocco (fine giro superset incluso), `advance` NON fa `bi++` cieco: chiama `goToNextIncomplete(fromBi)` che trova il **primo blocco non completato** partendo dal successivo, e se non ce n'è **a valle**, cerca **dall'inizio** (per recuperare i saltati). Se non ce n'è nessuno, resta sull'ultimo completato (nessuna chiusura).
- `jumpToBlock` resta per il salto manuale dalla lista.

### D. Entrare in un esercizio già completato
- Se salto (da lista) su un blocco **già completato**: la vista mostra l'esercizio ma con lo stato "completato" e un bottone esplicito **"+ Aggiungi serie"**. NON pre-carica una serie vuota; un tap per errore non crea done.
- Solo premendo "+ Aggiungi serie" si aggiunge una serie extra a quel blocco (via `_extraSets`, come le warm-up al volo) e si entra in inserimento.

### E. Fix annulla (undo) nel superset
- `showUndo` deve tornare indietro **coerentemente**: rimuove l'ultimo done, e riposiziona `bi/si` esattamente sul blocco/serie di quel done (usa `d.bi`/`d.si` del done rimosso), invece di `si--` lineare.

### F. Ripresa (parziale) coerente
- La ripresa resta: `resumedDoneBlocks` marca i blocchi completati in sessioni precedenti della stessa settimana. All'avvio si parte dal primo blocco non completato considerando ANCHE `resumedDoneBlocks`.
- `isBlockComplete` considera completati anche i blocchi in `resumedDoneBlocks`.

## File toccati
- `js/exec.js`: `advance` (no auto-finish + goToNextIncomplete), nuovo `isBlockComplete`/`goToNextIncomplete`, `renderExec` (stato "completato" + bottone aggiungi serie), `showUndo` (fix superset), `jumpToBlock`.
- `css/exec.css`: stile stato completato + bottone aggiungi serie.
- Nessun cambio backend (il conteggio serie è già per-done; il fix è tutto lato navigazione client).

## Verifica
- Superset 3+3 a giro pulito → 3/3 (già ok).
- Superset con undo a metà → conteggio resta corretto.
- Salto avanti (occupato) → finito l'esercizio fuori posto, torno al primo non completato.
- Blocco già completato → nessuna serie vuota su tap; "+ Aggiungi serie" funziona.
- Ultimo blocco con saltati → NON chiude, va al saltato. Con tutto fatto → resta, TERMINA manuale.
