---
name: agent-config
description: >-
  Onboarding guidato per il sistema Job Hunter. Usa questa skill quando un
  nuovo utente vuole attivare il sistema di ricerca lavoro autonoma:
  ingerisce il CV, fa domande su vincoli/preferenze (trasferimento, remoto,
  retribuzione, preavviso, diritto al lavoro, lingue), produce
  master-profile.yaml e search-profile.yaml nella cartella job-hunting su
  Google Drive, genera e attiva la config della routine job-watch. Trigger
  su qualsiasi richiesta di avviare/configurare/inizializzare il sistema o
  la ricerca lavoro, anche breve o generica — es. "inizializza il
  progetto", "salva queste skill e configurami", "iniziamo a fare
  job-hunting", "configurami il sistema". Non richiedere che l'utente
  nomini "agent-config" o "onboarding": il segnale è l'intento di avviare
  il processo. NON usarla per modificare un search-profile già esistente
  ("aggiorna il mio profilo di ricerca", "cambia i ruoli/le location"):
  per quello esiste la skill job-search-profile.
---

# agent-config

Orchestratore conversazionale che porta un utente da zero a sistema attivo (Modulo 1.1 del progetto Job Hunter). Produce due artefatti nella cartella `job-hunting` su Google Drive (`master-profile.yaml`, `search-profile.yaml`) e passa la palla a `job-alert-config` (1.2.1) per la config degli alert.

**Nota architetturale, da non dimenticare in futuri aggiornamenti**: l'integrazione GitHub di base disponibile in chat (quella in Impostazioni → Connettori) è di sola lettura/allegato file, non espone tool di scrittura/commit. Per questo TUTTA la knowledge base (profili, `source-log`, `role-fit-output` e ogni altro output dello Studio — decisione definitiva del progetto, non un punto aperto) vive su Google Drive, che ha un connettore con scrittura reale utilizzabile da qualsiasi chat. Git resta necessario solo per la routine stessa e ospita esclusivamente `state.json` di dedup (Claude Code ha accesso git nativo, indipendente dai connettori di chat) — vedi Precondizioni, punto 5.

Non eseguire questa skill silenziosamente: è interattiva per costruzione. Ogni sezione sotto corrisponde a un blocco di domande da fare in chat, una alla volta o in piccoli gruppi coerenti — mai tutte insieme in un unico wall of text.

## Passo 0 — Messaggio di apertura fisso

Prima di qualunque verifica tecnica, invia sempre questo messaggio (adattalo minimamente se necessario, ma mantieni struttura, contenuto e tono):

---

Sciaaaaaamn! 🎉 Iniziamo la configurazione del sistema Job Hunter.

Ti spiego prima cosa serve e cosa succederà, così puoi preparare tutto senza interruzioni a metà strada :)

**Cosa ti servirà collegato prima di iniziare:**
1. **Google Drive** 📁 — qui salveremo il tuo profilo di ricerca (esperienze, competenze, criteri di ricerca).
2. **Gmail** 📧 — necessario alla routine di ricerca automatica per leggere gli alert LinkedIn/Indeed, e per il tracker più avanti.
3. **Indeed** 🔍 — necessario alla routine per cercare annunci.
4. **Todoist** ✅ — necessario più avanti per tracciare le candidature.

Se non li hai già collegati, puoi farlo da Impostazioni → Connettori. Te li verifico comunque uno per uno tra un attimo.

**Un requisito che ti servirà solo alla fine, ma è meglio saperlo da subito:** un account GitHub con una repository vuota, dedicata a ospitare la routine di ricerca automatica (non serve per il tuo profilo, solo per farla girare in autonomia). Se non hai già un account GitHub, puoi crearne uno gratuito quando arriviamo a quel punto — non blocca l'inizio.

**Cosa ti chiederò:**
- Il tuo CV 📄 (in qualsiasi formato che riesci a caricare qui in chat)
- Alcune domande su vincoli e preferenze: disponibilità a trasferirti o lavorare da remoto, retribuzione attuale e aspettativa, preavviso, diritto al lavoro nei paesi che ti interessano, lingue
- Conferma su ruoli, location e criteri di esclusione per la ricerca

