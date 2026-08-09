# ForgE — Re-skin medievale/engraved

**Data:** 2026-06-21
**Tipo:** Re-skin completo dell'app esistente "Lift" → "ForgE"
**Scope:** Solo presentazione (CSS, HTML visibile, asset logo). Nessun cambio di logica, dati o backend.

---

## 1. Obiettivo

Dare all'app un'identità nuova chiamata **ForgE**, con estetica medievale/engraved ("lama d'acciaio nella fucina"), mantenendo invariata struttura, UX e funzionalità. Il carattere arriva da palette (ferro freddo + rosso), lettering blackletter sui titoloni e texture incisa leggera — **senza ornamenti pesanti né effetti che cadano nel ridicolo**.

L'app è web vanilla (HTML/CSS/JS) con design token centralizzati in `css/tokens.css` e un CSS per schermata. Il backend (`backend/*.gs`, Google Apps Script) **non viene toccato**.

---

## 2. Palette ("Ferro freddo + rosso")

Neutri acciaio/grigio-bluastro freddi. Rosso solo per azioni, accenti e PR-vivi. Nessun glow (già flat).

Nuovi valori in `:root` (`tokens.css`):

| Token | Valore indicativo | Ruolo |
|---|---|---|
| `--bg` | `#0c0e11` | nero-antracite freddo |
| `--surface` | `#13161b` | card base, acciaio brunito |
| `--surface-2` | `#1a1e25` | superfici sollevate |
| `--surface-3` | `#232830` | controlli, hover |
| `--surface-4` | `#2d333d` | elementi attivi |
| `--border` | `#2a3038` | bordo standard |
| `--border-soft` | `#1d222a` | bordo tenue |
| `--text` | `#e8e9ec` | bianco-osso |
| `--text-dim` | `#9aa1ad` | grigio acciaio |
| `--text-dim-2` | `#5b626e` | grigio spento |
| `--accent` | `#c1272d` | rosso sangue (dalla "E" del logo) — azioni primarie, accenti |
| `--accent-dim` | `#9c1f24` | hover/pressed accent |
| `--accent-ghost` | `rgba(193,39,45,0.13)` | sfondi tenui accent |
| `--gold` *(rinominato concettualmente come "argento")* | `#c9ced8` | PR / evidenze → **argento brunito** (ex giallo). Famiglia metallo, non litiga col rosso. |
| `--gold-ghost` | `rgba(201,206,216,0.10)` | sfondo tenue PR |
| `--danger` | `#e34b3a` | rosso più acceso/aranciato, distinto dall'accent, per stati di pericolo/eliminazione |
| `--danger-ghost` | `rgba(227,75,58,0.13)` | sfondo tenue danger |

**Nota chiave:** il nome del token `--gold` resta invariato (per non rompere i riferimenti nei CSS schermata), ma il **valore** diventa argento brunito. Stesso per `--gold-ghost`.

`--danger` e `--accent` sono entrambi rossi ma volutamente distinti (accent = sangue scuro, danger = rosso-arancio acceso) così "azione primaria" e "pericolo" restano leggibili.

---

## 3. Tipografia

Tre ruoli, niente serif:

- **`--font-display`** (nuovo token): `"UnifrakturCook", "UnifrakturMaguntia", serif` — blackletter. **Solo** per: nome app "ForgE", titoloni hero, numeri-hero molto grandi. Mai per corpo o dati piccoli.
- **`--font` (corpo/UI)**: `"Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif` — corpo, testo lungo, dati/numeri. Massima leggibilità.
- **`--font-cond`** (nuovo token): `"Oswald", "Inter", sans-serif` — condensato/marziale per label, bottoni, titoli di sezione (spesso maiuscoletto con letter-spacing).
- **`--font-mono`**: resta un mono per timer/valori incolonnati. Proposta `"JetBrains Mono", "Ubuntu Mono", monospace`. (Se non vale la pena aggiungere un import, si può tenere il mono attuale: decisione in fase di implementazione, default = JetBrains Mono.)

**Import font** in cima a `tokens.css` via Google Fonts: UnifrakturCook, Inter (300–700), Oswald (300–600), JetBrains Mono. Si rimuove l'import Ubuntu attuale.

**Regola d'oro leggibilità:** blackletter non scende mai sotto dimensioni "title". Tutto ciò che è interattivo o numerico-denso usa Inter o Oswald.

---

## 4. Texture & dettagli "engraved"

Sobri, leggeri, niente glow:

