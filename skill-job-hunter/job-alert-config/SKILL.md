---
name: job-alert-config
description: >-
  Genera le istruzioni passo-passo per impostare gli alert di lavoro su
  LinkedIn e Indeed a partire dal search-profile del sistema Job Hunter
  (job-hunting/search-profile.yaml su Google Drive). Usa SEMPRE questa
  skill quando: l'utente chiede "imposta gli alert", "come configuro gli
  alert LinkedIn/Indeed", "genera le ricerche salvate", "istruzioni per gli
  alert"; quando l'onboarding agent-config arriva al passo alert (passo 5);
  quando l'utente ha appena modificato il search-profile con
  job-search-profile e gli alert vanno riallineati. Produce istruzioni
  testuali da eseguire a mano (LinkedIn/Indeed sono dietro login: nessuna
  configurazione programmatica possibile), non file.
---

# job-alert-config

Modulo 1.2.1 del progetto Job Hunter. Trasforma il `search-profile` in **istruzioni operative** per impostare gli alert su LinkedIn e Indeed. L'impostazione resta manuale per l'utente: è una delle tre eccezioni manuali strutturalmente non eliminabili del progetto (servizi terzi dietro login, nessuna API pubblica utilizzabile) — non tentare scorciatoie tipo fetch/automazione delle pagine LinkedIn, non funzionano e non sono previste.

## Input

1. Leggi `job-hunting/search-profile.yaml` da Google Drive (verifica prima il connettore con `tool_search`). Se il file NON esiste: non inventare parametri — spiega che serve prima l'onboarding (`agent-config`) e fermati.
2. Campi usati: `ruoli_target` (titoli + sinonimi), `location_target` (aree + priorità + remoto/ibrido), `seniority`, `fonti` (quali piattaforme sono attive), `parametri_esecuzione.finestra_temporale_ore` (per la frequenza alert).
3. Se `ruoli_target` o `location_target` sono vuoti: fermati e dillo — un alert senza ruolo o senza location non è configurabile sensatamente.

## Derivazione degli alert (algoritmo)

1. **Base**: una combinazione ruolo × location = un potenziale alert. I `sinonimi` NON generano alert separati: entrano nella stessa query con OR dove la piattaforma lo consente (vedi `references/mappa-campi-piattaforme.md`), altrimenti si sceglie il solo `titolo_principale`.
2. **Priorità**: ordina per `location_target.priorita` (alta prima). Le location `accetta_remoto: true` generano anche la variante con filtro "Remoto" attivo dove la piattaforma la gestisce come filtro separato.
3. **Cap dichiarato**: proponi al massimo ~8-10 alert per piattaforma. Se le combinazioni superano il cap, mostra l'elenco completo ordinato per priorità e chiedi all'utente dove tagliare — non tagliare in silenzio. Troppi alert = digest rumoroso e email duplicate; il tuning a posteriori è mestiere di `job-alert-tuner` (1.2.2).
4. **Fonti disattive**: se in `fonti` una piattaforma è `attiva: false`, salta le sue istruzioni e dillo.

## Cosa gli alert NON possono fare (dichiaralo sempre all'utente)

Gli alert di LinkedIn/Indeed non applicano: esclusioni di titoli (Head of/Director/...), esclusioni di tipo contratto (stage), filtri di lingua del corpo annuncio, filtri su requisiti linguistici obbligatori. Questi filtri vengono applicati A VALLE dalla routine job-watch, che legge gli alert via Gmail e scarta secondo il `search-profile`. Dillo esplicitamente nelle istruzioni: l'utente NON deve aspettarsi alert già puliti, deve aspettarsi un digest già pulito.

## Vincoli critici di consegna (senza questi la routine è cieca)

Includi SEMPRE, in testa alle istruzioni, questi due punti:

1. **Consegna via email ATTIVA** verso la casella Gmail collegata al sistema: la routine legge gli alert da Gmail (`jobs-noreply@linkedin.com`, `alert@indeed.com`). Un alert solo-notifica-app è invisibile al sistema.
2. **Frequenza giornaliera** (o la più frequente disponibile): la routine lavora su una finestra di `finestra_temporale_ore` ore (default 48) — alert settimanali arriverebbero già vecchi.

## Output (in chat, non file)

Produci una checklist numerata, divisa per piattaforma, nello stile:

```
LINKEDIN — alert 1 di N
1. Vai su linkedin.com/jobs e cerca: <query>
2. Località: <location>  [+ filtro Remoto: sì/no]
3. Filtri consigliati: Livello esperienza = <mappatura seniority>; Data pubblicazione = Ultime 24 ore
4. Attiva "Crea avviso di offerte" per questa ricerca
5. Nelle impostazioni dell'avviso: frequenza Giornaliera, canale Email
```

Le mappature esatte campo-per-campo (nomi dei filtri, sintassi query, mappatura seniority→livelli piattaforma) sono in `references/mappa-campi-piattaforme.md`: leggilo prima di generare le istruzioni. I nomi dei menu delle piattaforme cambiano nel tempo: se l'utente segnala che un campo indicato non esiste più, adatta l'istruzione al concetto (es. "il filtro che limita per data di pubblicazione") invece di insistere sul nome esatto.

Chiudi SEMPRE con: l'elenco riassuntivo degli alert da creare (per spuntarli), e la richiesta di conferma esplicita "fatto, alert impostati" — è l'interfaccia che `agent-config` (passo 5) aspetta prima di procedere all'attivazione della routine. Se invece sei stato invocato dopo una modifica del profilo (via `job-search-profile`), ricorda anche di ELIMINARE o aggiornare gli alert vecchi non più coerenti, non solo aggiungere i nuovi.

## Cosa NON fare

- Non generare config "programmatica" o finti file di configurazione: sono istruzioni per un umano, per costruzione.
- Non inventare valori mancanti dal profilo (es. seniority assente → ometti il filtro e dillo, non scegliere tu un livello).
- Non promettere che gli alert filtreranno esclusioni/lingue (vedi sopra).
- Non dare per impostati gli alert senza conferma esplicita dell'utente.