**Cosa succederà dopo:**
1. Scrivo il tuo profilo su Google Drive
2. Ti do le istruzioni per impostare gli alert LinkedIn/Indeed
3. Attiviamo insieme la routine di ricerca automatica — questo passaggio si fa dall'**app desktop di Claude Code** 🖥️ (niente terminale, solo click)

Da lì in poi il sistema lavora per te: ricevi un digest via email con gli annunci trovati, e usi la chat per valutarli, preparare CV su misura e tenere traccia delle candidature.

Cominciamo dal CV — puoi caricarlo ora? 😊

---

Dopo questo messaggio, procedi comunque con la verifica tecnica concreta delle Precondizioni sotto — il messaggio dichiara i requisiti all'utente, ma non sostituisce il controllo reale via `tool_search`. Non fidarti della sola dichiarazione dell'utente "ce li ho tutti collegati": verificalo.

## Precondizioni

Prima di iniziare l'intervista, verifica CONCRETAMENTE (non a parole) i requisiti del sistema, anche quelli che questa skill non usa direttamente — è il primo punto di contatto interattivo, il posto giusto per bloccare l'intero onboarding se manca qualcosa:

1. **Google Drive** — usa `tool_search` con query tipo "Google Drive" per verificare che il connettore sia caricabile e utilizzabile in scrittura. È qui che vive tutta la knowledge base di questa skill: la cartella `job-hunting` con `master-profile.yaml`/`search-profile.yaml` (passo 4).
2. **Gmail** — `tool_search` query "Gmail". Necessario alla routine (lettura alert) e a valle per follow-up/bozze.
3. **Indeed** — `tool_search` query "Indeed jobs". Necessario alla routine per la ricerca.
4. **Todoist** — `tool_search` query "Todoist tasks". Non usato da questa skill, ma necessario al tracker (2.3): se manca, l'utente arriverebbe operativo su sourcing/studio senza poter usare il tracker.
5. **Repo GitHub vuota, dedicata alla sola routine** — NON verificabile da questa skill con un tool: l'integrazione GitHub di base in chat è di sola lettura/allegato, non scrive. Chiedi esplicitamente all'utente se ha già un account GitHub e una repository vuota dedicata a ospitare la routine Job Hunter; se non ce l'ha, spiega che gli servirà (repo vuota, nome a sua scelta) prima di attivare la routine al passo 6, e fatti dare nome/URL quando l'avrà creata. Questo prerequisito NON blocca i passi 1-4 (che scrivono solo su Drive): blocca solo l'avvio del passo 6.
6. **CV disponibile** — chiedilo come primo passo se non è già stato allegato in chat.

Per Drive/Gmail/Indeed/Todoist mancanti: fermati, dì per nome quale manca e perché serve, e chiedi di collegarlo prima di proseguire. Non generare un profilo incompleto "per ora" e non saltare la verifica assumendo che l'utente li abbia già collegati solo perché ha caricato il pacchetto di skill (il pacchetto e i connettori sono due cose diverse, vedi customer journey passi 2 e 3).

**Connettore che fallisce a metà flusso**: se un tool call fallisce durante l'intervista o alla scrittura (connettore scaduto, permesso revocato), non perdere il lavoro fatto: mostra subito in chat lo stato completo dei dati raccolti fino a quel punto (lo YAML parziale, se già formato), spiega quale connettore ha fallito e come ricollegarlo, e riprendi ESATTAMENTE dal passo interrotto — non ricominciare l'intervista da capo. Vale in particolare per la scrittura su Drive al passo 4: se fallisce, lo YAML completo va comunque mostrato in chat così l'utente non perde nulla, e la scrittura si ritenta dopo la riconnessione.

## Schema di riferimento

Gli schemi dato-agnostici sono in:
- `references/master-profile.schema.yaml`
- `references/search-profile.schema.yaml`

