# Vista "prossimi 5 giorni" nel modal — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Aggiungere al modal di `garage_check.html` una striscia di 5 quadratini cliccabili che mostrano l'esito dei 5 giorni successivi a quello visualizzato, permettendo di navigare avanti nel piano restando dentro il modal.

**Architecture:** Estrarre la logica di calcolo esito (oggi inline nel listener di `verifyBtn`) in una funzione pura `computeOutcome(dateStr, person)`. Aggiungere una funzione `renderStrip(dateStr, person)` che calcola i 5 giorni successivi e popola i quadratini nel DOM, con click handler che richiama `showModal` + si ri-renderizza da sola per la nuova data. `showModal` viene esteso per chiamare `renderStrip` a fine esecuzione (o nasconderla nei casi senza data/persona valida).

**Tech Stack:** HTML/CSS/JS vanilla, nessun build step, nessun framework, nessun test runner automatico — verifica manuale via screenshot headless (come già usato nel mockup di brainstorming) e apertura nel browser.

## Global Constraints

- File unico da modificare: `garage_check.html` (nessun nuovo file, nessuna dipendenza esterna).
- Bordo quadratino: `1px solid rgba(255,255,255,0.35)` su tutti i colori/kind.
- Contenuto quadratino: solo il numero del giorno (es. `4`), nessuna icona aggiuntiva.
- Giorni striscia: 5 giorni di calendario consecutivi successivi alla data del modal, weekend incluso, nessun filtro.
- Colori quadratino: stessi colori/gradienti pieni già definiti per `.modal.si/.no/.smart/.race/.weekend/.neutral`.
- Striscia nascosta (mai renderizzata) quando il modal è nei casi "Manca la data" / "Manca il nome" (nessuna data/persona di riferimento valida).
- Click su un quadratino aggiorna sia il modal (icona/titolo/dettaglio/colore/striscia) sia il campo data principale `giornoInput` in pagina.
- Nessuna modifica al flusso "seleziona giorno + Verifica" fuori dal modal.

---

### Task 1: Estrarre `computeOutcome` e verificare che `verifyBtn` continui a funzionare

**Files:**
- Modify: `garage_check.html:434-504` (listener `verifyBtn`)

**Interfaces:**
- Produces: `computeOutcome(dateStr, person)` → oggetto `{kind, icon, title, detail}` dove:
  - `kind`: uno tra `"weekend"`, `"neutral"`, `"smart"`, `"race"`, `"si"`, `"no"`
  - `icon`: stringa emoji
  - `title`: stringa titolo modal
  - `detail`: stringa dettaglio modal (può contenere `\n`)
  - Non tocca il DOM, non ha side effect, richiama solo `DATA`, `DAY_IT`, `DAY_FULL_IT` (già definite globalmente sopra nel file).
- Consumes: `DATA` (oggetto globale con `start`, `end`, `people`), `DAY_IT`, `DAY_FULL_IT` (già esistenti in `garage_check.html:356-359`).

Questo task isola la logica di business esistente senza cambiarne il comportamento osservabile: il pulsante "Verifica" deve produrre esattamente gli stessi modal di prima.

- [ ] **Step 1: Scrivere `computeOutcome` sopra al listener di `verifyBtn`**

Inserire subito prima di `document.getElementById("verifyBtn").addEventListener(...)` (circa `garage_check.html:434`):

```javascript
  function computeOutcome(dateStr, person) {
    const date = new Date(dateStr + "T00:00:00");
    const dow = date.getDay(); // 0=Sun..6=Sat

    if (dow === 0 || dow === 6) {
      const weekendFull = dow === 0 ? "domenica" : "sabato";
      return {
        kind: "weekend", icon: "🌊", title: "VATTINNE A MARE!",
        detail: `Ma l'hai controllato bene il calendario? La data selezionata è ${weekendFull}, altro che garage: rilassati!`
      };
    }

    if (dateStr < DATA.start || dateStr > DATA.end) {
      return {
        kind: "neutral", icon: "🚧", title: "Nessun dato",
        detail: "La data selezionata è fuori dal periodo del piano (04/08–30/09/2026)."
      };
    }

    const dayCode = DAY_IT[dow];
    const dayFull = DAY_FULL_IT[dow];

    if (person.smart.includes(dayCode)) {
      return {
        kind: "smart", icon: "🛌", title: "STATT A CAST!",
        detail: `${person.nome}, ${dayFull} è il tuo giorno di smart working. Resta a casa!`
      };
    }

    const access = person.access[dateStr];
    if (access === "SI") {
      if (dayCode === "VEN") {
        return {
          kind: "race", icon: "🏁", title: "FUSCE!!",
          detail: `${person.nome}, ${dayFull} l'accesso è libero ma i posti sono limitati: chi primo arriva....`
        };
      }
      return {
        kind: "si", icon: "👍", title: "TRASE!",
        detail: `${person.nome}, ${dayFull} hai accesso al garage. Verifica che non ci sia Consiglio e buona giornata!`
      };
    }

    return {
      kind: "no", icon: "🖐️", title: "DAFFÒRE!",
      detail: `${person.nome}, ${dayFull} non è il tuo turno in garage. Riprova un'altra volta!`
    };
  }
