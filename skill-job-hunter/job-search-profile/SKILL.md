---
name: job-search-profile
description: >-
  Editor del search-profile del sistema Job Hunter (file
  job-hunting/search-profile.yaml su Google Drive). Usa SEMPRE questa skill
  quando l'utente vuole modificare i criteri di ricerca lavoro DOPO
  l'onboarding: "aggiorna il mio profilo di ricerca", "cambia i ruoli che
  cerco", "aggiungi/togli una location", "escludi gli stage", "non voglio
  più annunci in spagnolo", "alza/abbassa la seniority", "cambia i settori",
  "aggiorna le esclusioni" — anche per richieste parziali tipo "aggiungi
  Amsterdam" o "togli i ruoli manageriali" se il contesto è la ricerca
  lavoro. NON usare per l'onboarding iniziale né per creare il profilo da
  zero: quello è mestiere della skill agent-config. NON usare per il
  master-profile (esperienze, CV, retribuzione): questa skill tocca solo i
  criteri di ricerca.
---

# job-search-profile

Modulo 1.2 del progetto Job Hunter. Modifica un `search-profile.yaml` **già esistente** nella cartella `job-hunting` su Google Drive, dopo l'onboarding. È un editor, non un creatore: la creazione iniziale e lo schema canonico vivono in `agent-config` (1.1) — questa skill non li duplica.

## Cosa fa / cosa NON fa

