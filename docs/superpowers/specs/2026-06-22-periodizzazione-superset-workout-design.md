# ForgE — Periodizzazione, superset, DB esercizi & workout

**Data:** 2026-06-22
**Tipo:** Evoluzione tecnica dell'app (modello dati + UI), basata su feedback d'uso reale e su una scheda periodizzata reale (coach Carlo Brodesco, 6 settimane).
**Scope:** Modello dati esercizi/schede, importer JSON, editor scheda, esecuzione workout. Backend Apps Script + frontend.

---

## 0. Contesto e problemi da risolvere

L'utente ha provato l'app con una scheda reale e ha trovato 6 problemi:

1. **DB esercizi** in inglese (873 voci, rumore) → vuole un DB **curato in italiano**, costruito sulle proprie schede, su un **foglio Google Sheets dedicato** (lega storico/pesi/reps/record per esercizio).
2. **Compilazione scheda (editor)**: inserimenti da tastiera scomodi → vuole preset/tasti/shortcut come già fatto nel workout.
3. **Rep/serie custom** da rivedere.
4. **Superset** non gestiti (la sua prima scheda ne ha).
5. **Serie di avvicinamento (warm-up)** non disponibili nel workout.
6. **Layout workout** da allargare (sfruttare sopra/sotto).

In più, la scheda reale è **periodizzata su 6 settimane**: stessi esercizi, set/reps/rest che variano per settimana, con serie miste (`2x7 2x6`), test (`test 8rm`), reps range (`8-12`), e serie "max". La scheda contiene 4 workout (A/B/D di pesi; C cardio/tempo).

---

## 1. Modello dati

### 1.1 Foglio esercizi (`exercises`, nuovo tab Sheets)

Sostituisce il DB inglese hardcoded (`js/exercises-db.js`, 873 voci → eliminato). Curato, in italiano, coi nomi del coach.

Colonne:
```
id            slug stabile (es. "spinte_manubri_panca_30")
nome          "Spinte manubri panca 30°"
gruppo        "Petto" | "Schiena" | "Spalle" | "Bicipiti" | "Tricipiti" | "Femorali" | "Quadricipiti" | ...
attrezzo      "manubri" | "bilanciere" | "cavi" | "macchina" | "smith" | "corpo libero" | ...
caricoConteggio   "singolo" | "doppio"   // doppio = peso per-attrezzo ×2 nel volume
unilaterale       "sì" | "no"             // sì = una riga vale dx+sx
noteDefault       testo opzionale (nota tecnica di default per l'esercizio)
```

**Regole derivate:**
- `caricoConteggio` default dedotto dall'attrezzo: `manubri` → `doppio`, tutto il resto → `singolo`. Sovrascrivibile a mano.
- `unilaterale` default `no`, impostato a mano dove serve (affondi 1 gamba, high row singolo, rowing singolo).
- Lo storico (`sets`), i record (`prs`) e i suggerimenti si legano a `exerciseRef` che ora punta a un id del foglio `exercises` (namespace dedicato, es. `ex:spinte_manubri_panca_30`).

### 1.1-bis Programma (aggiunta 2026-06-22 — avanzamento settimana)

**Evoluzione del modello:** la "settimana corrente" non vive sul singolo workout ma su un **programma** che li raggruppa. Un programma = un ciclo (es. "Brodesco 6 settimane") con più workout (A/B/C/D) che condividono la stessa progressione temporale.

```jsonc
{
  "id": "prog_xxx",
  "nome": "Brodesco 6 settimane",
  "dataInizio": "2026-06-22",     // ISO date; base per il calcolo settimana
  "weeks": 6,
  "weekOverride": null,           // se valorizzato (int), forza currentWeek ignorando la data
  "workouts": [ /* schede periodizzate, vedi §1.2 */ ]
}
```

**Settimana corrente (condivisa, calcolata per data):**
```
currentWeek = weekOverride != null
  ? clamp(weekOverride, 1, weeks)
  : clamp( floor( (oggi - dataInizio) / 7giorni ) + 1, 1, weeks )
```
- I **salti** di allenamento non sfasano la settimana (segue il calendario, come ragiona il coach).
- **Override manuale**: l'utente può forzare la settimana; impostare `weekOverride` (oppure, equivalente, spostare `dataInizio`). Scelta implementativa: `weekOverride` esplicito, più semplice da ragionare.
- Oltre `weeks` ⇒ programma **completato**.

**Storage backend:** nuovo tab `programs` (`id, nome, dataInizio, weeks, weekOverride, archived, createdAt, workoutsJSON`) — oppure, se più semplice, i workout restano in `templates` con un campo `programId` e il programma porta i metadati. Decisione in implementazione (default: tab `programs` con i workout serializzati in `workoutsJSON`, coerente con il pattern `structureJSON`).

**Migrazione concettuale:** "scheda = workout" (§1.2) resta valido come *unità interna*; il programma è il contenitore che porta `weeks`+settimana condivisa. Il singolo workout NON porta più `currentWeek` (lo eredita dal programma a runtime).

