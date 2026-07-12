# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Cos'è questa repo (leggi prima di tutto)

Non è un'applicazione: è un **pacchetto di 7 skill per Claude** scritte in prosa (Markdown + schemi YAML, più un template HTML in `cv-tailoring/assets/`). Non c'è codice eseguibile, né build, né test, né dipendenze. L'"artefatto" è il testo delle `SKILL.md`: il "runtime" è Claude che le legge e le esegue in chat.

Conseguenze pratiche per chi lavora qui:

- Un **bug** è quasi sempre un'incoerenza tra file: un contratto dato citato in due modi diversi, un enum divergente, un rimando a una skill che non fa più quello che si dice faccia, una `description` di frontmatter che non intercetta il caso d'uso reale.
- Un **improvement** è un cambio di prosa che modifica il comportamento di Claude a runtime. Va valutato come si valuta un prompt, non come si valuta una funzione: il criterio è "un'istanza di Claude che legge SOLO questo file si comporta come previsto?".
- Non esiste modo di "eseguire" la modifica per verificarla. La verifica è **rileggere il file dalla prospettiva di chi non ha il contesto della conversazione** e controllare le coerenze cross-file elencate sotto.

Il documento di riferimento sul dominio è il [README.md](README.md) (architettura, requisiti, principi). Questo file copre come *modificarlo*.

## Perimetro

La repo contiene solo `skill-job-hunter/`. **Non** contiene:

- la routine `job-watch` (Claude Code Routine in cloud, repo separata) — le skill la descrivono e ne dipendono, ma non la definiscono qui;
- i dati utente (profili, valutazioni, log) — vivono su Google Drive, mai in git.

Se una richiesta riguarda il comportamento della routine (dedup, `state.json`, digest email), il codice non è qui: si può solo aggiornare il *contratto* che le skill le impongono (es. `source-log-schema.md`).

Le cartelle `docs/`, `.docs/`, `staging/`, `.github/` sono vuote (solo `.DS_Store`): non sono struttura, sono residui.

## Comandi

Non c'è build né test. Le sole operazioni reali:

```bash
# Impacchettare le skill per il caricamento su Claude
zip -r skill-job-hunter.zip skill-job-hunter/

# Controlli minimi dopo una modifica (non esiste una suite: questi sono i check manuali che contano)
python3 -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]" skill-job-hunter/*/references/*.yaml
grep -L '^name:' skill-job-hunter/*/SKILL.md          # frontmatter mancante → skill non caricabile
grep -rn '{{' skill-job-hunter/cv-tailoring/assets/   # i placeholder devono esistere solo nel template
```

A runtime le skill installate su Claude vivono sotto `/mnt/skills/user/<nome-skill>/` — è il path con cui una skill ne referenzia un'altra (es. `job-search-profile` legge lo schema di `agent-config`). Se cambi il nome di una cartella, quel riferimento si rompe silenziosamente.

## Architettura: chi dipende da chi

Il grafo delle dipendenze è la cosa che serve capire prima di toccare qualsiasi file. Le skill si invocano e si rimandano l'una all'altra per nome, e ogni rimando è un contratto.

```
agent-config (1.1) ── crea master-profile.yaml + search-profile.yaml su Drive
   │                  possiede lo SCHEMA CANONICO di entrambi
   ├─→ job-alert-config (1.2.1)  legge search-profile → istruzioni alert (testo, non file)
   └─→ routine job-watch (fuori repo)  legge search-profile → digest email

job-search-profile (1.2)  EDITOR del search-profile esistente; NON ha una copia dello schema:
   │                       legge quello di agent-config (fonte di verità unica)
   └─→ dopo ogni modifica di ruoli/location/seniority rimanda a job-alert-config

job-alert-tuner (1.2.2)  legge source-log.csv (che la routine dovrà scrivere) → metriche
                          possiede il contratto source-log-schema.md

role-fit (2.1)  legge master-profile + JD → YAML in job-hunting/role-fit/ su Drive
   ├─→ cv-tailoring (2.2)  RIUSA match/gaps del role-fit se esiste
   └─→ application-tracker (2.3)  solo su richiesta esplicita dell'utente
```

I tre storage e la loro divisione di responsabilità (Drive = knowledge base, git = solo `state.json`, Todoist = unica fonte di verità sullo stato-workflow) sono descritti nel README. **Non spostare dati tra questi tre luoghi senza una ragione esplicita**: la divisione non è estetica, deriva dal fatto che il connettore GitHub in chat è di sola lettura e che due fonti dello stesso stato divergono.