- **Grana/noise sottile** sul `body` (richiama il nero graffiato del logo): overlay a bassissima opacità via `background-image` (SVG noise inline o pattern data-URI) in `base.css`. Pseudo-elemento `body::before` fixed, `pointer-events: none`, opacità ~0.03–0.05.
- **Bordi incisi**: dove ci sono bordi pieni su card/bottoni, introdurre un look "hairline doppio" (es. `border` + `box-shadow: inset 0 0 0 1px` tenue, oppure bordo + filetto interno) tramite token/utility, applicato con parsimonia su card principali e dialog.
- **Filetti decorativi** per separatori di sezione: linee sottili (1px) eventualmente con piccolo ornamento centrale, riusando classi esistenti tipo `.section-label`/`.label-micro` senza cambiarne il markup.
- Nessun ornamento pesante (niente fregi/cornici barocche). Il mood resta flat + inciso.

Questi effetti vivono in `base.css` e nei token; le schermate li ereditano.

---

## 5. Logo & splash

- **Placeholder SVG** "ForgE": mark coerente (incudine + diamante + lettering blackletter) creato come `assets/logo.svg` (sostituisce/aggiorna quello attuale). L'HTML continua a puntare a `assets/logo.png` con fallback `onerror` a `assets/logo.svg`, così **quando l'utente metterà il PNG reale in `assets/logo.png` apparirà automaticamente** senza modifiche al codice.
- **Splash** (`splash.css` + markup in `index.html`): nome "ForgE" in blackletter (`--font-display`); barra di caricamento ristilizzata "barra di forgia" (riempimento rosso brace `--accent`, fondo acciaio).
- **`<title>`** in `index.html`: `Lift` → `ForgE`.
- **`--theme-color`** meta: aggiornato al nuovo `--bg`.
- Ogni stringa **visibile** "LIFT"/"Lift" → "ForgE" (splash-name, title). Nomi di classi CSS, ID, commenti e codice JS/GS che citano "lift" **restano invariati**.

---

## 6. Scope tecnico (file toccati)

1. **`css/tokens.css`** — riscrittura `:root`: nuova palette, nuovi token font (`--font-display`, `--font-cond`), nuovi import Google Fonts, eventuali token per bordo inciso. Aggiornare il commento d'intestazione (vibe ForgE).
2. **`css/base.css`** — texture noise su `body`, bordi incisi su `.dlg-box`/utility, restyle bottoni dialog (`--accent` rosso, testo leggibile), filetti separatori. Applicare `--font-display`/`--font-cond` dove sensato a livello base.
3. **CSS per schermata** (`home`, `editor`, `exec`, `finish`, `profile`, `history`, `stats`, `library`, `schede`, `splash`) — passare sulle **~83 occorrenze** di colori/font hard-coded e agganciarle ai token. Applicare blackletter ai titoloni hero, Oswald a label/bottoni dove migliora il mood. Nessun cambio di layout/markup salvo lo splash.
4. **`index.html`** — `<title>`, `theme-color`, testo splash "ForgE", riferimenti logo (resta il pattern `onerror`).
5. **`assets/logo.svg`** — nuovo placeholder ForgE.

**Fuori scope:** backend `.gs`, logica JS, struttura dati, rinominare cartelle/file, rinominare classi/ID/commenti che citano "lift".

---

## 7. Criteri di successo

- L'app si apre con splash "ForgE" blackletter e barra rosso-brace.
- Palette ferro-freddo + rosso applicata coerentemente su tutte le schermate; nessun residuo del teal `#00C9A7` o del giallo PR.
- Titoloni in blackletter leggibili; corpo e dati in Inter/Oswald perfettamente leggibili; nessun blackletter su testo piccolo.
- Texture incisa percepibile ma discreta (non disturba la lettura).
- Mettendo un `assets/logo.png` reale, appare al posto del placeholder senza modifiche.
- Nessuna regressione funzionale: navigazione, editor, esecuzione, storico, stats funzionano come prima.

---

## 8. Rischi & mitigazioni

- **"Cadere nel ridicolo":** mitigato limitando blackletter ai titoloni, palette sobria, zero ornamenti pesanti.
- **Leggibilità dati allenamento:** mitigata usando Inter/mono per numeri (peso/reps/timer).
- **Hard-coded sparsi:** le 83 occorrenze vanno riviste a mano schermata per schermata; rischio di mancarne qualcuna → verifica finale con ricerca dei vecchi valori (`#00C9A7`, `Ubuntu`, giallo PR) per assicurarsi che non restino.
- **Performance noise overlay:** usare un pattern leggero (data-URI piccolo), `pointer-events:none`, una sola istanza fixed.