```

- [ ] **Step 2: Riscrivere il listener di `verifyBtn` per usare `computeOutcome`**

Sostituire il corpo del listener esistente (`garage_check.html:434-504`) con:

```javascript
  document.getElementById("verifyBtn").addEventListener("click", () => {
    const dateStr = giornoInput.value;
    const personId = select.value;

    if (!dateStr) {
      showModal("neutral", "📅", "Manca la data", "Seleziona prima un giorno da verificare.");
      return;
    }
    if (!personId) {
      showModal("neutral", "🙋", "Manca il nome", "Seleziona chi sei dal menu a tendina.");
      return;
    }

    const person = DATA.people.find(p => String(p.id) === String(personId));
    const outcome = computeOutcome(dateStr, person);
    showModal(outcome.kind, outcome.icon, outcome.title, outcome.detail, dateStr, person);
  });
```

Nota: `showModal` riceve ora anche `person` come sesto argomento — verrà usato nel Task 3 da `renderStrip`. In questo task `showModal` non è ancora stata modificata per accettarlo: aggiungere il parametro `person` alla firma esistente di `showModal` (`garage_check.html:403`) fin da questo step, senza ancora usarlo nel corpo:

```javascript
  function showModal(kind, icon, title, detail, dateStr, person) {
    modal.className = "modal " + kind;
    modalIcon.textContent = icon;
    modalTitle.textContent = title;
    modalDate.textContent = formatDayLabel(dateStr);
    modalDetail.textContent = detail;
    raceBar.style.display = kind === "race" ? "block" : "none";
    overlay.classList.add("open");
  }
```

- [ ] **Step 3: Verifica manuale nel browser**

Aprire `garage_check.html` nel browser (doppio click o `start garage_check.html`), selezionare una persona e una data nota (es. una data "SI" non di venerdì, poi una di venerdì, poi una "NO", poi uno smart day, poi un weekend, poi una data fuori 04/08–30/09/2026) e premere "Verifica accesso" per ognuna. Confermare che titolo/icona/colore/dettaglio del modal sono identici a prima della modifica (nessun cambiamento visibile atteso in questo task).

- [ ] **Step 4: Commit**

```bash
git add garage_check.html
git commit -m "refactor: estrae computeOutcome dalla logica del pulsante verifica"
```

---

### Task 2: Aggiungere markup e stile CSS della striscia dei 5 giorni

**Files:**
- Modify: `garage_check.html` (blocco `<style>`, circa righe 294-317, e markup del modal, circa riga 350)

**Interfaces:**
- Produces: elemento `<div class="day-strip" id="dayStrip"></div>` nel markup del modal, e regole CSS `.day-strip`, `.day-strip .day-chip` (con varianti `.si/.no/.smart/.race/.weekend/.neutral`).
- Consumes: nessuna (solo markup/CSS statico, popolato a runtime nel Task 3).

- [ ] **Step 1: Aggiungere il markup del contenitore striscia nel modal**

In `garage_check.html`, dentro `<div class="modal" id="modal">`, subito dopo `<button class="close-btn" id="closeBtn">Chiudi</button>` (circa riga 351), aggiungere:

```html
      <div class="day-strip" id="dayStrip" style="display:none;"></div>
```

Il markup completo del modal diventa:

```html
  <div class="overlay" id="overlay">
    <div class="modal" id="modal">
      <div class="icon" id="modalIcon"></div>
      <div class="title" id="modalTitle"></div>
      <div class="modal-date" id="modalDate"></div>
      <div class="race-bar" id="raceBar" style="display:none;"></div>
      <div class="detail" id="modalDetail"></div>
      <button class="close-btn" id="closeBtn">Chiudi</button>
      <div class="day-strip" id="dayStrip" style="display:none;"></div>
    </div>
  </div>