## Coerenze cross-file da verificare a ogni modifica

Questa è la checklist che rende efficienti i prompt di bugfix e analisi. Se tocchi la colonna di sinistra, controlla tutto quello a destra:

| Se modifichi… | Controlla anche |
|---|---|
| `agent-config/references/search-profile.schema.yaml` | l'intervista in `agent-config` passo 3, le mappature in `job-search-profile` §2, i campi consumati da `job-alert-config` (§Input), `mappa-campi-piattaforme.md`, `example-search-profile.yaml` |
| `agent-config/references/master-profile.schema.yaml` | l'intervista in `agent-config` passi 1-2, gli input di `role-fit` e `cv-tailoring` (entrambe leggono solo da qui) |
| l'enum `fonti.nome` (`indeed`/`linkedin_alert`/`indeed_alert`) | `source-log-schema.md` (colonna `fonte`, dichiarata "stessi valori di `search-profile.fonti.nome`"), `role-fit` (`meta.sorgente`, che aggiunge `manuale`), le label sorgente di `application-tracker` (`@indeed`/`@linkedin`/`@manuale` — **la mappatura non è 1:1: 4 valori di `sorgente` contro 3 label, e nessuna skill la definisce esplicitamente. È un seam reale**) |
| `role-fit-output.schema.yaml` (score/esito) | `cv-tailoring` §Input punto 3, `application-tracker` (descrizione del task rimanda al file role-fit), il passaggio `promosso_a_tracker` |
| il nome file dei role-fit (`<YYYY-MM-DD>-<azienda-slug>-<ruolo-slug>.yaml`, slug minuscolo/trattini) | è definito **due volte**: in `role-fit` §Output e nell'header di `role-fit-output.schema.yaml`; `application-tracker` lo referenzia nella descrizione del task |
| `source-log-schema.md` (colonne, enum `esito`) | le metriche di `job-alert-tuner` (usano `ricerca_id`, `annuncio_id`, `scartato_dedup`, `scartato_lingua`, `scartato_livello`, `non_lavorato_cap` per nome); `annuncio_id` è dichiarato "lo stesso identificatore usato in `state.json`" — contratto con la routine fuori repo |
| la chiusura di `job-alert-config` (richiesta di conferma "fatto, alert impostati") | `agent-config` passo 5: quella conferma è l'interfaccia che aspetta prima di attivare la routine (passo 6) — è dichiarato in entrambi i file |
| i percorsi su Drive (`job-hunting/...`) | tutte le skill: ogni path è ripetuto in prosa in più file, non c'è una costante |
| una `description` di frontmatter | le altre skill con trigger vicini — le description sono il **router**: `agent-config` vs `job-search-profile` (creare vs modificare) e `role-fit` vs `cv-tailoring` sono le due coppie che si confondono più facilmente |

## Invarianti di progetto (non "rimuoverli per semplificare")

Sono decisioni prese e motivate dentro le skill. Un'istanza futura che le trova "ridondanti" e le toglie sta rompendo il sistema, non ripulendolo:

- **Nessuno score numerico** in `role-fit`: l'enum ordinale (`forte`/`buono`/`parziale`/`debole`) esiste apposta per non invitare a filtri a soglia. La valutazione informa, non scarta — e lo score non viaggia mai da solo: ha senso solo accanto a match/gaps/considerazioni.
- **Nessuna assunzione silenziosa**: un campo non confermato dall'utente resta vuoto/null. Mai un default plausibile.
- **Nessuna scrittura senza conferma**: ogni skill mostra il contenuto prodotto (YAML, diff dei campi toccati, bozza dei contenuti del CV) e aspetta conferma esplicita prima di scrivere su Drive/Todoist o renderizzare. Il pattern è ripetuto in tutte le skill che scrivono.
- **Nessun campo fuori schema**: un campo nuovo è un'evoluzione dello schema e si decide a livello di progetto, non dentro un'intervista o una modifica — la skill si ferma e lo segnala.
- **Nessun task Todoist nasce da solo**: la promozione al tracker è sempre un'azione esplicita dell'utente.
- **Mai inviare email**: solo bozze Gmail (vale per follow-up, cover, risposte).
- **`cv-tailoring` non fabbrica**: seleziona, riordina, enfatizza ciò che è nel `master-profile`. Nient'altro.
- **Nessuna skill usa la memoria dell'account** per rispondere a domande che l'intervista deve porre: ogni skill deve funzionare identica per un utente di cui non si sa nulla.
- **Un solo schema canonico**: non creare copie "di comodo" di uno schema dentro un'altra skill.

