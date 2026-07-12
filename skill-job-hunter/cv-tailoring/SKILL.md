---
name: cv-tailoring
description: >-
  Genera CV su misura in PDF, cover letter e messaggio diretto al recruiter
  a partire dal master-profile del sistema Job Hunter e da una job
  description. Usa SEMPRE questa skill quando l'utente dice: "fammi il CV",
  "fammi il CV per questa posizione", "tara/adatta il CV alla JD", "scrivi
  la cover letter", "preparami il messaggio per il recruiter",
  "candidiamoci a X", "prepara i materiali per la candidatura" — anche se
  chiede solo uno dei tre artefatti. Il CV esce sempre come PDF renderizzato
  server-side nel code tool; i dati vengono SOLO dal master-profile su
  Google Drive, mai inventati.
---

# cv-tailoring

Modulo 2.2 del progetto Job Hunter. Produce fino a tre artefatti coerenti tra loro per una candidatura: **CV in PDF**, **cover letter**, **DM al recruiter**. Stesso posizionamento su tutti e tre, lunghezza diversa per canale. Se l'utente ne chiede uno solo, produci quello — ma tienili concettualmente allineati se poi chiede gli altri.

## Regola d'oro (non negoziabile)

Il contenuto viene ESCLUSIVAMENTE dal `master-profile`. Il tailoring è **selezione, riordino ed enfasi** — mai fabbricazione: niente esperienze gonfiate, competenze aggiunte, date ritoccate, risultati inventati. I gap rispetto alla JD non si nascondono con vaghezza: si gestiscono con onestà nel posizionamento (nella cover si può argomentare l'affinità; nel CV semplicemente non si mente). Se il master-profile non basta per una sezione che la JD renderebbe importante, dillo all'utente: la soluzione è aggiornare il profilo (via `agent-config` o correzione su Drive), non improvvisare.

## Input

1. **master-profile** — `job-hunting/master-profile.yaml` da Google Drive (verifica connettore con `tool_search`). Se manca → onboarding non fatto → rimanda ad `agent-config` e fermati.
2. **JD** — stesse regole di intake di `role-fit`: LinkedIn sempre incollata a mano, Indeed via connettore/alert Gmail, altrimenti testo incollato. Senza JD chiedi: "per quale posizione?" (un CV "generico" è possibile solo se l'utente lo chiede esplicitamente — in quel caso salta la parte di tailoring e usa il profilo completo).
3. **role-fit-output, se esiste** — cerca in `job-hunting/role-fit/` una valutazione per la stessa azienda+ruolo. Se c'è, RIUSALA: i `match` diventano i punti da enfatizzare, i `gaps` con relativo peso guidano cosa argomentare in cover e cosa non promettere. Se non c'è, non è bloccante: puoi proporre di fare prima un `role-fit` (utile ma non obbligatorio) o procedere direttamente.

## Flusso

### 1. Posizionamento (una frase, prima di tutto)

Formula in una frase chi è l'utente PER QUESTA posizione (es. "profilo dati con N anni su <stack affine a quello richiesto>, forte su <requisito core della JD>"). È il filo che tiene insieme CV, cover e DM. Mostralo all'utente e fallo confermare prima di costruirci sopra: se il posizionamento è sbagliato, tutto il resto lo sarà.

### 2. Selezione contenuti per il CV

- **Esperienze**: tutte in ordine cronologico inverso (i buchi insospettiscono più dei ruoli poco affini), ma bullet ricalibrati: per le esperienze affini alla JD 3-4 bullet con i `risultati_quantificabili` in testa; per le altre 1-2 bullet essenziali.
- **Skill**: raggruppate e ordinate per rilevanza rispetto alla JD; le skill richieste dalla JD e presenti nel profilo vanno visibili subito. Le skill richieste e ASSENTI non compaiono (regola d'oro). Includi lo strato AI-adoption se il profilo lo valorizza e la JD/l'azienda lo rende rilevante.
- **Progetti/certificazioni**: solo se rilevanti per la JD, altrimenti la sezione si omette.
- Target: 1 pagina fino a ~6-8 anni di esperienza, massimo 2.

### 3. Bozza in chat → conferma

Mostra la bozza dei CONTENUTI (posizionamento, bullet riscritti, ordinamento) in chat prima del rendering. È il punto dove l'utente corregge enfasi e formulazioni. Non renderizzare prima della conferma.

### 4. Render PDF (pipeline adattiva, MAI hardcodata)

Il contratto è: **l'utente vuole il suo CV in PDF**. Il come si adatta all'istanza:

- **Default** (utente senza pipeline propria): compila `assets/cv-template.html` con i contenuti confermati (sostituisci i placeholder `{{...}}`, duplica i blocchi BEGIN/END per le voci ripetute, rimuovi le sezioni vuote), poi nel code tool: `pip install weasyprint --break-system-packages` e renderizza in PDF. Fallback se weasyprint non è installabile nel sandbox corrente: `reportlab` (tipicamente preinstallato) costruendo un layout equivalente a codice. Ultimo fallback: consegna il file HTML pronto per "stampa → salva come PDF" dal browser, dicendolo esplicitamente — mai consegnare nulla in silenzio.
- **Utente con formato proprio**: se l'utente ha un suo sorgente/pipeline (es. un sorgente LaTeX, un suo template), usalo per la SUA istanza: aggiorna quel sorgente con i contenuti tailorati e compila con il motore adatto disponibile nel sandbox (es. `pdflatex`). Chiedi, non assumere — e non promuovere il metodo di un utente a standard della skill.
- Il PDF finale va salvato in output e presentato all'utente con il tool di presentazione file. Controlla il risultato (una pagina? testo tagliato? placeholder residui `{{`?) prima di presentarlo.

### 5. Cover letter

250-350 parole, stesso posizionamento del CV. Struttura: aggancio specifico alla posizione/azienda (mai template-vuoto "sono entusiasta di candidarmi") → 2 match concreti con evidenze dal profilo → gestione onesta del gap più rilevante SE argomentabile (affinità, velocità di apprendimento dimostrata — senza scuse né bugie) → chiusura con disponibilità. Lingua: quella della JD, salvo diversa richiesta.

### 6. DM al recruiter

60-120 parole: il posizionamento compresso. Chi sei in mezza frase, il match più forte, una chiusura che chiede il passo successivo. Niente riassunto del CV: è un messaggio LinkedIn, non una cover corta. Stessa lingua della cover.

### 7. Consegna e destinazioni

- CV: file PDF presentato in chat. OFFRI (non fare d'ufficio) di salvarne una copia su Drive in `job-hunting/` (es. sottocartella `cv/`), utile come storico per candidatura.
- Cover e DM: testo in chat. Se l'utente vuole spedire, la bozza email si prepara via Gmail (bozza, MAI invio diretto).
- Se la candidatura è già nel tracker (2.3), suggerisci di annotare nel task che i materiali sono pronti — su richiesta.

## Cosa NON fare

- Non inventare NULLA che non sia nel master-profile (vale per CV, cover e DM allo stesso modo).
- Non hardcodare un motore di render come unico possibile: la gerarchia è weasyprint → reportlab → HTML consegnato, più la pipeline propria dell'utente se esiste.
- Non renderizzare senza la conferma dei contenuti (passo 3).
- Non produrre tre artefatti con posizionamenti diversi: se l'utente cambia il posizionamento su uno, riallinea gli altri.
- Non inviare mai email/messaggi: solo bozze.