```

- [ ] **Step 2: Aggiungere le regole CSS della striscia**

Nel blocco `<style>`, subito dopo la regola `.modal .close-btn:active { ... }` (circa `garage_check.html:311`), aggiungere:

```css
  .modal .day-strip {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    gap: 8px;
    margin-top: 20px;
  }
  .modal .day-chip {
    aspect-ratio: 1 / 1;
    border-radius: 13px;
    border: 1px solid rgba(255,255,255,0.35);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.85rem;
    font-weight: 700;
    color: #fff;
    cursor: pointer;
    user-select: none;
    background: none;
    padding: 0;
    font-family: inherit;
  }
  .modal .day-chip.si { background: linear-gradient(160deg, #2fbf6b, #1f9d55); }
  .modal .day-chip.no { background: linear-gradient(160deg, #ff2800, #a30000); }
  .modal .day-chip.smart { background: linear-gradient(160deg, #8a6bff, #6a4cff); }
  .modal .day-chip.race { background: linear-gradient(160deg, #ffb020, #ff7a1a); }
  .modal .day-chip.weekend { background: linear-gradient(160deg, #6ec6e0, #3f96b8); }
  .modal .day-chip.neutral { background: linear-gradient(160deg, #7c8794, #5a636e); }
```

Nella media query esistente `@media (max-width: 380px)` (circa `garage_check.html:313-316`), aggiungere una riga per ridurre leggermente il gap su schermi stretti:

```css
  @media (max-width: 380px) {
    .modal .icon { font-size: 4.2rem; }
    .modal .title { font-size: 1.7rem; }
    .modal .day-strip { gap: 6px; }
  }
```

- [ ] **Step 3: Verifica manuale nel browser**

Aprire `garage_check.html`, premere "Verifica accesso" per aprire un modal qualsiasi: la striscia non deve essere visibile (resta `display:none`, nessun contenuto ancora generato). Confermare che il layout del modal non è cambiato (nessuna striscia vuota visibile, nessun overflow).

- [ ] **Step 4: Commit**

```bash
git add garage_check.html
git commit -m "style: aggiunge markup e CSS per la striscia dei 5 giorni nel modal"
```

---

### Task 3: Implementare `renderStrip` e collegarla a `showModal`

**Files:**
- Modify: `garage_check.html` (funzione `showModal`, circa riga 403, e area subito sotto `computeOutcome`)

**Interfaces:**
- Consumes: `computeOutcome(dateStr, person)` (Task 1), elemento `#dayStrip` (Task 2), `DATA.start`/`DATA.end`, `giornoInput`, `updateDaySummary()` (esistente, `garage_check.html:418-421`).
- Produces: `renderStrip(dateStr, person)` — calcola i 5 giorni successivi a `dateStr`, popola `#dayStrip` con 5 `<button class="day-chip {kind}">{giorno}</button>`, e imposta un click handler su ciascuno che richiama `computeOutcome` + `showModal` per la nuova data. `showModal` estesa per chiamare `renderStrip` (o nasconderla) in base a `dateStr`/`person`.

- [ ] **Step 1: Scrivere `renderStrip` subito dopo `computeOutcome`**

Inserire dopo la chiusura di `computeOutcome` (fine Task 1, prima del listener `verifyBtn`):

```javascript
  const dayStrip = document.getElementById("dayStrip");

  function addDays(dateStr, n) {
    const date = new Date(dateStr + "T00:00:00");
    date.setDate(date.getDate() + n);
    const y = date.getFullYear();
    const m = String(date.getMonth() + 1).padStart(2, "0");
    const d = String(date.getDate()).padStart(2, "0");
    return `${y}-${m}-${d}`;
  }

  function renderStrip(dateStr, person) {
    dayStrip.innerHTML = "";
    for (let i = 1; i <= 5; i++) {
      const chipDate = addDays(dateStr, i);
      const outcome = computeOutcome(chipDate, person);
      const dayNum = String(parseInt(chipDate.split("-")[2], 10));

      const chip = document.createElement("button");
      chip.type = "button";
      chip.className = "day-chip " + outcome.kind;
      chip.textContent = dayNum;
      chip.addEventListener("click", () => {
        giornoInput.value = chipDate;
        updateDaySummary();
        showModal(outcome.kind, outcome.icon, outcome.title, outcome.detail, chipDate, person);
      });
      dayStrip.appendChild(chip);
    }
    dayStrip.style.display = "grid";
  }
```

- [ ] **Step 2: Estendere `showModal` per chiamare/nascondere `renderStrip`**

Sostituire la funzione `showModal` (aggiornata nel Task 1 con il parametro `person`) con:

```javascript
  function showModal(kind, icon, title, detail, dateStr, person) {
    modal.className = "modal " + kind;
    modalIcon.textContent = icon;
    modalTitle.textContent = title;
    modalDate.textContent = formatDayLabel(dateStr);
    modalDetail.textContent = detail;
    raceBar.style.display = kind === "race" ? "block" : "none";

    if (dateStr && person) {
      renderStrip(dateStr, person);
    } else {
      dayStrip.style.display = "none";
    }

    overlay.classList.add("open");
  }
```

Questo copre automaticamente i due casi "Manca la data" / "Manca il nome" (chiamati senza `dateStr`/`person` in `garage_check.html`, vedi righe con `showModal("neutral", "📅", ...)` e `showModal("neutral", "🙋", ...)`): la striscia resta nascosta perché `dateStr` è `undefined`.

- [ ] **Step 3: Verifica manuale nel browser — striscia base**

Aprire `garage_check.html`, selezionare una persona e una data "SI" infrasettimanale non di venerdì, premere "Verifica accesso". Confermare:
- La striscia appare sotto "Chiudi" con 5 quadratini colorati.
- I numeri corrispondono ai 5 giorni successivi alla data scelta (calendario reale, weekend incluso).
- I colori corrispondono agli esiti attesi per quella persona in quei giorni (incrociare con `data_v3.json`/i dati embeddati in `DATA` se serve verificare).
- Ogni quadratino ha un bordo sottile visibile.

- [ ] **Step 4: Verifica manuale nel browser — navigazione e sincronizzazione**

Con il modal ancora aperto, cliccare un quadratino: confermare che il modal si aggiorna (icona/titolo/dettaglio/colore) per la nuova data, che la striscia si rigenera con i 5 giorni successivi alla nuova data, e che chiudendo il modal il campo "Giorno" in pagina mostra ora la data appena selezionata dal quadratino (non quella originale).

- [ ] **Step 5: Verifica manuale nel browser — casi limite**

Testare:
- Una data entro pochi giorni dal 30/09/2026 (fine periodo): i quadratini oltre il 30/09 devono apparire grigi (`neutral`) e, se cliccati, aprire il modal "Nessun dato" con striscia aggiornata di conseguenza (che a sua volta avrà tutti/quasi tutti i quadratini grigi).
- Un venerdì con accesso "SI": il quadratino di quel giorno deve essere arancio (`race`).
- Un giorno di smart working della persona selezionata: quadratino viola (`smart`).
- Premere "Verifica accesso" senza aver scelto data o persona: confermare che la striscia NON appare (nessun quadratino, nessuno spazio vuoto anomalo nel modal).

- [ ] **Step 6: Commit**

```bash
git add garage_check.html
git commit -m "feat: aggiunge vista prossimi 5 giorni cliccabile nel modal"
```

---

### Task 4: Aggiornare TODO.md

**Files:**
- Modify: `TODO.md:107-110`

**Interfaces:** Nessuna (solo documentazione).

- [ ] **Step 1: Segnare la voce come completata**

In `TODO.md`, sostituire:

```markdown
- [ ] **Vista "prossimi 5 giorni"** per la persona selezionata: invece di
      verificare un giorno alla volta, mostrare a colpo d'occhio la
      settimana corrente (utile per organizzarsi in anticipo).
      - Sforzo: medio (nuova sezione UI, non solo modal).
```

con:

```markdown
- [x] **Vista "prossimi 5 giorni"** per la persona selezionata: striscia di
      5 quadratini cliccabili nel modal, uno per ciascuno dei 5 giorni
      successivi a quello mostrato, colorati come l'esito corrispondente;
      cliccando si naviga avanti restando nel modal.
      - Fatto: 2026-07-30.
```

- [ ] **Step 2: Commit**

```bash
git add TODO.md
git commit -m "docs: segna vista prossimi 5 giorni come completata"
```
