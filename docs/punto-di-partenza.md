# Punto di partenza: fatti e decisioni sul flusso di lavoro con agenti

Raccoglie quanto emerso dal 25 al 31 ottobre 2026, in una discussione tra il proprietario (`jorsm`) e Claude Code nata da una proposta di "etichette delle PR" nel repo `homelab-mobile`. Serve come base per l'agente che lavorerà in questo repo.

Non è una spec. Nessun meccanismo qui descritto è ancora costruito o provato. Ogni voce dice se è un **fatto** (verificato, con la fonte), una **decisione** del proprietario, una **proposta** da validare, o una **questione aperta**.

## 1. Perché questo repo

- Il proprietario sta cercando un flusso di lavoro con agenti che regga su progetti diversi: una web app con app Android (`homelab-mobile`), un'estensione VS Code (`markdown-reader`), altri.
- Finora l'harness (spec, `AGENTS.md`, skill `prepare-pr`, hook git, gate dei controlli) è stato copiato da un repo all'altro, e si disallinea.
- Decisione (31 ottobre): questo repo è insieme **laboratorio** e **futuro template**. Qui si discutono e si validano i meccanismi uno alla volta. Solo dopo si decide se adattare gli altri repo.
- Decisione: repo **pubblico**, così altri possono provare il flusso e dare feedback. App finta in **TypeScript**, come gli altri progetti, per riusare gli strumenti.
- Decisione: si ottimizza per **Claude Code**. Gli altri agenti (Codex, Copilot) si considerano quando il flusso con Claude è rodato.

## 2. L'esperienza da cui si parte

Fatti osservati nei repo del proprietario al 28 ottobre 2026:

| Osservazione | Dati |
| --- | --- |
| Volume di PR | `vertex-ai-models-chat-provider` 22, `markdown-reader` 10, `homelab-mobile` 17. PR aperte contemporaneamente, in tutti i repo: circa una dozzina |
| Etichette | nessuna PR le usava |
| Conflitti sull'indice delle spec | in `homelab-mobile` 6 PR di spec su 7 in conflitto con `develop`: tutte aggiungevano una riga in cima alla stessa tabella di `docs/specs/index.md`, e Prettier riallinea l'intera tabella |
| Collisioni di numero | una vera: due spec diverse con il numero 0003, una rinumerata 0004 a mano. Il numero 0004 era stato "riservato" a voce |
| Dimensione delle spec | la spec principale di `homelab-mobile` ha 831 righe; nessuna regola limita la dimensione |
| Citazioni delle spec | circa 220 citazioni del tipo "spec 0003" in 165 file |
| Lettura dei soli numeri | il proprietario, leggendo "la 0006 condizionata alla 0010, la coda alla 0009", non sapeva di cosa si parlasse senza aprire tre PR |
| Disallineamento della documentazione | è servita una skill apposta (`align-agents-md`) per tenere vero `AGENTS.md`; una spec riportava due collisioni dove ce n'era una |
| CI | girava solo sulle PR verso `develop` e `main`: le PR impilate su un altro branch non avevano controlli |

## 3. Principi

Emersi nella discussione e condivisi dal proprietario:

1. **Strumenti per ciò che ha una risposta sì/no ricavabile dai file; memoria dell'agente per ciò che richiede giudizio.** La memoria non è sbagliata in sé: va scelta caso per caso (§10).
2. **La mappa va portata all'agente.** Se l'agente ha una mappa chiara di dove stanno le informazioni, le legge; se non c'è, le ignora. Non va a cercarle da solo.
3. **Prima le funzioni esistenti (GitHub, Claude Code), poi le nostre.** Più volte nella discussione si stava reinventando qualcosa che esisteva già.
4. **Un rimando deve rompersi quando il bersaglio cambia,** altrimenti punta in silenzio alla cosa sbagliata.
5. **Il rigore segue il rischio.** Il flusso può essere uguale ovunque; le guardie crescono con il danno possibile. Un'app personale con qualche bug non fa danni; un sistema che tocca dati economici di clienti reali sì.
6. **Ogni meccanismo arriva con la sua ipotesi e la sua misura.** Se non migliora niente, si toglie (§9).

## 4. Fatti verificati

### GitHub

