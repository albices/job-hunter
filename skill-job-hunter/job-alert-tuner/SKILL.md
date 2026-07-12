---
name: job-alert-tuner
description: >-
  Metriche di tuning delle ricerche del sistema Job Hunter a partire dal
  source-log su Google Drive (job-hunting/source-log.csv): overlap tra
  ricerche, numerosità per ricerca, tasso di annunci fuori scope. Usa
  SEMPRE questa skill quando l'utente chiede: "come stanno andando le
  ricerche/gli alert", "quali ricerche rendono", "ci sono alert
  doppi/inutili", "tuning del sourcing", "metriche della routine",
  "conviene togliere qualche alert", o vuole capire se le fonti del
  digest producono rumore. Produce analisi e raccomandazioni in chat: NON
  modifica da sola profilo o alert.
---

# job-alert-tuner

Modulo 1.2.2 del progetto Job Hunter. Analizza il `source-log` prodotto dalla routine `job-watch` e risponde a tre domande: quali ricerche portano volume, quali si sovrappongono, quali portano rumore (annunci fuori scope). L'output informa; le modifiche restano all'utente, tramite `job-search-profile` (1.2) per i criteri e `job-alert-config` (1.2.1) per riallineare gli alert.

## Contratto dati

Il `source-log` vive su **Google Drive**, `job-hunting/source-log.csv` (non su git: lì vive solo `state.json`). Lo schema completo — colonne, enum, semantica "una riga = un annuncio osservato da una ricerca in una run" — è in `references/source-log-schema.md`: **leggilo prima di ogni analisi**, è il contratto che questa skill stessa ha definito e che la strumentazione della routine dovrà rispettare.

## Caso base da gestire per primo: il log non c'è (previsto, non un errore)

Il `source-log` è scritto dalla ROUTINE, e la strumentazione della routine potrebbe non essere ancora stata fatta (è un task aperto del progetto, fuori dal perimetro dello Studio). Quindi, prima di tutto:

1. Verifica il connettore Drive (`tool_search`), poi cerca `job-hunting/source-log.csv`.
2. **File assente** → spiega con calma: "il source-log non esiste ancora: lo produce la routine job-watch, che va prima strumentata per scriverlo (il contratto è già definito in questa skill). Finché non succede, non ci sono metriche calcolabili." Nessun crash, nessun tentativo di ricostruire il log da altre fonti (email digest, state.json): dati parziali produrrebbero metriche fuorvianti.
3. **File vuoto o solo header** → stesso messaggio, più il fatto che il file esiste ma nessuna run ha ancora loggato.
4. **File presente ma con poche run** (1-2 `run_id` distinti) → calcola comunque, ma dichiara che con così poche esecuzioni le metriche sono indicative, non conclusive.

## Parsing (robusto per costruzione)

Usa il code tool (pandas) per leggere il CSV. Righe malformate (colonne mancanti, enum sconosciuti in `esito` o `fonte`): scartale, contale, e riporta il conteggio nell'output ("N righe malformate ignorate") — non fermarti e non correggerle inventando valori. Se le righe malformate superano ~20% del totale, segnala che il log è probabilmente corrotto o che la routine ha deviato dal contratto: in quel caso le metriche non sono affidabili e la cosa va sistemata alla fonte.

## Metriche (definizioni esatte)

Calcola sulle run disponibili (o su una finestra se l'utente la chiede, es. "ultimo mese" filtrando `run_id`):

1. **Numerosità per ricerca** — per ogni `ricerca_id`: righe totali, media per run, trend (prime run vs ultime). Una ricerca che porta ~0 annunci per molte run è morta o mal configurata.
2. **Overlap tra ricerche** — per ogni coppia di `ricerca_id`: quanti `annuncio_id` condividono nella stessa run (contando anche le righe `scartato_dedup`, che esistono apposta). Deriva per ogni ricerca la **resa unica**: quota di annunci portati SOLO da quella ricerca. Resa unica bassa + alto overlap con un'altra = candidata alla rimozione.
3. **Tasso fuori scope per ricerca** — quota di righe con `esito` in {`scartato_lingua`, `scartato_livello`} sul totale della ricerca. Alto fuori scope = query troppo larga (es. location che pesca annunci in lingua esclusa) — costa tempo di pipeline anche se il digest resta pulito.
4. (Di contorno) **quota `non_lavorato_cap`** complessiva: se è ricorrente, il cap della routine sta tagliando materiale — informazione utile per `parametri_esecuzione`.

## Output (in chat)

1. Una tabella riassuntiva per `ricerca_id`: volume medio/run, resa unica %, fuori scope %, note.
2. Le coppie con overlap rilevante.
3. **2-4 raccomandazioni qualitative**, nello stile del progetto (pesate, non binarie): non "elimina la ricerca X" ma "X porta il 90% di annunci già portati da Y e quasi nulla di unico: candidata alla rimozione — la decisione è tua". Ogni raccomandazione indica anche DOVE si agisce: criteri → `job-search-profile`, alert sulle piattaforme → `job-alert-config`.
4. Le soglie usate (es. "resa unica < 15% = bassa") sono euristiche dichiarate nel testo, mai tagli automatici.

## Cosa NON fare

- Non modificare `search-profile.yaml` né generare istruzioni alert: solo raccomandare e rimandare a 1.2 / 1.2.1.
- Non ricostruire dati mancanti da fonti alternative (digest email, state.json).
- Non presentare metriche su 1-2 run come conclusive.
- Non inventare colonne o esiti fuori dal contratto: se il log contiene valori non previsti, è la routine che ha deviato — segnalalo, non adattare silenziosamente il contratto.