### 1.2 Workout periodizzato (unità interna del programma)

Un **workout** (es. "Workout A") è un'unità dentro un programma. La scheda PDF reale = 1 programma con 3 workout (A, B, D; C fuori scope).

Si estende `templates.structureJSON` (campo già esistente) con:
```jsonc
{
  "weeks": 6,
  "currentWeek": 1,            // settimana attuale (1-based); avanza al completamento
  "blocks": [
    {
      "exerciseRef": "ex:spinte_manubri_panca_30",
      "note": "mantieni il carico del test, segui la progressione",
      "rest": "2'30\"",
      "supersetGroup": null,        // string (es. "A") = stesso superset; null = singolo
      "perWeek": [                   // ESATTAMENTE `weeks` elementi
        { "sets": [ {"reps":"8rm","type":"test"}, {"reps":6,"type":"work"}, {"reps":6,"type":"work"} ] },
        { "sets": [ {"reps":6,"type":"work"}, {"reps":6,"type":"work"}, {"reps":6,"type":"work"} ] }
        // ... fino a weeks
      ]
    }
  ]
}
```

### 1.3 Serie (set)

```jsonc
{ "reps": <numero> | "<min>-<max>" | "<n>rm" | "max", "type": "work" | "warmup" | "test" | "max" | "failure" }
```
- **warmup illimitate**: si possono inserire quante serie warm-up si vuole, mescolate alle altre.
- Serie miste tipo `2x7 2x6` → quattro righe: due `{reps:7}` + due `{reps:6}`.
- Reps range (`8-12`, `7-10`) → stringa nel campo `reps`.

### 1.4 Volume, PR, 1RM, manubri, unilaterale

- **Volume di una serie** = `peso × reps × (caricoConteggio == "doppio" ? 2 : 1)`.
  Es. panca manubri 40 kg/manubrio × 8 reps = `40 × 8 × 2 = 640 kg`.
- **Warm-up**: **incluse** nel volume (è carico spostato), **escluse** da PR e 1RM stimato.
- **PR / record**: mostra il peso **inserito** dall'utente (per-manubrio, es. "42 kg/manubrio"), NON il valore raddoppiato.
- **1RM stimato**: calcolato solo su serie di lavoro (work/test/max/failure), peso inserito.
- **Unilaterale (`sì`)**: nel workout una riga-serie vale per entrambi i lati (etichetta "per lato"); l'utente inserisce i dati una volta sola. Volume conta una volta (il ×2 dei manubri è cosa diversa e indipendente).

---

## 2. Importer JSON & Editor

### 2.1 Importer

**Aggiornamento 2026-06-22:** il JSON di import descrive un **intero programma** (più workout + data inizio + settimane condivise), non più un singolo workout. Un solo incolla = programma completo.

- In creazione nuovo programma, accanto a "da zero": **"Incolla JSON"**.
- **Formato JSON di import — PROGRAMMA** (umano-leggibile, italiano):
```jsonc
{
  "nome": "Brodesco 6 settimane",
  "dataInizio": "2026-06-22",
  "settimane": 6,
  "workout": [
    {
      "nome": "Workout A",
      "esercizi": [ /* come sotto */ ]
    },
    { "nome": "Workout B", "esercizi": [ ... ] },
    { "nome": "Workout D", "esercizi": [ ... ] }
  ]
}
```
  - `settimane` è del programma; ogni esercizio ha `perSettimana` con esattamente `settimane` elementi.
  - Ogni workout in `workout[]` ha la stessa forma esercizi di prima:
```jsonc
{
  "nome": "Workout A",
  "esercizi": [
    {
      "nome": "Spinte manubri panca 30°",
      "gruppo": "Petto",
      "attrezzo": "manubri",
      "nota": "mantieni il carico del test, segui la progressione",
      "rest": "2'30\"",
      "superset": null,                 // o un'etichetta es. "A"
      "perSettimana": [
        { "serie": [ {"reps":"8rm","tipo":"test"}, {"reps":6}, {"reps":6} ] },
        { "serie": [ {"reps":6}, {"reps":6}, {"reps":6} ] }
        // ... fino a "settimane"
      ]
    }
  ]
}
```
  - `tipo` omesso ⇒ `work`. `superset` con stessa etichetta ⇒ stesso gruppo.
