---
name: role-fit
description: >-
  Valuta il fit tra una job description e il profilo dell'utente
  (master-profile del sistema Job Hunter) e salva l'esito strutturato su
  Google Drive (job-hunting/role-fit/). Usa SEMPRE questa skill quando
  l'utente: incolla una JD o un link a un annuncio chiedendo un parere
  ("valuta questo annuncio", "che fit ho?", "fammi il fit check", "ci sto
  dentro per questa posizione?"), arriva da un annuncio del digest della
  routine job-watch, o chiede di rivalutare/aggiornare l'esito di una
  valutazione già fatta. La valutazione informa, non scarta: mai ridurla a
  un sì/no.
---

# role-fit

Modulo 2.1 del progetto Job Hunter. Confronta una job description con il `master-profile` dell'utente e produce una valutazione qualitativa + un output strutturato su Drive. È il giudizio assistito del sistema: il verdetto finale resta sempre all'utente.

## Input

### 1. Il master-profile

Leggi `job-hunting/master-profile.yaml` da Google Drive (verifica prima il connettore con `tool_search`). Se manca → l'utente non ha fatto l'onboarding: rimanda ad `agent-config` e fermati. Se esiste ma ha vuoti rilevanti per la JD in esame (es. nessun `livello_per_skill`, esperienze senza `risultati_quantificabili`): valuta comunque, ma DICHIARA i limiti ("non ho dati su X nel tuo profilo, questa parte della valutazione è meno solida").

### 2. La JD (regole di intake, decisione fissa del progetto)

- **LinkedIn**: SEMPRE incollata a mano dall'utente. Il fetch delle pagine LinkedIn è bloccato (verificato): se l'utente dà solo un link LinkedIn, chiedi il testo — non tentare il fetch, non valutare dal solo titolo.
- **Indeed**: se l'utente arriva dal digest o da un alert con link/ID Indeed, recupera il testo COMPLETO via connettore Indeed (`get_job_details`); in subordine estrai dal corpo dell'alert su Gmail. Non fidarti del solo titolo o snippet.
- **Manuale**: testo incollato da qualunque altra fonte — va bene.
- **JD troncata o sospetta di esserlo** (finisce a metà frase, mancano requisiti in un annuncio che chiaramente li aveva): chiedi il testo completo, NON valutare a metà — una valutazione su una JD parziale sembra completa e non lo è, che è peggio di nessuna valutazione.

Registra la `sorgente` (enum: `indeed`, `linkedin_alert`, `indeed_alert`, `manuale`) — servirà nell'output e, a valle, alle label del tracker.

## La valutazione (il cuore — stile obbligatorio)

Stile: quello delle valutazioni qualitative del progetto — **2-4 bullet**, sostanza e pesi, zero riempitivo. Tre componenti:

1. **Match principali** (1-3 punti): i punti di forza CONCRETI del profilo rispetto a QUESTA JD — non l'elenco delle skill, ma l'incrocio ("chiedono orchestrazione dati su cloud: 3 anni di pipeline su <piattaforma> coprono il requisito core").
2. **Gap pesati**: per ogni gap, quanto pesa DAVVERO e perché — mai un nudo "manca X". La domanda a cui rispondere: quanto è centrale nella JD, quanto è colmabile, quanto è affine a ciò che il profilo già fa. Esempio del taglio giusto: "chiedono <tecnologia mai usata>: gap reale ma non critico — il pattern è lo stesso di <cosa affine nel profilo>, colmabile in settimane; pesa di più l'assenza di esperienza con <requisito centrale della JD>". Pesi: `critico` / `rilevante` / `marginale`.
3. **Considerazioni**: livello/seniority del titolo vs profilo (es. titolo "Senior" ma requisiti da medio — o il contrario), segnali dall'annuncio (JD fotocopia, range retributivo vs aspettative del profilo se noto, red flag), lingua dell'annuncio se tange le regole del `search-profile`.

Poi lo **score**: enum `forte | buono | parziale | debole` (semantica esatta in `references/role-fit-output.schema.yaml`). Convenzione di progetto: NIENTE punteggio numerico — falsa precisione che invita a filtri a soglia. Lo score non viaggia mai da solo: è il riassunto dei bullet, non il loro sostituto.

**Nessun filtro binario**: anche su un fit `debole` la valutazione spiega perché e si ferma lì — la decisione di candidarsi o no è dell'utente. Non scrivere "te lo sconsiglio" come verdetto: scrivi cosa pesa e lascia il verbo all'utente.

## Output strutturato su Drive

1. Mostra la valutazione in chat (bullet + score).
2. Componi lo YAML secondo `references/role-fit-output.schema.yaml` (leggilo: contiene anche le convenzioni complete di `score` ed `esito`).
3. `esito` iniziale: `valutato`. Se l'utente nello stesso scambio dichiara già la decisione, usa `da_candidare` o `scartato_dopo_fit`.
4. Conferma leggera prima di scrivere ("salvo la valutazione su Drive?" — o procedi se l'utente ha già chiesto esplicitamente di salvare), poi scrivi su Drive: cartella `job-hunting/role-fit/`, file `<YYYY-MM-DD>-<azienda-slug>-<ruolo-slug>.yaml`. Crea la sottocartella `role-fit` se non esiste ancora (dentro `job-hunting`, mai altrove).
5. Se esiste già un file per stessa azienda+ruolo (rivalutazione): chiedi se aggiornare il file esistente o salvarne uno nuovo con la data odierna — non sovrascrivere in silenzio.

## Aggiornare un esito

Se l'utente comunica una decisione su una valutazione passata ("ho deciso di candidarmi a X", "lascia perdere Y"): trova il file in `job-hunting/role-fit/`, aggiorna il campo `esito` (e `note` se dà contesto), riscrivi. Se dice di aver GIÀ creato il task o chiede di crearlo → vedi sotto.

## Ponte verso il tracker (manuale per costruzione)

Chiusa la valutazione, se il fit lo merita PROPONI (mai eseguire d'ufficio) la promozione al tracker: "vuoi che la aggiunga al tracker candidature?". Solo su richiesta esplicita si invoca `application-tracker` (2.3) — decisione fissa del progetto: sourcing → tracker è manuale, nessun task Todoist nasce senza un'azione dell'utente. Alla promozione, aggiorna `esito` a `promosso_a_tracker`: da quel momento lo stato-workflow vive SOLO su Todoist, questo file non lo replica più.

## Cosa NON fare

- Non valutare da solo titolo/snippet: senza il corpo della JD non c'è valutazione, c'è una nota in "da verificare".
- Non inventare esperienze o skill non presenti nel master-profile per migliorare il fit.
- Non produrre punteggi numerici, percentuali di match o classifiche tra annunci diversi.
- Non creare task Todoist automaticamente.
- Non scrivere su Drive senza aver mostrato la valutazione in chat.
