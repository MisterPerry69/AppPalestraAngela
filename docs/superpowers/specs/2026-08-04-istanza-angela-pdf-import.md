# ForgE — Istanza Angela: config, estrattore PDF, stile, no-AI

**Data:** 2026-08-04
**Scope:** SOLO la cartella `Angela/` (istanza separata, copia del progetto). Il progetto root NON viene toccato.

## Contesto
`Angela/` è una copia completa di ForgE per un secondo utente, con il suo Sheet + deploy (impostati dall'utente). 4 modifiche:
1. Config isolata (GAS_URL + nome) fuori dal codice.
2. Estrattore PDF → JSON dell'import (Gemini legge il PDF, anteprima/editor, poi importer esistente).
3. Stile: accento rosa `#ce3689`/`#e27fb6`.
4. Fine sessione senza feedback AI: mood + energia + nota personale; storico mostra la nota al posto del riquadro AI.

Gemini resta configurato (serve all'estrattore PDF); sparisce solo il commento AI di sessione.

---

## 1. Config isolata

**`js/config.js`** (NUOVO, caricato per primo in `index.html`):
```js
const FORGE_CONFIG = {
  GAS_URL: "https://script.google.com/macros/s/.../exec", // deploy di QUESTA istanza
};
```
- `api.js`: `const GAS_URL = (typeof FORGE_CONFIG !== "undefined" && FORGE_CONFIG.GAS_URL) || "<fallback>";`
- `index.html`: `<script src="js/config.js"></script>` prima di `api.js`.
- Per una nuova istanza si modifica SOLO `config.js`.

**Nome utente (backend):** `Code.gs getBootstrapData` — `name: "Ale"` → `PropertiesService.getScriptProperties().getProperty("USER_NAME") || "Atleta"`. Nuova funzione setup `setupProfile()` che imposta `USER_NAME`.

---

## 2. Estrattore PDF → JSON

### 2.1 Input (frontend)
- Pannello import (`editor.js _importPanelHtml`): aggiunta sezione **"Carica PDF scheda"** con `<input type="file" accept="application/pdf">`.
- Al select: leggo il file con `FileReader` → base64 (senza prefisso dataURL). Tetto sicurezza: se `file.size > 8MB` → messaggio "PDF troppo grande" e stop.
- Chiamo `apiPost("lift_extract_pdf", { pdfBase64, filename })`.

### 2.2 Estrazione (backend)
**`AI.gs`**: nuova `callAIWithPdf(prompt, pdfBase64)` — come `callAI` ma il payload Gemini include `inline_data` (mimeType `application/pdf`, `data: base64`) accanto al `text`. Stesso fallback catena modelli.

**`Feedback.gs` o nuovo `PdfImport.gs`**: `extractPdfProgram(payload)`:
- Prompt `buildPdfExtractPrompt()`: descrive lo SCHEMA JSON del programma (identico a quello di import: `{nome, dataInizio, settimane, workout:[{nome, esercizi:[{nome,gruppo,attrezzo,nota,rest,superset,tipoEsercizio?,perSettimana:[{serie:[{reps,tipo}]}|{durata,parametri}]}]}]}`) + REGOLE di conversione:
  - Serie miste: `2x6 1xMAX` → 2 serie `{reps:6}` + 1 `{reps:"max",tipo:"max"}`. `TEST 8RM` → `{reps:"8rm",tipo:"test"}`. `3x8 + 1x8 + 20" rest+ MAX` → interpretare come serie multiple + una `max` (best effort).
  - Range `8/12` o `8-12` → `{reps:"8-12"}`.
  - Note lunghe / RPE / "carico sett 1" → campo `nota` dell'esercizio.
  - `settimane` = numero settimane della scheda (Angela=4). Ogni esercizio ha `perSettimana` con ESATTAMENTE `settimane` elementi (se una settimana manca, ripeti l'ultima nota o lascia range base).
  - Cardio/tempo ("Tapis 15 min", "scala 15 min liv X") → `tipoEsercizio:"durata"`.
  - Superset: esercizi marcati "Super Set" consecutivi → stessa etichetta superset.
  - `dataInizio`: se presente nel PDF (formato gg/mm/aaaa) usala in ISO; altrimenti stringa vuota (l'utente la mette in anteprima).
  - Rispondi SOLO col JSON, niente markdown.
- `callAIWithPdf(prompt, pdfBase64)` → `callAIJson`-style parse (strip ```). Se null/malformato → `{status:"ERROR"}`.
- Ritorna `{status:"OK", program: <json parsato>}`.

### 2.3 Anteprima + editor "correggi valori" (frontend)
Nuovo modulo **`js/pdfimport.js`**:
- Ricevuto il JSON, apre una schermata anteprima (riuso `screen-editor` o overlay dedicato) che mostra la scheda tradotta.
- **Editor "correggi valori"** (livello scelto): per ogni workout → esercizi; per ogni esercizio, campi editabili: `nome`, `gruppo`, `attrezzo`, `nota`, `rest`, `superset`; e per ogni settimana la lista serie con reps (testo) + tipo (select work/test/max/warmup/failure) e n° serie (aggiungi/togli riga serie). NIENTE riordino esercizi né creazione superset ex-novo.
- Campi programma editabili: `nome`, `dataInizio` (date picker), `settimane` (mostrato, derivato).
- Validazione al salvataggio: riuso `parseImportProgrammaJson` (già valida perSettimana==settimane, ecc.). Se fallisce, evidenzia l'errore.
- "Salva programma" → `apiPost("lift_import_programma", ...)` (identico al flusso attuale).
- Bottone "Rigenera con AI" (riprova `lift_extract_pdf`) se l'estrazione è da buttare.

### 2.4 Router
`Code.gs`: `case "lift_extract_pdf": return _json(extractPdfProgram(payload));`

---

## 3. Stile girly (accento rosa)

**`css/tokens.css`** (Angela):
```
--accent: #ce3689;
--accent-dim: #a82a6e;
--accent-ghost: rgba(206, 54, 137, 0.13);
--accent-2: #e27fb6;   /* varianti/hover se serve */
--on-accent: #fff2f8;
```
Heatmap stats (rgba rosso 193,39,45) → rosa 206,54,137. Nessun altro cambiamento (blackletter/logo restano). `theme-color` meta in index.html → `#0c0e11` invariato.

---

## 4. Fine sessione senza AI

**`finish.js`**: il flusso 3-step diventa: step 1 mood/energia/nota → salva (niente step "spinner AI" né chiamata `lift_generate_session_feedback`). Il risultato mostra PR/volume/durata + la nota inserita, non il testo AI.

**`Feedback.gs` / `Sessions.gs`**: `saveSession` non cambia (salva già `sessionNotes`). Rimuovo/bypasso la chiamata AI di sessione lato client (non serve toccare il backend `generateSessionFeedback`, semplicemente il client non lo chiama).

**`history.js` `_renderSessionDetail`**: al posto del riquadro `sd-feedback` (aiFeedback) mostro `sessionNotes` (nota personale) con label "Nota". Se vuota, niente riquadro.

**AI.gs**: resta (serve al PDF). Nessuna rimozione.

---

## File toccati (tutti sotto `Angela/`)
- NUOVI: `js/config.js`, `js/pdfimport.js`, (eventuale `backend/PdfImport.gs` o dentro Feedback.gs).
- MOD frontend: `index.html` (script config+pdfimport), `js/api.js` (GAS_URL da config), `js/editor.js` (bottone carica PDF), `js/finish.js` (no AI), `js/history.js` (nota al posto AI), `css/tokens.css` (rosa), `css/stats.css` (heatmap rosa).
- MOD backend: `Code.gs` (USER_NAME + router pdf + setupProfile), `AI.gs` (callAIWithPdf + prompt).

## Criteri di successo
- Carico il PDF di Angela → l'app mostra la scheda tradotta in anteprima editabile → correggo eventuali errori → salvo → il programma compare come gli altri.
- GAS_URL e nome fuori dal codice logico (config.js + property).
- Accento rosa ovunque era rosso.
- Fine sessione: solo mood/energia/nota; storico mostra la nota, niente AI.

## Rischi
- **Qualità estrazione AI**: schede complesse (Angela ha `3x8+1x8+20"rest+MAX`) → l'AI approssima. Mitigato dall'editor di correzione (obbligatorio) + rigenera.
- **Payload PDF**: base64 di 166KB ≈ 220KB testo, ok per Apps Script. Tetto 8MB come guard-rail.
- **Gemini PDF**: verificare che i modelli in catena (`gemini-2.5-flash-lite/flash/2.0-flash`) accettino inline_data PDF; flash lite potrebbe non gestire bene i PDF → mettere flash come primo per il PDF.