- **Esercizi non presenti** nel foglio `exercises`: **creati in automatico** con default (gruppo/attrezzo dal JSON, `caricoConteggio` dedotto dall'attrezzo, `unilaterale` = no). Aggiustabili dopo.
- **Validazione**: JSON malformato o `perSettimana.length != settimane` ⇒ messaggio d'errore chiaro, nessuna scheda parziale creata.
- **Dato di partenza**: vengono forniti i JSON pronti di Workout A/B/D estratti dal PDF reale.

### 2.2 Editor manuale

- Resta per ritocchi. Migliorato con **input rapidi** (stepper +/−, preset/chip) — niente tastiera obbligatoria, riusando il pattern già presente nel workout (`openNum` in `exec.js`).
- Supporta: aggiunta/rimozione serie per-settimana, tipi di serie (work/warmup/test/max/failure), reps range, superset (raggruppamento blocchi), rest per esercizio.

---

## 3. Workout (esecuzione)

La vista **focus** (una serie alla volta, grande) è una scelta deliberata dell'utente — **si mantiene**, non si torna alla lista completa (già provata e scomoda). Si migliora così:

### 3.1 Layout allargato (problema #6)
- **Sopra**: esercizio corrente + **nota del coach sempre visibile** (es. "discesa 3″ fermo 1″"); progress "Esercizio 2/8 · Settimana 3"; stato superset ("Superset A: Curl + French press").
- **Centro**: serie corrente in focus, peso/reps grandi, suggerimenti (oggi / serie precedente / ultima volta).
- **Sotto**: azione primaria (registra serie / avvia recupero) + timer di recupero (rest per esercizio, già nel modello).

### 3.2 Warm-up al volo (problema #5)
- Bottone **"+ avvicinamento"** sull'esercizio corrente: aggiunge una serie `type: warmup` al volo, marcata visivamente, con peso suggerito più basso. Quante se ne vuole.

### 3.3 Input rapidi (rifinitura #2)
- Stepper +/− e chip preset (già esistenti via `openNum`) restano; tastiera disponibile ma mai obbligatoria.

### 3.4 Panoramica + salto rapido
- **Bottone laterale** apre una **tendina/overlay** con tutti gli esercizi del giorno e le rispettive serie (fatte/da fare), **evidenziando dove sei**.
- È **interattiva**: toccando un esercizio/serie ci si salta (per rifare o anticipare).

---

## 4. Backend (Apps Script)

Il backend già regge gran parte del modello:
- `templates.structureJSON` ospita la periodizzazione (nessun cambio di schema foglio).
- `sets` ha già `setType`, `blockType`, `weight`, `reps`, `est1RM`, `isPR` → base per warm-up/superset/serie miste.
- `prs` lega i record a `exerciseRef`.

Modifiche backend:
- **Nuovo tab `exercises`** (header §1.1) + funzioni read/seed (riuso pattern `_readTab`/`_appendRow`, registrazione in `TAB`/`HEADERS` in `Code.gs`).
- API per: leggere catalogo esercizi, creare esercizio (auto-create da import), leggere/scrivere schede periodizzate.
- Logica volume/PR/1RM aggiornata per: warm-up esclusi da PR/1RM, `caricoConteggio` ×2 nel volume, unilaterale.
- Migrazione: i template esistenti (mock/eventuali reali) restano compatibili (struttura senza `weeks` trattata come scheda a 1 settimana).

---

## 5. Ordine di lavoro

1. **Modello dati + foglio `exercises`** (backend: tab, header, seed, API catalogo; frontend: rimozione DB inglese, nuovo accesso esercizi).
2. **Importer JSON** (formato, parser, validazione, auto-create esercizi) + **JSON pronti** di Workout A/B/D dal PDF.
3. **Editor scheda** (periodizzazione, serie/tipi, superset, rest, input rapidi).
4. **Workout** (layout allargato, warm-up al volo, panoramica con salto rapido).

Ogni fase poggia su basi corrette dalla precedente.

---

## 6. Fuori scope (per ora)

- **Workout C** (cardio/tempo: tapis/scala a durata, velocità, pendenza, "max tutte"). Da affrontare dopo con un tipo esercizio "a durata".
- **Service worker / offline**.
- Traduzione del vecchio DB inglese (eliminato, non tradotto).

---

## 7. Criteri di successo

- Inserisco la mia scheda **una volta** (via JSON) e l'app sa a che settimana sono, mostrando set/reps/rest corretti senza reinserimenti.
- Superset visibili e gestiti in editor e workout.
- Posso aggiungere serie di avvicinamento (quante voglio), che contano nel volume ma non nei PR.
- Peso manubri inserito per-manubrio; volume raddoppiato; PR mostra il peso per-manubrio.
- Esercizi unilaterali: una riga per entrambi i lati.
- Workout in vista focus allargata, nota coach sempre visibile, panoramica del giorno con salto rapido.
- DB esercizi curato in italiano su foglio Sheets; storico/PR coerenti per esercizio.

---

## 8. Rischi & mitigazioni

- **Compatibilità schede esistenti**: struttura senza `weeks` → trattata come 1 settimana (no crash).
- **Parsing JSON utente**: validazione robusta + messaggi chiari; mai creare schede parziali.
- **est1RM/PR con manubri e warm-up**: test mirati sulla logica volume/PR (doppio vs singolo, warm-up esclusi).
- **Migrazione exerciseRef**: il vecchio formato `public:<id>` non esiste più nel nuovo catalogo → import/seed deve mappare ai nuovi id `ex:<slug>`; schede vecchie con ref orfani segnalate, non silenziosamente rotte.