## Vincoli d'ambiente (dichiarati nelle skill come fatti, non ipotesi)

Non "risolverli": sono vincoli strutturali. I primi due sono all'origine di due delle tre eccezioni manuali del progetto (la terza — promozione al tracker — è manuale per scelta, vedi README); gli altri due sono limiti di piattaforma che le skill già gestiscono.

- Il **fetch delle pagine LinkedIn è bloccato** (dichiarato "verificato" in `role-fit`): le JD LinkedIn si incollano a mano. Non proporre scraping o fetch.
- **LinkedIn/Indeed sono dietro login, senza API pubblica utilizzabile per gli alert**: `job-alert-config` produce istruzioni per un umano, per costruzione. Non generare finti file di configurazione.
- Il **connettore Drive non ha append**: chi scrive log fa read-modify-write (documentato in `source-log-schema.md`).
- Gli **alert delle piattaforme non filtrano** esclusioni/lingue: quei filtri li applica a valle la routine. Ogni testo che promette il contrario è un bug.

## Convenzioni di scrittura delle SKILL.md

- Lingua: **italiano**, coerente con tutto il resto. Vale per la prosa delle skill — gli artefatti per l'utente hanno una regola propria: cover letter e DM escono nella lingua della JD, salvo diversa richiesta (`cv-tailoring`).
- Frontmatter: `name` (= nome cartella) e `description` in terza persona, che elenca i **trigger reali** in linguaggio d'utente ("aggiungi Amsterdam", "che fit ho?") e i **trigger negativi** ("NON usare per…"). La description è ciò che decide se la skill viene invocata: è la parte più load-bearing del file.
- Struttura ricorrente: contesto/modulo in apertura → precondizioni/input → flusso → **Cosa NON fare** in chiusura (presente in tutte e 7; una sezione "Casi limite" esiste solo dove serve, es. `application-tracker`). La sezione finale non è decorativa: raccoglie i divieti con la loro motivazione e va estesa, non tagliata.
- Tono: imperativo verso Claude, con la *ragione* accanto al vincolo. I "perché" nel testo sono ciò che impedisce a un'istanza futura di razionalizzare un'eccezione.
- Ogni default numerico va **dichiarato come default** e reso confermabile dall'utente (48h finestra, 15 annunci/run, follow-up +7gg poi +7/+10gg, ~8-10 alert per piattaforma, finestra email 7gg, soglia resa unica <15%, soglia righe malformate del log ~20%).
- Verifica connettori CONCRETA: ogni skill verifica il connettore che le serve con `tool_search` prima di usarlo — mai fidarsi della sola dichiarazione dell'utente. Pattern presente in tutte le skill che toccano Drive/Gmail/Indeed/Todoist.
- Profilo mancante ≠ crearlo: se `master-profile`/`search-profile` non esiste su Drive, la skill rimanda ad `agent-config` e si ferma. Nessuna skill crea i profili fuori dall'onboarding (un profilo creato fuori intervista nascerebbe monco).
- Dato illeggibile → mostrare e chiedere, mai correggere inventando: YAML corrotto si mostra grezzo con l'errore (`job-search-profile`), righe CSV malformate si scartano e si contano (`job-alert-tuner`). Se il log devia dal contratto, ha deviato la routine: si segnala, non si adatta il contratto.
- Gestione fallimenti connettore: ogni skill che scrive deve mostrare in chat il contenuto prodotto se la scrittura fallisce, e riprendere dal passo interrotto (mai ricominciare l'intervista da capo). È un pattern ripetuto: rispettalo nelle skill nuove.

## Punti aperti (dichiarati, non da "correggere" di iniziativa)

- Il `source-log.csv` **non viene ancora scritto**: la strumentazione della routine è fuori dal perimetro di questa repo. `job-alert-tuner` gestisce l'assenza del file come caso previsto — non è un bug.
- Non è verificato che una **Claude Code Routine in cloud** veda lo stesso connettore Drive disponibile in chat. `agent-config` passo 6 lo dichiara all'utente come ipotesi da testare: non riscriverlo come fatto acquisito.
