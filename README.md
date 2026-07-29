# Garage Check

SPA mobile-first per verificare, giorno per giorno, se una persona ha accesso al garage aziendale in base al piano di rotazione tra 21 persone.

**App pubblicata**: https://enzolamacchia.github.io/checkingarage/garage_check.html

## Cos'è

Il garage ha capienza limitata in alcuni giorni della settimana. Ogni persona ha uno o più giorni fissi di smart working (in cui non può comunque accedere) e, nei giorni a capienza limitata, un turno a rotazione. `garage_check.html` permette di selezionare una data e il proprio nome per sapere subito se quel giorno si ha accesso, con un modal a tema "ticket da parcheggio" (SI/NO/smart/weekend/fuori periodo).

L'app è **installabile come PWA** (icona sulla home screen, funziona offline dopo la prima apertura) e ricorda l'ultima persona selezionata.

## Regole di accesso attuali (piano v3)

Periodo valido: **04/08/2026 – 30/09/2026**.

- **Lunedì, martedì, mercoledì, giovedì**: capienza fissa di esattamente 7 accessi al giorno, a rotazione.
- **Venerdì**: unico giorno ad accesso libero (chiunque non sia in smart working quel giorno).
- **Equità di rotazione**: ogni finestra di 3 giorni-limitati-consecutivi garantisce almeno 1 accesso a testa tra gli idonei di quella finestra.
- **KETTY**: accesso fisso ogni mercoledì (non in rotazione), più accesso libero di venerdì se non in smart; esclusa dal vincolo di equità sugli altri giorni limitati.
- **Ultimo giorno residuo** (30/09, finestra di equità incompleta): assegnazione casuale tra tutti gli idonei, Ketty inclusa alla pari.

Le regole possono cambiare nel tempo: sono implementate in `build_v3.py`, non hardcoded nell'HTML.

## Struttura del repo

| File | Descrizione |
|---|---|
| `garage_check.html` | La SPA pubblicata su GitHub Pages. Dati embeddati inline come JSON. |
| `build_v3.py` | Script Python **persistente e riutilizzabile** che genera il piano a partire da `LISTA GARAGE.xlsx`. Non va cancellato: va rilanciato ogni volta che cambiano regole, periodo o anagrafica. |
| `data_v3.json` | Dati generati da `build_v3.py`, stesso contenuto embeddato in `garage_check.html`. |
| `piano_rotazione_garage_v3.html` | Vista tabellare di tutto il piano (una riga per persona, una colonna per giorno), con totali per persona e per giornata. |
| `Piano_Rotazione_Garage_v3.xlsx` | Stesso piano in formato Excel (fogli Regole / Piano giornaliero / Riepilogo capienza). |
| `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png` | Configurazione PWA (installabilità e funzionamento offline). |
| `devEL-logo120trasp.png` | Logo mostrato nella card della SPA. |

File locali non tracciati (vedi `.gitignore`): `LISTA GARAGE.xlsx` (anagrafica sorgente), versioni precedenti del piano (v1, v2), `TODO.md` (elenco privato di migliorie future).

## Come rigenerare il piano

Quando cambia un giorno di smart working, l'anagrafica, o le regole di accesso:

1. Aggiornare `LISTA GARAGE.xlsx` (colonne: ID, NOME, GIORNO SMART, TARGA).
2. Rilanciare lo script:
   ```
   py build_v3.py
   ```
   Lo script valida automaticamente il risultato (capienza esatta 7/giorno, presenza fissa di Ketty il mercoledì, equità di rotazione) e si ferma con un errore se qualcosa non torna, prima di scrivere i file di output.
3. Copiare il nuovo `data_v3.json` dentro `garage_check.html` (sostituendo il blocco `const DATA = {...}`), oppure aggiornare lo script per farlo automaticamente.
4. Verificare visivamente l'app in locale, poi `git add`, `commit`, `push`.

## Sviluppo locale

Nessuna build necessaria: `garage_check.html` è una pagina statica autosufficiente. Per testare service worker e installabilità PWA (che richiedono un contesto sicuro, non `file://`), servire la cartella con un server locale, ad esempio:

```
py -m http.server 8765
```

e aprire `http://127.0.0.1:8765/garage_check.html`.