- FA: legge `job-hunting/search-profile.yaml` da Drive, applica modifiche puntuali richieste dall'utente (ruoli, location, seniority, esclusioni, settori, lingue annuncio, fonti, parametri esecuzione), riscrive il file.
- NON FA: creare il profilo da zero (se manca → redirect ad `agent-config`), modificare `master-profile.yaml` (chi è l'utente, non cosa cerca), inventare campi fuori schema, rifare l'intervista di onboarding.

## Schema di riferimento (contratto)

Lo schema canonico è `search-profile.schema.yaml` dentro la skill `agent-config` (tipicamente `/mnt/skills/user/agent-config/references/search-profile.schema.yaml`). Leggilo prima di modificare: definisce campi validi, enum e semantica. NON esiste una copia dello schema qui dentro — è voluto, per avere un'unica fonte di verità.

Se il file schema non è raggiungibile (installazione parziale del pacchetto skill): usa come contratto la struttura del `search-profile.yaml` esistente dell'utente (i campi che già contiene sono per costruzione conformi allo schema), segnala all'utente che lo schema canonico non è raggiungibile, e limita le modifiche ai campi già presenti nel file — niente campi nuovi a schema non verificabile.

Un'istanza di esempio compilata (dati della prima istanza di test del progetto, nessun dato anagrafico) è in `references/example-search-profile.yaml`: usala per capire come si compila un campo, MAI come default da copiare nel profilo di un altro utente.

## Precondizioni

1. **Google Drive** — verifica con `tool_search` (query "Google Drive") che il connettore sia disponibile in lettura E scrittura. Se manca: fermati, spiega che serve, chiedi di collegarlo.
2. **File esistente** — cerca `search-profile.yaml` nella cartella `job-hunting` su Drive. Se NON esiste: NON crearlo qui. Spiega che il profilo si crea con l'onboarding (`agent-config`) e chiedi se vuole avviarlo. Un profilo creato fuori dall'intervista di onboarding salterebbe le domande su vincoli/preferenze e nascerebbe monco.

## Flusso

### 1. Leggi lo stato attuale

Scarica e parsa `job-hunting/search-profile.yaml`. Se lo YAML è corrotto/non parsabile: mostra all'utente il contenuto grezzo e l'errore, chiedi come procedere (correzione manuale guidata campo per campo, oppure ricreazione via `agent-config`) — NON riscrivere silenziosamente un file che non riesci a leggere.

### 2. Interpreta la richiesta → campi target

Mappa la richiesta dell'utente sui campi dello schema. Esempi di mappatura:

- "aggiungi Amsterdam" → `location_target` (nuova voce: chiedi anche `accetta_remoto`/`accetta_ibrido`/`priorita`, non inventarli)
- "togli i ruoli manageriali" → `esclusioni.titoli_da_escludere`
- "non voglio più annunci in spagnolo" → `lingue_annuncio` (chiedi: escluso se prevalente, se obbligatorio, o entrambi? La distinzione cambia il comportamento della routine)
- "cerca anche Data Platform Engineer" → `ruoli_target` (nuovo titolo o sinonimo di uno esistente? Chiedi se ambiguo)
- "alza il cap ad annunci più recenti" → `parametri_esecuzione`

Se la richiesta è vaga ("migliora il profilo", "sistemalo"): mostra i valori attuali raggruppati per sezione e chiedi cosa cambiare — non proporre modifiche di tua iniziativa.

Se la richiesta non ha posto nello schema (es. "escludi le aziende sotto i 50 dipendenti" — non esiste un campo per dimensione azienda nel search-profile): dillo esplicitamente, NON forzare il dato in un campo affine e NON inventare un campo nuovo. Un campo nuovo è un'evoluzione dello schema, che si decide a livello di progetto, non dentro una modifica.

### 3. Mostra il diff, poi conferma

Prima di scrivere, mostra SEMPRE un confronto prima/dopo dei soli campi toccati (non l'intero file, salvo richiesta). Le modifiche multiple in un'unica richiesta si raggruppano in un solo diff e una sola conferma. Aspetta conferma esplicita.

### 4. Riscrivi il file su Drive

Riscrivi `job-hunting/search-profile.yaml` completo (stesso file, stesso percorso), preservando INTATTI tutti i campi non toccati — inclusi eventuali campi valorizzati che la modifica non riguarda. Confronta mentalmente il file in uscita con quello in entrata: l'unica differenza devono essere i campi confermati al passo 3.

Se la scrittura fallisce (connettore scaduto a metà flusso): mostra in chat lo YAML completo aggiornato così l'utente non perde la modifica, spiega il problema e ritenta dopo la riconnessione.

### 5. Promemoria a valle (obbligatorio, non opzionale)

Dopo ogni scrittura riuscita, segnala le conseguenze a valle in base ai campi toccati:

- `ruoli_target`, `location_target`, `seniority` modificati → **gli alert LinkedIn/Indeed sono ora disallineati**: proponi di rigenerare le istruzioni con `job-alert-config` (1.2.1). Gli alert sono impostati a mano dietro login: nessun sistema li aggiorna da solo.
- `lingue_annuncio`, `esclusioni` modificati → nessuna azione manuale: la routine li applica a valle alla prossima esecuzione (leggendo il profilo aggiornato da Drive).
- `parametri_esecuzione`, `fonti` modificati → idem, effetto dalla prossima esecuzione della routine.

## Coerenza con master-profile (avvisa, non bloccare)

Se una modifica contraddice quanto noto dal `master-profile` (es. aggiunge una location in un paese dove `diritto_al_lavoro` risulta `no` o `da_verificare`, o una lingua annuncio che l'utente non ha dichiarato di conoscere): segnala la tensione e chiedi conferma, ma se l'utente conferma procedi — il search-profile è suo, la skill avvisa, non decide. Non leggere il master-profile preventivamente a ogni modifica: solo quando il campo toccato ha una controparte lì (location↔diritto al lavoro/trasferimento, lingue annuncio↔lingue).

## Cosa NON fare

- Non creare il file se manca (→ `agent-config`).
- Non scrivere senza il diff + conferma del passo 3.
- Non riempire campi collaterali con default plausibili non richiesti (es. `priorita: media` su una location nuova senza averlo chiesto).
- Non usare la memoria dell'account per decidere i valori: i valori li dà l'utente attivo, la skill deve funzionare identica per un utente di cui non sai nulla.
- Non rilanciare l'intervista completa di onboarding per una modifica puntuale.