- Non si può chiedere una review a se stessi né approvare le proprie PR. Finché le PR degli agenti risultano aperte dall'account del proprietario, `review-requested:@me` resta vuota e Approve non esiste.
- Lo stato di bozza (draft) è nativo e si cambia con `gh pr ready` e `gh pr ready --undo`.
- La stessa sintassi di ricerca vale su github.com, con `gh` e nelle query salvate dell'estensione GitHub Pull Requests di VS Code. Per esempio: `is:open is:pr author:@me draft:false` cerca in tutti i repo.
- Su GitHub Free i repository privati non hanno protezione dei branch né ruleset. Chi può fare push può anche fare merge.
- Con il merge squash, GitHub può usare titolo e descrizione della PR come messaggio del commit. Così `git blame` porta al perché anche senza GitHub. Questa voce è da memoria, non riverificata.

### Claude Code

Fonte: documentazione ufficiale, `code.claude.com/docs/en/hooks` e `/memory`, letta il 29 ottobre 2026.

- **Hook.** Ogni hook riceve in JSON `session_id`, `transcript_path`, `cwd`, `hook_event_name`, e per gli strumenti `tool_name`, `tool_input`, `tool_use_id`. Nei subagenti riceve anche `agent_id`.
- **Aggiungere contesto.** `PreToolUse` e `PostToolUse` possono aggiungere contesto con `hookSpecificOutput.additionalContext`. Il testo arriva accanto al risultato dello strumento. La documentazione consiglia frasi fattuali, non ordini, per non attivare le difese contro la prompt injection.
- **`SessionStart`** distingue `startup`, `resume`, `clear`, `compact` e `fork`.
- **Regole con percorsi.** I file `.claude/rules/*.md` con frontmatter `paths:` (glob) si caricano solo quando l'agente legge, scrive o modifica un file corrispondente, anche con `cat` o `head`. Le regole senza `paths:` si caricano sempre.
- **File di istruzioni nelle sottocartelle.** `CLAUDE.md` e `AGENTS.md` dentro una sottocartella si caricano quando l'agente apre un file lì.
- **Dopo una compattazione** il `CLAUDE.md` della radice si ricarica subito; le regole con `paths:` e i file delle sottocartelle si ricaricano al primo file corrispondente toccato.
- **Worktree.** Le regole con percorsi valgono per un worktree diverso solo se la sessione parte lì o ci si sposta con `/cd`.
- **`AGENTS.md`** si legge solo se non c'è un `CLAUDE.md` nella cartella di lavoro o sopra. Richiede Claude Code v2.1.277 o successive.
- **Plugin.** Possono contenere hook, skill e comandi, e si installano in più repo invece di copiarli. Questa voce è da memoria, da verificare.

### OpenSpec

Fonte: repo `Fission-AI/OpenSpec` e la sua cartella `docs/`, letti il 28 ottobre 2026.

- **Struttura.** `openspec/specs/<area>/spec.md` descrive come funziona il sistema oggi. `openspec/changes/<nome>/` è una modifica proposta, con `proposal.md` (perché), `design.md` (come), `tasks.md` (cosa fare) e i delta delle spec.
- **Formato delle spec.** Requisiti con SHALL o MUST e scenari GIVEN/WHEN/THEN. Solo comportamento osservabile: niente nomi di classi o librerie.
- **Delta.** `ADDED`, `MODIFIED` (sostituisce il requisito con lo stesso nome) e `REMOVED`, nello stesso percorso della spec che modificano.
- **Archiviazione.** `archive` applica i delta alle spec e sposta la modifica in `changes/archive/AAAA-MM-GG-<nome>/`.
- **Git.** Non tocca mai git. La guida per i team suggerisce un branch e una PR per modifica, e di archiviare dopo il merge.
- **Verifica.** `/opsx:verify` confronta il codice con la modifica (completezza, correttezza, coerenza). Non è attivo di base (profilo "expanded"), non blocca, e nulla garantisce che giri con un contesto pulito.
- **Conflitti.** Due modifiche allo stesso requisito si scontrano all'archiviazione. In passato la seconda cancellava in silenzio scenari della prima, come documenta `openspec-parallel-merge-plan.md`. Le versioni recenti la bloccano e la segnalano in `validate`.
- **Le modifiche scartate** non hanno un posto: l'archivio è solo per le concluse.
- **Mappa spec ↔ file.** Nella documentazione letta non c'è un legame tra spec e file di codice.
- **Diffusione.** Circa 73.000 stelle, 5.000 fork, licenza MIT, nato nell'agosto 2025, rilasci quasi settimanali (v1.15.0 il 24 ottobre 2026). Su Hacker News quasi nessuna discussione.

