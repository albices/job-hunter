# source-log — contratto dato (definito da job-alert-tuner, modulo 1.2.2)

Questo file È il contratto che la strumentazione della routine `job-watch` dovrà rispettare quando inizierà a scrivere il `source-log`. La strumentazione della routine è fuori dal perimetro dello Studio: finché non viene fatta, il file non esiste e `job-alert-tuner` lo gestisce come caso previsto (vedi SKILL.md).

## Dove vive

- **Google Drive**, cartella `job-hunting`, file `source-log.csv` (NON su git: su git vive solo `state.json` della routine).
- Scritto in **append** dalla routine a ogni esecuzione (il connettore Drive non ha un vero append: la routine legge il file, accoda le righe della run corrente, riscrive — a suo carico).
- Letto da `job-alert-tuner` (1.2.2).

## Formato

CSV, UTF-8, separatore virgola, quoting standard (campi con virgole tra doppi apici), prima riga = header. Scelto CSV e non YAML perché è un log per-riga che cresce nel tempo: ispezionabile a occhio su Drive, parsabile banalmente con pandas nel code tool.

## Semantica: una riga = un annuncio osservato da una ricerca in una esecuzione

Se lo stesso annuncio è restituito da 3 ricerche diverse nella stessa run → 3 righe (è esattamente questo che rende calcolabile l'overlap). Include anche gli annunci poi scartati o già visti: il log fotografa cosa PORTA ogni ricerca, non cosa sopravvive alla pipeline.

## Colonne

| Colonna | Obbligatoria | Tipo | Contenuto |
|---|---|---|---|
| `run_id` | sì | string | Timestamp ISO 8601 dell'esecuzione (es. `2026-07-05T06:00:00Z`). Identico per tutte le righe della stessa run. |
| `fonte` | sì | enum | `indeed` \| `linkedin_alert` \| `indeed_alert` (stessi valori di `search-profile.fonti.nome`). |
| `ricerca_id` | sì | string | Identificatore STABILE della ricerca che ha prodotto la riga. Convenzione: per la ricerca diretta Indeed `indeed:<titolo>:<location>` (es. `indeed:data engineer:milano`, tutto minuscolo); per gli alert email il nome/subject-base dell'alert (es. `linkedin_alert:data engineer amsterdam`). La stabilità nel tempo è ciò che rende confrontabili le run: la routine non deve cambiare formato tra una run e l'altra. |
| `annuncio_id` | sì | string | ID/URL canonico dell'annuncio — LO STESSO identificatore usato in `state.json` per il dedup. È la chiave che permette di calcolare l'overlap. |
| `esito` | sì | enum | Esito pipeline per questa run: `incluso_principale` \| `incluso_da_verificare` \| `scartato_lingua` \| `scartato_livello` (Head of/Director/stage) \| `scartato_dedup` (già in state.json) \| `non_lavorato_cap` (tagliato dal cap max annunci). |
| `titolo` | no | string | Titolo annuncio, per leggibilità/debug. |
| `azienda` | no | string | Azienda, per leggibilità/debug. |
| `location` | no | string | Location dichiarata dall'annuncio. |

## Esempio

```csv
run_id,fonte,ricerca_id,annuncio_id,esito,titolo,azienda,location
2026-07-05T06:00:00Z,indeed,indeed:data engineer:milano,https://it.indeed.com/viewjob?jk=abc123,incluso_principale,Data Engineer,Azienda X,Milano
2026-07-05T06:00:00Z,indeed,indeed:bi developer:milano,https://it.indeed.com/viewjob?jk=abc123,scartato_dedup,Data Engineer,Azienda X,Milano
2026-07-05T06:00:00Z,linkedin_alert,linkedin_alert:etl amsterdam,https://www.linkedin.com/jobs/view/999,incluso_da_verificare,ETL Developer,Azienda Y,Amsterdam
```

(La seconda riga mostra il caso overlap: stesso `annuncio_id` da due ricerche; per la seconda risulta `scartato_dedup` perché già processato.)

## Note per la strumentazione della routine (task aperto)

- Log da scrivere nello stesso turno del commit di `state.json` (punto 8 della routine), per non avere run loggate a metà.
- Se la scrittura del source-log fallisce ma il resto della run è andato: NON bloccare digest/stato — il log è telemetria, la pipeline è il prodotto. Segnalare l'anomalia nel digest.
- `scartato_dedup` è attribuibile per ricerca solo se il dedup avviene DOPO la raccolta per-ricerca (com'è oggi al punto 3 della routine): la routine sa quale ricerca ha riportato l'ID già visto.