Leggili prima di condurre l'intervista: ogni campo dello schema è una domanda potenziale. Non inventare campi che non sono nello schema; se durante l'intervista emerge un dato utile che non ha posto nello schema, segnalalo all'utente invece di infilarlo a forza da qualche parte — potrebbe voler dire che lo schema va aggiornato (torna al progetto, non decidere da solo).

## Flusso

### 1. Ingestione CV → bozza master-profile

- Chiedi il CV se non presente.
- Estrai dal CV tutto ciò che mappa direttamente sui campi di `master-profile.schema.yaml`: esperienze, progetti, skill tecniche, lingue, formazione, certificazioni.
- Presenta la bozza risultante e chiedi conferma/correzioni prima di proseguire. Un CV può essere ambiguo o incompleto (date mancanti, stack non esplicito) — segnala i buchi invece di indovinare.
- Non chiedere ancora i campi che il CV non può contenere (retribuzione, preavviso, disponibilità): vengono dopo, sono domande dirette non deducibili da un documento.

### 2. Domande su vincoli e preferenze

Copri, in gruppi tematici separati (non tutto insieme):

**Mobilità**
- Disponibilità trasferimento (sì / no / solo alcune aree — e quali)
- Disponibilità remoto (full remote / ibrido / solo sede / indifferente)

**Economico**
- Retribuzione attuale (valore, lordo/netto, periodicità)
- Aspettativa (range, stesse unità)
- Eventuale flessibilità/note

**Contrattuale**
- Preavviso (durata, eventuali vincoli particolari)

**Legale**
- Cittadinanza/e
- Diritto al lavoro per le aree in cui l'utente vuole cercare (non dare per scontato che coincida con la cittadinanza — un permesso di soggiorno, un passaporto UE aggiuntivo, ecc. possono cambiare la risposta)

**Lingue**
- Livello per ciascuna lingua rilevante, e contesto d'uso (lavorativo quotidiano vs solo letto) — serve sia a `master-profile` sia, a valle, a filtrare gli annunci in `search-profile`

Ogni gruppo di domande scrive direttamente nei campi corrispondenti dello schema. Se l'utente salta una domanda o risponde "non so", lascia il campo vuoto/null nello YAML — non riempirlo con un default plausibile.

### 3. Produzione search-profile

A questo punto hai i dati per derivare (non indovinare — derivare, con conferma esplicita) `search-profile`:

- **Seniority**: dagli anni di esperienza e dal tipo di ruoli avuti in `master-profile.esperienze`, proponi un livello (`junior/medio/senior/...`) e chiedi conferma — non scriverlo senza validazione, perché la percezione di seniority dell'utente può differire dal dato grezzo.
- **Ruoli target**: chiedi esplicitamente quali titoli cercare (non dedurli automaticamente dal ruolo attuale — un utente potrebbe voler cambiare ruolo).
- **Location target**: chiedi le aree geografiche di interesse, incrociando con `disponibilita_remoto`/`disponibilita_trasferimento` già raccolti.
- **Esclusioni**: chiedi se ci sono titoli o tipi di contratto da escludere sempre (es. ruoli manageriali, stage) — proponi i default tipici (Head of/Director/VP/C-level, Internship/Traineeship/Stage) ma fai confermare, non assumerli silenziosamente.
- **Settori**: target ed esclusi, se l'utente ha preferenze.
- **Lingue annuncio**: da `master-profile.lingue`, proponi quali lingue sono accettabili per il corpo di un annuncio e quali lo scartano se prevalenti/obbligatorie — richiede una domanda esplicita, perché "so l'inglese" non implica automaticamente "accetto annunci il cui corpo è in inglese ma richiede altra lingua come requisito".
- **Parametri esecuzione**: finestra temporale e max annunci per esecuzione — proponi i default usati nella routine esistente (48 ore, 15 annunci) solo come punto di partenza dichiarato, fai confermare.

### 4. Scrittura su Google Drive