### Contesto dello "spec-driven development"

- **Non esiste uno standard** per le spec scritte per agenti: ci sono strumenti popolari (Spec Kit di GitHub, circa 143.000 stelle; Kiro di AWS; OpenSpec; Tessl).
- **Thoughtworks Technology Radar,** novembre 2025: tecnica in "Assess". Flussi elaborati, file generati lunghi e difficili da rivedere.
- **Birgitta Böckeler, martinfowler.com,** ottobre 2025:
  - un unico flusso non va bene per lavori di dimensioni diverse;
  - troppo Markdown da rivedere ("preferirei rivedere il codice");
  - falso senso di controllo, perché gli agenti ignorano parti delle spec;
  - distingue spec-first, spec-anchored e spec-as-source.
- **Standard esistenti vicini:**
  - **ADR/MADR:** decisioni numerate con stato, indice generato da strumenti come `adr-tools`;
  - **`log4brains`:** la data davanti al nome per evitare collisioni;
  - **RFC di Rust:** il numero è quello della PR;
  - **Spec Kit:** cartelle numerate da uno script, con branch con lo stesso numero (da memoria).

## 5. Decisioni del proprietario

| Tema | Decisione | Quando |
| --- | --- | --- |
| Etichette | Scartato il sistema di sette etichette sincronizzate con labeler e filtri nell'app: costava più di quanto dava | 26 ott |
| `needs-decision` | Sì: segnala una decisione da prendere (veloce, dal telefono), distinta da una code review (lunga). Uguale in tutti i repo | 26 ott |
| Chi ha il turno | Si usano le funzioni di GitHub (bozza, review) invece di etichette come `awaiting-code-review` | 26 ott |
| Priorità | Non servono: le PR sono poche. Eventualmente `p1`, `p2` | 26 ott |
| Spec scartate | Si uniscono nel repo con stato `rejected`, invece di chiudere la PR senza merge | 26 ott |
| Sottocartelle per stato | No: lo stato sta nel frontmatter; spostare file rompe i link | 26 ott |
| PR | Sono un parcheggio di idee e lavoro: "un branch, un post-it". Servono al proprietario per lavori paralleli e scarico mentale, e agli agenti come compito da completare, non come contesto di progetto | 26 ott |
| Citazioni | Formato `spec 0003-interfaccia-mobile §5.11` (numero e nome del file, più la sezione), imposto da codice | 30 ott |
| Rimandi alle sezioni | Adottati (`§`) | 30 ott |
| Dimensione delle spec | È un problema: va limitata. Il valore è aperto | 30 ott |
| Revisione | Serve uno step di revisione con contesto pulito, perché chi implementa ignora parti della spec | 30 ott |
| Rigore | Proporzionato al rischio del repo | 30 ott |
| Agenti | Si ottimizza per Claude Code | 31 ott |
| Questo repo | Pubblico, TypeScript, laboratorio e template | 31 ott |

## 6. Proposte da validare

Ogni proposta ha un'ipotesi in §9.

### Spec

- **Indice.** Togliere l'indice scritto a mano: `npm run specs` lo stampa dal frontmatter.
- **Numeri.** `npm run spec:new <nome>` crea il file con il primo numero libero, guardando `develop` e i branch `spec/*` remoti (`git ls-remote`). Un test rifiuta due spec con lo stesso numero. Una volta unita, una spec non cambia più nome.
- **Formato del nome.** Il numero con il nome: `0012-notifiche-agenti.md`. Il numero tiene ordine e brevità; il nome fa fallire i rimandi se il bersaglio cambia.
- **Una PR di spec tocca solo i suoi file.** Le modifiche ad altre spec si elencano in una sezione e si applicano all'accettazione.
- **Decisioni.** Ogni decisione ha una sezione numerata e citabile nella spec a cui appartiene, con le alternative scartate. Niente cartella di decisioni separata: le decisioni trasversali restano tra le invarianti di `AGENTS.md`. Un'ipotesi intermedia, file separati per decisione, è stata considerata e lasciata da parte per non avere due posti da allineare.
- **"Non deve".** Ciò che non si deve mai fare diventa un requisito esplicito ("SHALL NOT"), possibilmente con un test: un agente può ignorare un documento, non un test che fallisce.
- **Limite di dimensione** controllato da un test, con il messaggio "dividi la spec".

### Mappa

