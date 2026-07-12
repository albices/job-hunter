# Job Hunter

Pacchetto di **7 skill per Claude** che compongono un sistema di ricerca lavoro semi-autonoma: dall'onboarding (CV → profilo) al sourcing degli annunci, dalla valutazione del fit alla generazione dei materiali di candidatura, fino al tracking delle risposte.

Questa repo contiene **solo le skill**. La routine automatica che pesca gli annunci (`job-watch`) vive in una repo separata, e tutta la knowledge base dell'utente vive su Google Drive — vedi [Architettura](#architettura).

---

## Il sistema in due fasi

**Fase 1 — Sourcing** (arriva un digest di annunci via email, senza che l'utente faccia nulla)

| Skill | Modulo | Cosa fa |
|---|---|---|
| [`agent-config`](skill-job-hunter/agent-config/SKILL.md) | 1.1 | Onboarding guidato: ingerisce il CV, intervista su vincoli e preferenze, scrive `master-profile.yaml` + `search-profile.yaml` su Drive, attiva la routine |
| [`job-search-profile`](skill-job-hunter/job-search-profile/SKILL.md) | 1.2 | Editor del `search-profile` **dopo** l'onboarding (ruoli, location, esclusioni, lingue) |
| [`job-alert-config`](skill-job-hunter/job-alert-config/SKILL.md) | 1.2.1 | Genera le istruzioni passo-passo per creare gli alert LinkedIn/Indeed a partire dal `search-profile` |
| [`job-alert-tuner`](skill-job-hunter/job-alert-tuner/SKILL.md) | 1.2.2 | Metriche sul `source-log`: overlap tra ricerche, resa unica, tasso di annunci fuori scope |

**Fase 2 — Studio** (l'utente lavora gli annunci in chat)

| Skill | Modulo | Cosa fa |
|---|---|---|
| [`role-fit`](skill-job-hunter/role-fit/SKILL.md) | 2.1 | Valuta il fit tra una JD e il profilo; salva l'esito strutturato su Drive |
| [`cv-tailoring`](skill-job-hunter/cv-tailoring/SKILL.md) | 2.2 | CV su misura in PDF + cover letter + DM al recruiter, tutti dal `master-profile` |
| [`application-tracker`](skill-job-hunter/application-tracker/SKILL.md) | 2.3 | Pipeline candidature su Todoist, follow-up e aggiornamento stati dalle email |

---

## Architettura

Tre luoghi, ognuno con un ruolo preciso:

- **Google Drive**, cartella `job-hunting/` — **tutta** la knowledge base: `master-profile.yaml`, `search-profile.yaml`, `role-fit/*.yaml`, `source-log.csv`, eventuali CV generati. Vive qui perché il connettore Drive ha scrittura reale utilizzabile da qualsiasi chat Claude, mentre l'integrazione GitHub in chat è di sola lettura/allegato: da qui la scelta.
- **GitHub** — ospita la routine `job-watch` e **solo** `state.json` (dedup degli annunci già visti, branch `claude/job-watch-state`). Nessun profilo, nessun output.
- **Todoist**, progetto `Job Hunter` — l'unica fonte di verità sullo **stato-workflow** delle candidature. Il `role-fit-output` su Drive si ferma a `promosso_a_tracker` e non replica lo stato: due fonti dello stesso stato divergerebbero.

### Contratti dato

Gli schemi sono dato-agnostici e versionati insieme alle skill:

- [`master-profile.schema.yaml`](skill-job-hunter/agent-config/references/master-profile.schema.yaml) — chi è l'utente (esperienze, skill, retribuzione, vincoli, strato AI-adoption)
- [`search-profile.schema.yaml`](skill-job-hunter/agent-config/references/search-profile.schema.yaml) — cosa cercare (ruoli, location, esclusioni, lingue annuncio, fonti)
- [`role-fit-output.schema.yaml`](skill-job-hunter/role-fit/references/role-fit-output.schema.yaml) — esito di una valutazione (score ordinale, gap pesati, esito)
- [`source-log-schema.md`](skill-job-hunter/job-alert-tuner/references/source-log-schema.md) — il log per-riga che la routine deve scrivere perché il tuning sia calcolabile

Lo schema canonico ha **una sola** copia: `job-search-profile` legge quello di `agent-config`, non ne tiene una propria.

---

## Requisiti

Connettori Claude da collegare (Impostazioni → Connettori):

- **Google Drive** (in scrittura) — knowledge base
- **Gmail** — la routine legge gli alert LinkedIn/Indeed; il tracker legge le risposte e vi crea le bozze (follow-up, risposte — mai invii)
- **Indeed** — ricerca diretta degli annunci
- **Todoist** — tracker candidature

Serve inoltre un **account GitHub con una repo vuota dedicata** alla routine (non serve per il profilo: solo per far girare `job-watch` in autonomia). L'attivazione della routine si fa dall'**app desktop di Claude Code**, senza terminale.

## Installazione

Le skill sono un pacchetto da caricare su Claude:

```bash
zip -r skill-job-hunter.zip skill-job-hunter/
```

e poi caricarlo tra le Skill del proprio account Claude. Una volta installate, l'onboarding parte da solo: basta scrivere in chat *"inizializza il sistema di job hunting"* — `agent-config` fa il resto (CV, intervista, scrittura su Drive, alert, routine).

In alternativa, per usarle in Claude Code, copia le cartelle sotto `.claude/skills/`.

---

## Principi di progetto

Sono vincoli espliciti scritti dentro ogni skill, non stile:

- **La valutazione informa, non scarta.** Nessun punteggio numerico e nessun filtro binario: `role-fit` usa un enum ordinale (`forte`/`buono`/`parziale`/`debole`) proprio per non invitare a filtri a soglia. Lo score non viaggia mai da solo: ha senso solo accanto a match, gap pesati e considerazioni. La decisione di candidarsi resta sempre dell'utente.
- **Nessuna assunzione silenziosa.** Un campo che l'utente non ha confermato resta vuoto: mai riempito con un default plausibile.
- **Nessuna scrittura silenziosa.** Ogni skill mostra il contenuto prodotto (YAML, diff, bozza dei contenuti) e chiede conferma prima di scrivere su Drive o Todoist; se la scrittura fallisce a metà, il contenuto si mostra comunque in chat e si riprende dal passo interrotto.
- **Nessun task nasce da solo.** La promozione al tracker è sempre un'azione esplicita dell'utente: la routine non scrive su Todoist.
- **Mai inviare email.** Follow-up, cover e risposte si consegnano come **bozze** Gmail.
- **Il tailoring non fabbrica.** `cv-tailoring` seleziona, riordina ed enfatizza ciò che è nel `master-profile` — non aggiunge esperienze né competenze.

### Le tre eccezioni manuali (strutturali, non lacune)

1. **Alert LinkedIn/Indeed** — le piattaforme sono dietro login e senza API utilizzabile: `job-alert-config` produce istruzioni per un umano, non configurazione programmatica.
2. **JD di LinkedIn** — il fetch delle pagine è bloccato: il testo va incollato a mano.
3. **Promozione al tracker** — manuale per scelta, non per limite tecnico (vedi sopra).

Gli alert delle piattaforme **non** applicano le esclusioni (titoli manageriali, stage, lingua dell'annuncio): quei filtri li applica a valle la routine leggendo il `search-profile`. L'utente non deve aspettarsi alert puliti, ma un digest pulito.

---

## Stato

Le 7 skill sono complete. Restano aperti due punti, dichiarati nelle skill stesse:

- **Strumentazione del `source-log`**: il contratto è definito, ma la routine non lo scrive ancora — finché non lo fa, `job-alert-tuner` non ha metriche da calcolare (lo gestisce come caso previsto, senza errori).
- **Accesso a Drive dalla routine cloud**: non è stato verificato che una Claude Code Routine in cloud veda lo stesso connettore Drive disponibile in chat. È un'ipotesi da testare prima di fare affidamento sulla routine in produzione.
