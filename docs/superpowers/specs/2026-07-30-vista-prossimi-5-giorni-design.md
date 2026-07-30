# Design: Vista "prossimi 5 giorni" nel modal

Data: 2026-07-30

## Contesto

`garage_check.html` mostra, dopo aver premuto "Verifica", un modal con l'esito
del giorno selezionato per la persona scelta (sì / no / smart / race /
weekend / fuori periodo). Il TODO elenca tra le feature utente una "Vista
prossimi 5 giorni" per organizzarsi in anticipo senza dover ripetere
data+verifica un giorno alla volta.

Questo design integra la vista direttamente nel modal esistente, invece di
creare una sezione UI separata.

## Obiettivo

Nel modal, sotto ai contenuti già presenti, aggiungere una riga di 5
quadratini con bordi stondati — uno per ciascuno dei 5 giorni successivi al
giorno mostrato nel modal. Ogni quadratino comunica a colpo d'occhio, tramite
colore, cosa aspettarsi in quel giorno, e permette di navigare avanti
restando dentro il modal.

## Comportamento

- **Giorni mostrati**: i 5 giorni di calendario successivi alla data
  corrente del modal (weekend inclusi, nessun filtro).
- **Colore quadratino**: stesso colore/gradiente pieno usato dal modal per
  quell'esito — verde (`si`), rosso (`no`), viola (`smart`), arancio
  (`race`, venerdì), celeste (`weekend`), grigio (`neutral`, fuori periodo
  pubblicato).
- **Bordo**: `1px solid rgba(255,255,255,0.35)` su ogni quadratino, per
  restare visibile anche quando il colore del quadratino coincide con quello
  di sfondo del modal (es. quadratino verde dentro modal verde).
- **Contenuto quadratino**: solo il numero del giorno (es. `4`), minimal,
  centrato. Nessuna icona aggiuntiva, anche per i casi race/weekend — il
  colore basta.
- **Click su un quadratino**: ricalcola l'esito per quella data (stessa
  persona selezionata) e aggiorna il modal aperto — icona, titolo, dettaglio,
  colore di sfondo — e rigenera la striscia dei 5 giorni successivi a
  partire dalla nuova data. L'utente può così scorrere in avanti restando
  dentro il modal.
- **Sincronizzazione**: il click aggiorna anche il campo data principale
  (`giornoInput`) in pagina, così se l'utente chiude il modal lo stato
  visibile in pagina resta coerente con l'ultimo giorno consultato.
- **Giorni fuori periodo pubblicato**: quadratino grigio neutro (stesso
  stile del modal `neutral`), cliccabile; il click apre il modal "Nessun
  dato" con lo stesso messaggio già usato oggi per le date fuori periodo.

## Implementazione

- Estrarre la logica di calcolo esito (oggi inline nel listener di
  `verifyBtn`: weekend → fuori periodo → smart → race/sì/no) in una
  funzione pura `computeOutcome(dateStr, person)` che ritorna
  `{kind, icon, title, detail}` senza toccare il DOM.
- Sia il listener di `verifyBtn` sia la costruzione della striscia dei 5
  giorni richiamano `computeOutcome`, evitando la duplicazione della logica
  di business già esistente.
- Nuova funzione `renderStrip(dateStr, person)` che, dato il giorno
  "centrale" del modal, calcola i 5 giorni successivi, richiama
  `computeOutcome` per ciascuno e popola i quadratini (colore da `kind`,
  numero del giorno, click handler che richiama `showModal` con i nuovi
  dati e poi `renderStrip` di nuovo con la nuova data).
- `showModal` viene esteso per accettare/richiamare `renderStrip` alla fine,
  così la striscia si aggiorna automaticamente ogni volta che il modal
  cambia contenuto (sia dal pulsante Verifica sia dal click su un
  quadratino).

## Spazio e stile

- Modal ha `max-width: 380px`: 5 quadratini con gap ridotto stanno
  comodamente (~60-64px l'uno su schermo pieno).
- Quadratino: altezza ~44-48px, `border-radius` coerente con lo stile
  arrotondato del modal (12-14px), font numero ridotto (~0.85rem, peso 700).
- Sotto i 380px (media query già esistente in `garage_check.html`), ridurre
  leggermente padding/gap dei quadratini per mantenere leggibilità.

## Non in scope

- Nessuna modifica al flusso principale "seleziona giorno + Verifica" fuori
  dal modal.
- La striscia esiste solo dentro il modal aperto, non come sezione
  permanente in pagina.
- Nessuna icona aggiuntiva nei quadratini oltre al numero e al colore.