- **Regole di Claude Code come mappa.** Una regola con `paths:` per area contiene due o tre righe di rimandi (`spec … §…`), non contenuti. È la "progressive disclosure" delle skill applicata alla documentazione:

  | Livello | Quando entra nel contesto | Cosa contiene |
  | --- | --- | --- |
  | 1 | sempre | `AGENTS.md` / `CLAUDE.md`: cartelle, comandi, invarianti |
  | 2 | al primo file toccato | la regola del percorso |
  | 3 | su richiesta dell'agente | la sezione della spec |

- **Test della mappa:**
  - ogni glob delle regole corrisponde ad almeno un file;
  - ogni file del codice è coperto da almeno una regola, con un elenco esplicito di eccezioni (file generati, lock);
  - ogni rimando `§` punta a una spec e a una sezione esistenti.
- **Citazioni nel codice.** Una regex trova tutto ciò che somiglia a una citazione (`spec` seguito da un numero, oppure `§`) e rifiuta le forme non canoniche, con un messaggio che dice come scriverle. Gira su tutto il repo, non solo sui file cambiati: costa meno di un secondo, e chi rinomina una sezione vede tutti i rimandi rotti. Lo esegue `npm run check`, e lo Stop hook lo esegue a fine turno: l'agente corregge prima di fermarsi.
- **Fonte unica tra campo `code:` delle spec e regole.** Le regole scritte a mano, oppure generate dal frontmatter delle spec: da decidere nella prova.
- **Docstring e commenti.** Le docstring spiegano il codice dove sta: gli agenti le aggiornano insieme al codice, e il proprietario non ha visto il contrario. Ciò che sta altrove (spec, README, altri file) va richiamato con un rimando verificabile, non rispiegato.

### PR

- **Ciclo:** bozza mentre lavora l'agente, pronta quando tocca al proprietario (`gh pr ready` alla fine di `/prepare-pr`), di nuovo bozza quando il proprietario chiede modifiche.
- **Descrizione e commenti.** La descrizione appartiene all'agente: è il riepilogo per la revisione. I commenti sono del proprietario: istruzioni sul lavoro di quella PR, che non vanno nella spec. L'agente non li modifica, risponde a ciascuno con "Fatto in `sha`" o con il motivo, e porta nella spec ciò che è contesto di progetto.
- **Hook della PR del branch.** Quando l'agente entra in un branch con una PR aperta, riceve titolo, descrizione e commenti; a ogni messaggio dell'utente riceve i commenti nuovi; dopo una compattazione si ricarica tutto. Uno stato per sessione evita le ripetizioni: due PR correlate nella stessa sessione restano separate. Il branch che conta è quello della cartella di lavoro dell'agente, non quello aperto dal proprietario in VS Code: ogni lavoro ha il suo worktree.
- **Hook del lavoro in corso.** Quando l'agente tocca un file, segnala le altre sessioni o PR che lo stanno modificando. Informa, non blocca. Due fonti:
  - in locale, i worktree con `git worktree list` più `git diff --name-only`: niente rete, e si vede anche il lavoro non pushato;
  - su GitHub, una query GraphQL delle PR aperte con i file, in 0,3–0,8 s, fatta all'avvio e rinfrescata ogni pochi minuti in una cache.
- **Verifica con contesto pulito.** Prima di segnare la PR pronta, un subagente nuovo riceve solo la spec o la modifica e il diff, senza la conversazione di chi ha implementato, e riporta dove il codice si allontana dalla spec.
- **CI su tutte le PR,** anche quelle impilate su branch diversi da `develop`.

### Identità degli agenti

| | Stesso account del proprietario | Identità separata (GitHub App) |
| --- | --- | --- |
| Configurazione | nessuna | app, chiave privata, token agli agenti |
| Coda del proprietario | `author:@me draft:false`, mescolata alle PR aperte a mano | `review-requested:@me` |
| Approve | impossibile | possibile |
| Chi ha fatto cosa | indistinguibile, salvo `Co-Authored-By` | visibile |
| "Gli agenti non fanno merge" | memoria | sempre memoria, senza protezione dei branch |

Proposta: nei repo a basso rischio basta lo stesso account. L'identità separata serve quando servono le approvazioni, cioè nei repo critici. Il flusso va scritto in modo che il passaggio cambi solo la query della coda.

## 7. Questioni aperte