- Verifica se esiste già una cartella `job-hunting` nel Drive dell'utente (cerca prima di crearne una nuova, per non duplicarla).
- Scrivi al suo interno `master-profile.yaml` e `search-profile.yaml` usando il tool di scrittura Drive (non un semplice allegato/lettura).
- Mostra il contenuto finale all'utente prima di scriverlo. Non scrivere silenziosamente — è un dato che alimenta tutto il resto del sistema (routine, role-fit, cv-tailoring): un errore qui si propaga ovunque.

### 5. Passaggio a job-alert-config (alert LinkedIn/Indeed)

Segui la sequenza del customer journey (punto 4, a→b→c): dopo la scrittura di `search-profile.yaml`, il passo successivo è la configurazione degli alert, PRIMA di attivare la routine — la routine legge alert LinkedIn via Gmail, quindi attivarla senza alert configurati la farebbe partire su una fonte vuota.

- Richiama esplicitamente la skill `job-alert-config` (1.2.1), passandole `search-profile.yaml`. Non duplicarne la logica qui: quella skill produce le istruzioni su come impostare i campi degli alert su LinkedIn/Indeed (sono dietro login, quindi istruzioni per l'utente, non config programmatica).
- Aspetta conferma esplicita dall'utente che gli alert sono stati effettivamente impostati prima di passare al punto 6. Non proseguire per inerzia.

### 6. Verifica e attivazione della routine

Architettura a due binari, per costruzione:
- **Repo GitHub** (raccolta al prerequisito 5): ospita SOLO `state.json` (dedup della routine, branch `claude/job-watch-state`) — nient'altro: niente profili, niente `source-log`, niente output dello Studio. Tutta la conoscenza vive su Google Drive.
- **Google Drive, cartella `job-hunting`**: ospita `master-profile.yaml`/`search-profile.yaml`, scritti al passo 4.

Compiti di questo passo:
- Recupera/conferma il nome o URL della repo GitHub raccolto al prerequisito 5. Se non è ancora stato creato, fermati qui e aspetta che l'utente lo faccia — non generare istruzioni di attivazione routine senza una repo reale.
- **Punto aperto da segnalare esplicitamente all'utente, non da assumere risolto**: la routine (Claude Code Routine in cloud) dovrà leggere `master-profile.yaml`/`search-profile.yaml` da Drive a runtime. Non è stato verificato in questo progetto se Claude Code Routine ha accesso allo stesso connettore Drive disponibile in chat normale — è un'ipotesi da testare, non un fatto. Dillo all'utente prima di procedere e suggerisci di verificarlo con un test mirato (analogo a quello già fatto per git) prima di fare affidamento sulla routine in produzione.
- Guida l'attivazione della routine come Claude Code Routine in cloud, dall'app desktop Claude Code (nessun terminale). Spiega i passaggi a livello di cosa cliccare, non assumere che l'utente sappia già come si crea una routine.
- Mostra, a scopo di verifica, i valori scritti su Drive al passo 4, così l'utente capisce cosa la routine dovrà leggere.

### 7. Utente operativo

Solo a questo punto l'onboarding è completo: profilo scritto, alert impostati, routine attiva. Dillo esplicitamente all'utente e ricorda dove vivono i due canali con cui interagirà da qui in poi (digest via email, chat per lo Studio). Ricorda anche che i criteri di ricerca si modificano in futuro con la skill `job-search-profile` (senza rifare l'onboarding) e che, dopo ogni modifica, gli alert LinkedIn/Indeed vanno riallineati con `job-alert-config`.

## Cosa NON fare

- Non riempire mai un campo con un valore plausibile ma non confermato dall'utente ("assunzione silenziosa").
- Non usare dati di memoria dell'account per rispondere a domande che l'intervista dovrebbe porre all'utente attivo in quel momento — la skill deve funzionare identica per un utente di cui non sai nulla.
- Non saltare la conferma finale prima della scrittura su Drive.
- Non inventare campi fuori schema: se serve un campo nuovo, fermati e segnalalo invece di forzarlo in un campo esistente.
