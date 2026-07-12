---
name: application-tracker
description: >-
  Tracker delle candidature del sistema Job Hunter su Todoist (progetto
  "Job Hunter": 1 candidatura = 1 task "Ruolo @ Azienda"). Usa SEMPRE
  questa skill quando l'utente vuole: aggiungere o promuovere una
  candidatura ("aggiungi al tracker", "mi sono candidato a X",
  "promuovi questo annuncio"), aggiornare uno stato ("ho fatto il
  colloquio", "mi hanno rifiutato", "ho ritirato la candidatura"),
  controllare le risposte ("ci sono novità sulle candidature?", "guarda
  se mi hanno risposto"), preparare un follow-up ("prepara il follow-up
  per X"), o vedere la pipeline ("a che punto sono le candidature").
  NON usare per la gestione produttività generale su Todoist
  (backlog, roadmap, task personali): questa skill tocca SOLO il
  progetto candidature.
---

# application-tracker

Modulo 2.3 del progetto Job Hunter: lo stato-workflow delle candidature vive qui, su Todoist. È l'unica fonte di verità per "a che punto è" una candidatura (il `role-fit-output` su Drive si ferma a `promosso_a_tracker`, per costruzione). Due principi sopra tutto:

1. **Nessun task nasce da solo**: la promozione dal digest/valutazione al tracker è SEMPRE un'azione esplicita dell'utente (decisione fissa del progetto). La routine non scrive su Todoist, e questa skill non crea task "per completezza".
2. **Nessun cambio di stato silenzioso**: ogni modifica derivata da una email va mostrata (email + azione proposta) e confermata prima di toccare Todoist.

## Struttura su Todoist (crea al primo uso, con conferma)

- **Progetto**: `Job Hunter`. Se non esiste, proponi di crearlo — non crearlo senza dirlo. Se esiste un progetto omonimo usato per altro, chiedi prima di usarlo.
- **Sezioni = stati ATTIVI** (l'ordine è il funnel): `Da candidare` → `Candidata` → `In corso` → `Offerta`.
- **Chiusura = completamento del task** + una label di esito: `@esito-rifiuto`, `@esito-ritiro`, `@esito-accettata`. Niente sezione "Chiusa": i task completati restano interrogabili in Todoist e il funnel mostra solo ciò che è vivo.
- **Label sorgente** (dalla valutazione o dal digest): `@indeed`, `@linkedin`, `@manuale`.
- **Anatomia del task**:
  - Titolo: `Ruolo @ Azienda` (es. `Data Engineer @ Azienda X`) — esattamente questo formato, è la chiave del matching.
  - Descrizione: link alla JD; riferimento al file role-fit su Drive se esiste (`job-hunting/role-fit/<file>.yaml`); **log eventi datato**, una riga per evento (`2026-07-05 — candidatura inviata`, `2026-07-10 — ack ricevuto`). Il log si APPENDE, mai riscritto.
  - Commenti: riservati al registro email processate (vedi sotto).
  - Due date: SOLO per la prossima azione attesa (follow-up o colloquio), non per scadenze decorative.

## Promozione di una candidatura (manuale)

Su richiesta esplicita ("aggiungila al tracker", "mi sono candidato a X"):

1. **Raccogli il minimo**: ruolo, azienda, link JD, sorgente. Se arrivi da un `role-fit` in chat, hai già tutto; se l'utente arriva dal digest, fatti dare il link.
2. **DEDUP PRIMA di creare** (obbligatorio): cerca nel progetto (task attivi E completati) per azienda e ruolo, con normalizzazione fuzzy: minuscolo, senza punteggiatura, senza suffissi societari (S.r.l., S.p.A., B.V., GmbH, Inc, Ltd, AB, SA), tolleranza per varianti di titolo ("BI Developer" ~ "Business Intelligence Developer"). Match probabile → mostra il task esistente e chiedi: è la stessa candidatura (aggiorno quello) o una posizione diversa nella stessa azienda (creo un nuovo task)? NON creare in caso di dubbio non risolto.
3. **Crea il task**: sezione `Da candidare` se deve ancora inviare, `Candidata` se ha già inviato (chiedi quale delle due, non assumere). Label sorgente. Prima riga di log nella descrizione.
4. Se esiste un role-fit-output per la posizione, ricorda (alla skill `role-fit` o direttamente se il contesto è in chat) di aggiornare il suo `esito` a `promosso_a_tracker`.

## Avanzamenti dichiarati dall'utente

"Ho inviato la candidatura", "ho il colloquio martedì", "mi hanno fatto un'offerta": sposta il task nella sezione corrispondente, appendi la riga di log, gestisci la due date (vedi follow-up). Le dichiarazioni dirette dell'utente non richiedono la conferma extra prevista per le email — è lui la fonte.

Chiusure: "mi hanno rifiutato" → label `@esito-rifiuto`, riga di log, completa il task. "Lascio perdere / ritiro" → `@esito-ritiro`, idem. "Ho accettato!" → `@esito-accettata`, completa, e proponi di chiudere per ritiro le altre candidature ancora attive (proponi: la decisione è sua).

## Follow-up

- Al passaggio in `Candidata`: proponi una due date a **+7 giorni** (default del progetto, dichiarato — l'utente può cambiarlo o rifiutarlo).
- A due date raggiunta, quando l'utente lo chiede ("prepara il follow-up per X" o "cosa c'è in scadenza?"): genera la bozza di follow-up — DM breve (60-100 parole, cortese, un riferimento concreto alla candidatura, una domanda chiara sullo stato) o **bozza email in Gmail** (contratto `follow-up-draft`: bozza, MAI invio diretto). Dopo il follow-up: riga di log + proponi nuova due date a +7/+10 giorni o rimozione.
- Colloquio fissato: due date = data del colloquio, sezione `In corso`.

## Aggiornamento stato via email (il pezzo delicato)

SOLO su richiesta esplicita ("controlla le risposte", "novità?") — mai in autonomia.

1. **Recupero**: Gmail `search_threads` su una finestra recente (default: 7 giorni, dichiaralo; l'utente può allargarla). Cerca in modo mirato: per ogni task attivo, query con nome azienda e/o ruolo; più una passata generica su mittenti tipici di ATS/recruiting se i task attivi sono pochi. Leggi i thread candidati con il contenuto completo, non gli snippet.
2. **Mappatura email → candidatura** (fuzzy, a livelli):
   - *Match forte*: dominio o nome del mittente riconducibile all'azienda del task E il titolo del ruolo compare in subject/body → procedi con conferma leggera.
   - *Match medio*: solo l'azienda matcha, e c'è UN solo task attivo per quell'azienda → proponi l'associazione, chiedi conferma.
   - *Ambiguo*: l'azienda matcha ma ci sono PIÙ task per quell'azienda, oppure scrive un'agenzia/ATS il cui dominio non c'entra con l'azienda (caso frequente: `no-reply@ats-di-terzi.com`) → mostra l'email e chiedi a quale candidatura appartiene. Se l'utente la associa, annota l'associazione mittente→task nel log del task: le email successive dello stesso thread/mittente matcheranno da sole.
   - *Nessun match*: segnalala come "email orfana" e chiedi se riguarda una candidatura fuori tracker — NON forzare l'associazione al task più simile.
3. **Dedup email** (obbligatorio, prima di ogni azione): ogni email processata si registra come COMMENTO sul task: `email-processata: <gmail_message_id> — <classificazione> — <data>`. Prima di proporre un'azione, controlla i commenti del task: message id già presente → salta senza dire nulla (non è una novità).
4. **Classificazione** in quattro classi: `ack` (conferma ricezione candidatura) · `rejection` · `invito_colloquio` · `ping` (richiesta info/disponibilità/documenti). In dubbio tra due classi, mostra l'email e chiedi — un falso rejection che chiude un task è il danno peggiore che questa skill possa fare.
5. **Azione per classe** (sempre: proposta → conferma → esecuzione → log + commento dedup):
   - `ack` → riga di log; nessun cambio sezione.
   - `rejection` → proponi: label `@esito-rifiuto` + completamento task. Conferma esplicita SEMPRE, anche su match forte.
   - `invito_colloquio` → proponi: sezione `In corso` + due date alla data del colloquio se presente nella mail (se manca, chiedi) + eventuale bozza di risposta.
   - `ping` → mostra la richiesta e proponi una bozza di risposta (bozza Gmail, mai invio).
6. **Riepilogo finale**: cosa è stato aggiornato, cosa è in attesa di decisione, le orfane.

## Casi limite

- **Due candidature stessa azienda**: sempre chiedere, mai indovinare dal solo mittente.
- **Email su task già chiuso** (es. rejection dopo un ritiro): appendi la riga di log al task completato senza riaprirlo, e segnalalo all'utente.
- **Todoist irraggiungibile a metà operazione**: riporta cosa è stato fatto e cosa no, così l'utente non resta con uno stato a metà senza saperlo.
- **Volumi**: se i task attivi sono tanti (>15), fai il controllo email per gruppi e dillo, invece di degradare la qualità del matching.

## Cosa NON fare

- Non creare task senza richiesta esplicita (né dalla routine, né "già che ci sono").
- Non chiudere/spostare task su base email senza conferma.
- Non inviare mai email: solo bozze.
- Non toccare progetti Todoist diversi da `Job Hunter`.
- Non riscrivere il log eventi: solo append.
- Non processare due volte la stessa email (commento dedup prima di tutto).