1. **Spec per epic o per area.** Il proprietario le ha descritte sia come "epic" sia come "fatti", che sono due ruoli diversi:

   | | Epic | Area |
   | --- | --- | --- |
   | Nasce da | un lavoro | una parte del sistema (un modulo) |
   | Dove sta la verità di oggi | sparsa tra più spec | in un file |
   | Il perché del lavoro | dentro la spec | in una proposta archiviata, o nella PR |
   | Mappa file → spec | larga | stretta |

   La domanda pratica: quando un agente deve cambiare una parte del sistema, quante spec deve leggere per sapere com'è oggi? Il modello di OpenSpec (aree per i fatti, modifiche temporanee per il lavoro) usa entrambi i ruoli, al prezzo di un passaggio di archiviazione. Il proprietario non è persuaso dalle aree.
2. **OpenSpec:** adottare lo strumento, o prenderne solo le idee (aree, delta, archivio) con strumenti propri per mappa e PR.
3. **Limite di dimensione delle spec:** quante righe.
4. **Spec piccole nella stessa PR del codice:** sì per una spec con un solo deliverable? In quel caso l'accettazione non è un merge, ma un commento del proprietario più un commit che porta lo stato ad `accepted`.
5. **Accettazione e bozza.** Il proprietario ha chiesto se l'uscita dalla bozza possa segnare l'accettazione di una spec. Proposta: no. La bozza dice di chi è il turno, non se la spec è approvata. L'approvazione resta il merge, o il commit di cui alla domanda 4.
6. **Merge squash** con la descrizione della PR come messaggio del commit.
7. **Identità separata degli agenti:** quando.
8. **Licenza del template** (proposta: MIT), e lingua: italiano per il proprietario, inglese per chi lo prova da fuori.
9. **Distribuzione:** hook e skill copiati nel template, oppure un plugin di Claude Code condiviso, con le regole che restano nel repo perché i percorsi sono suoi.

## 8. Strumenti e memoria, caso per caso

| Cosa | Proposta | Chi lo garantisce | Perché |
| --- | --- | --- | --- |
| Format, lint, tipi, test | gate | Stop hook, pre-push, CI | sì/no |
| Commit su `develop` / `main` | rifiutati | hook git | sì/no |
| Nome del branch | controllato | pre-push | sì/no |
| Indice delle spec | generato | script | sì/no |
| Numero della spec | assegnato | script e test | sì/no |
| Percorsi delle regole e copertura dei file | controllati | test | sì/no |
| Forma ed esistenza delle citazioni | controllate | test | sì/no |
| Dimensione delle spec | limitata | test | sì/no |
| Leggere le regole del file toccato | portate all'agente | Claude Code (regole con `paths:`) | l'agente non cerca da solo |
| Leggere la PR del branch e i commenti | portata all'agente | hook | idem |
| Lavoro in corso sugli stessi file | segnalato | hook | idem |
| Verifica del codice contro la spec | subagente con contesto pulito | skill `/prepare-pr` | resta giudizio, con occhi nuovi |
| Bozza e pronta | passo di `/prepare-pr` | memoria | errore raro, danno piccolo |
| `needs-decision` | messa e tolta dall'agente | memoria | giudizio |
| Cosa va nella spec e cosa nella PR | | memoria | giudizio |
| Risposte ai commenti | | memoria | giudizio; se mancano si vede subito |
| Docstring vere | | memoria, presenza imposta dal lint | regge per il codice vicino |
| Gli agenti non fanno merge | | memoria | GitHub Free non lo impedisce |

## 9. Valutazione

Tre domande diverse:

1. **I meccanismi funzionano?** Test con guasti seminati: per ogni controllo, un caso che deve farlo fallire. Che una regola si sia caricata si vede nella trascrizione della sessione.
2. **Cambiano il comportamento degli agenti?** Una suite di compiti con trappole, eseguita con varianti diverse.
3. **Il risultato è migliore, e non peggiore in modi nascosti?** Misure dall'uso reale, più soglie di guardia.

### Suite di compiti (proposta)

- **I compiti** sono scritti prima di costruire i meccanismi. Ognuno ha una trappola:
  - un "non deve" violato dalla soluzione ovvia;
  - un'alternativa già scartata;
  - un file toccato da un'altra sessione;
  - un commento del proprietario che cambia il compito;
  - una parte della spec che si tende a ignorare.
- **Alcuni compiti** vengono da lavoro vero, per non scriverli su misura per il sistema.
- **Le varianti:** sistema completo; niente (solo codice e README); senza un pezzo alla volta. Le varianti senza un pezzo dicono quale meccanismo vale il suo costo.
- **L'esecuzione:** `claude -p` in un worktree pulito, modello fisso, tre ripetizioni per compito e variante. Si confrontano percentuali.
- **Il giudizio:** test dove possibile; altrimenti un agente con una griglia, e un campione ricontrollato dal proprietario alla cieca.
- **Il costo:** partire con 5 compiti e 2 varianti (30 esecuzioni), poi aggiungere solo ciò che è in dubbio.
- **Dove sta:** in `evals/`, con `npm run eval -- --variant …` e risultati versionati.

### Ipotesi per meccanismo

| Meccanismo | Ipotesi | Trappola |
| --- | --- | --- |
| Regole con `paths:` | meno violazioni dei "non deve" rispetto a nessuna mappa | compito la cui soluzione ovvia viola un requisito di un'altra sezione |
| Decisioni con alternative scartate | meno soluzioni già scartate riproposte | compito in cui la prima idea è proprio quella scartata |
| Hook della PR del branch | i commenti del proprietario vengono eseguiti senza doverli ripetere | commento su GitHub che cambia il compito |
| Hook del lavoro in corso | meno conflitti tra sessioni parallele | due sessioni sullo stesso file |
| Verifica con contesto pulito | trova scostamenti dalla spec che chi implementa non vede | deviazione seminata dalla spec |
| Indice generato e `spec:new` | zero conflitti e collisioni | due PR di spec parallele (deterministico) |
| Test delle citazioni | nessun rimando rotto su `develop` | rinomina di una sezione |

### Uso reale

Misure ricavabili da `gh` e `git`:
- commenti del proprietario per PR;
- commit dopo "pronta";
- PR di correzione nei 14 giorni successivi al merge;
- conflitti e rinumerazioni;
- CI rossa al primo push;
- token e tempo per PR, dalle trascrizioni;
- tempo di revisione percepito, con un voto da 1 a 5.

Il confronto con i repo passati indica una tendenza, non una causa.

### Soglie di guardia ("non peggio, anche se funziona")

- costo per compito: non oltre +20%;
- contesto caricato da regole e hook in una sessione tipica;
- latenza degli hook: sotto 200 ms;
- Markdown da leggere per PR;
- rigidità: almeno un compito in cui la spec suggerisce una soluzione peggiore di quella possibile;
- Goodhart: controllo soddisfatto senza fare la cosa, per esempio una citazione formalmente giusta ma non pertinente.

### Errori di metodo da evitare

- **Ipotesi e soglie scritte prima delle misure.**
- **Esecuzioni legate a un modello preciso,** e rifatte quando cambia.
- **Pochi dati:** si vedono solo differenze grandi, che sono le uniche che giustificano complessità.
- **Il proprietario non è un giudice cieco:** da qui il campione alla cieca.

## 10. Ordine di lavoro proposto

1. Valutazione: la struttura di `evals/`, le ipotesi, i primi compiti.
2. Formato delle spec, con le questioni aperte 1–5.
3. Mappa: regole con `paths:`, citazioni, test. Primo esperimento: verificare il caricamento delle regole e il loro ritorno dopo una compattazione.
4. PR: ciclo, commenti, hook della PR del branch.
5. Coordinamento tra sessioni.
6. Guardie per livello di rischio.

## 11. Fonti

- [OpenSpec](https://github.com/Fission-AI/OpenSpec): `docs/concepts.md`, `docs/team-workflow.md`, `docs/existing-projects.md`, `docs/reviewing-changes.md`, `openspec-parallel-merge-plan.md`, `CHANGELOG.md`
- [Claude Code: hooks](https://code.claude.com/docs/en/hooks) e [memoria e regole](https://code.claude.com/docs/en/memory)
- [Thoughtworks Radar: Spec-driven development](https://www.thoughtworks.com/radar/techniques/spec-driven-development)
- [Böckeler, Understanding Spec-Driven-Development: Kiro, spec-kit, and Tessl](https://www.martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)
- [Recensione di OpenSpec, andrew.ooo](https://andrew.ooo/posts/openspec-spec-driven-development-review)
- [Hashrocket, OpenSpec vs Spec Kit](https://hashrocket.com/blog/posts/openspec-vs-spec-kit-choosing-the-right-ai-driven-development-workflow-for-your-team)
