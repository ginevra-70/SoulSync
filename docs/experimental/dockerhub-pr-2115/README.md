# Proposta Docker Hub privata — SoulSync PR #2115

Questa documentazione accompagna l’integrazione sperimentale nel checkout della fork SoulSync. Nessuna GitHub Action è stata eseguita; nessuna immagine è stata costruita o pubblicata; nessun repository Docker Hub è stato creato o modificato. Non è stato effettuato alcun accesso a TrueNAS o alla rete locale.

## Origine e destinazione fissate

- Repository che ospita il workflow: `ginevra-70/SoulSync`.
- Il workflow accetta soltanto dispatch manuali da `ginevra-70/SoulSync`, sul suo branch predefinito corrente.
- Repository sorgente del checkout: esclusivamente `ginevra-70/SoulSync`.
- Branch sorgente atteso nella fork: `fix/music-assistant-scan-reliability`.
- Commit sorgente esatto: `8a94131ab8ec8e962d522fecd815d4ce29aed0ac`.
- Destinazione Docker Hub: `DOCKERHUB_NAMESPACE/soulsync`.
- Unico tag che il workflow può inviare: `DOCKERHUB_NAMESPACE/soulsync:pr-2115`.
- Piattaforme: `linux/amd64` e `linux/arm64`.
- Trigger: solo `workflow_dispatch`; per pubblicare occorrono `mode: publish` e `authorize_publish: yes`.

Il repository upstream è rilevante solo come riferimento al progetto originale. Il workflow, la verifica branch/commit, il checkout, le label dell'immagine e la destinazione di pubblicazione fanno riferimento alla fork `ginevra-70/SoulSync`.

## Controlli nella fork

Prima del checkout, entrambi i job verificano via API che il repository dell'evento sia esattamente `ginevra-70/SoulSync`, che l'owner sia `ginevra-70` e che il dispatch provenga dal branch predefinito corrente della fork. Poi il workflow scarica esclusivamente la fork al commit fissato, confronta `HEAD` con il commit atteso e interroga il branch `fix/music-assistant-scan-reliability` nella fork: il suo SHA deve coincidere con il commit atteso. Se branch o commit non sono presenti nella fork, se il branch punta altrove, o se l'API non risponde come previsto, il job fallisce.

Verifica GitHub in sola lettura aggiornata il **16 novembre 2026**: la fork `ginevra-70/SoulSync` ha branch predefinito `main`; il branch sorgente `fix/music-assistant-scan-reliability` punta al commit `8a94131ab8ec8e962d522fecd815d4ce29aed0ac`, che risulta **unsigned**. La PR #2115 è [aperta](https://github.com/Nezreka/SoulSync/pull/2115) e usa quel branch della fork. La verifica dei metadati della fork, del default branch e del branch sorgente è stata eseguita il 16 novembre 2026 tramite GitHub.

La verifica GitHub del **16 novembre 2026** riporta il commit come **unsigned** (`verification: false`, motivo `unsigned`). Per questa build sperimentale si accetta tale condizione: il workflow effettua il checkout allo SHA completo e verifica che il branch atteso nella fork punti ancora allo stesso SHA. La fissazione dello SHA identifica il contenuto della build, ma non autentica crittograficamente l'identità dell'autore. Nessuna nuova verifica della firma è stata eseguita in questa revisione.

Il nome Git dell'autore/committer era `BoulderBadgeDad`, mentre le API commit di GitHub associavano autore e committer all'account `Nezreka` sia nella fork sia nell'upstream. L'associazione è metadato GitHub e non prova chi abbia materialmente scritto il codice. Il workflow etichetta l'immagine con il repository fork e lo SHA, senza dichiarare che il commit è firmato o attribuirne l'autorialità.

### Branch predefinito verificato

Il branch predefinito della fork `ginevra-70/SoulSync` è `main`, verificato tramite GitHub il **16 novembre 2026**. Il workflow va quindi integrato in `main` per rendere disponibile il dispatch manuale; il controllo runtime verifica che l'esecuzione provenga da `refs/heads/main`. Il branch sorgente della build resta `fix/music-assistant-scan-reliability` al commit `8a94131ab8ec8e962d522fecd815d4ce29aed0ac` (unsigned).

## Secrets e variabile richiesti per un futuro setup

Per un futuro setup, in `ginevra-70/SoulSync` → Settings → Secrets and variables → Actions configurare:

- Secret `DOCKERHUB_USERNAME`: username/account identifier Docker Hub usato per autenticarsi.
- Secret `DOCKERHUB_TOKEN`: token Docker Hub personale o dell'organizzazione. Non incollarlo in chat e non inserirlo nei file.
- Variable `DOCKERHUB_NAMESPACE`: namespace Docker Hub che possiede il repository `soulsync`.

Il namespace Docker Hub può essere diverso dall'username usato per l'autenticazione. Il workflow richiede che tutti e tre i valori siano presenti e valida il formato del namespace prima di usarlo.

Prima di una futura pubblicazione, creare manualmente `DOCKERHUB_NAMESPACE/soulsync` in Docker Hub e impostarlo **Private**. Il workflow non crea né cambia repository o visibilità: interroga l'API autenticata e prosegue solo se il repository esiste, il nome e namespace corrispondono e `is_private` è `true`. Se è assente, pubblico o non verificabile, si arresta. Docker Hub può creare un repository mancante al primo push secondo la privacy predefinita del namespace; per questo il workflow richiede che il repository privato esista già. [Guida privacy Docker Hub](https://docs.docker.com/learn/content/share-your-image-on-docker-hub/), [API repository](https://docs.docker.com/reference/api/hub/latest/operations/GetRepository/), [API token](https://docs.docker.com/reference/api/hub/latest/operations/AuthCreateAccessToken/), [permessi dei token](https://docs.docker.com/security/access-tokens/reference/).

Docker Hub supporta PAT con risorse e scope selezionabili; usare un token con i permessi minimi necessari per leggere i metadati e pubblicare nell'unico repository previsto. Per un namespace organizzativo, un organization access token può essere usato sugli endpoint moderni namespace-scoped se ha scope e accesso al repository appropriati. Nessuna credenziale reale è stata richiesta o usata durante la preparazione. [Autenticazione e scope Docker Hub](https://docs.docker.com/reference/api/hub/latest/), [creazione token](https://docs.docker.com/reference/api/hub/latest/operations/AuthCreateAccessToken/).

## Comportamento e protezioni

- I soli trigger sono manuali. Il job scan ha `contents: read`; il job publish ha `contents: read`. Non è concesso `packages: write` o alcun altro permesso GitHub non necessario.
- La pubblicazione richiede sia `mode: publish` sia la conferma esplicita `authorize_publish: yes`. Il valore predefinito è `no`.
- La scansione Gitleaks è presente sia nel job scan sia nel job publish, prima delle verifiche Docker Hub e del build. Entrambi i checkout usano `fetch-depth: 0`; upload artefatti, commenti e sommario Gitleaks sono disabilitati.
- Il job publish controlla che il repository Docker Hub sia privato e che il tag `pr-2115` non esista, poi ripete entrambi i controlli immediatamente prima dell'azione build-and-push.
- Solo il job publish riceve i secrets Docker Hub e solo nei passaggi che li richiedono. Le credenziali non sono date a Gitleaks né scritte nel checkout.
- Buildx riceve un unico tag: `${DOCKERHUB_NAMESPACE}/soulsync:pr-2115`. Le label di provenienza e revisione riportano la fork e il commit fissato.
- Il file `.dockerignore` conserva le regole esistenti e include le quattro esclusioni approvate per il contesto Docker.
- Le GitHub Actions esterne sono fissate a SHA completi e immutabili: checkout v6.0.2 (`de0fac2e4500dabe0009e67214ff5f5447ce83dd`), Gitleaks Action v3.0.0 (`e0c47f4f8be36e29cdc102c57e68cb5cbf0e8d1e`), QEMU v4.3.0 (`1f40c72289eff860ee54a304f1438e3cff362e0a`), Buildx v4.4.1 (`f87e5991a6d7451dcb8d9637bfbc97413f497069`), Docker login v4.4.0 (`af1e73f918a031802d376d3c8bbc3fe56130a9b0`) e build-push v7.4.0 (`c3c9e263c25d99ce0380d002d59b67737d91b0dc`). Il pin fissa il codice dell'action, ma non verifica in modo indipendente il codice né le dipendenze transitive.

## Limiti e criticità residue

- **Commit unsigned accettato per questa build sperimentale.** La verifica GitHub del 16 novembre 2026 conferma che il branch della fork punta allo SHA atteso, ma il commit non ha una firma verificata. La build è fissata allo SHA esatto; chi rivede la proposta accetta che ciò non autentica la firma dell'autore.
- **Il tag va reso immutabile dall'utente in Docker Hub.** Docker Hub documenta tag immutabili e API dedicate. Prima della pubblicazione, configurare `pr-2115` come tag immutabile (regola regex `^pr-2115$`) dalla UI e non cambiare tale impostazione; questa proposta non modifica impostazioni Docker Hub. L'immutabilità protegge il tag dopo la prima creazione, mentre il precheck di esistenza resta non atomico e la concorrenza serializza soltanto esecuzioni di questo workflow. [Tag immutabili](https://docs.docker.com/docker-hub/repos/manage/hub-images/immutable-tags/), [verifica API](https://docs.docker.com/reference/api/hub/latest/operations/VerifyRepositoryImmutableTags/).
- **Private non significa accesso individuale.** Possono fare pull utenti, membri e token a cui siano stati concessi i permessi Docker Hub. Verificare separatamente i membri e i token del namespace.
- **Ambito Gitleaks limitato alla cronologia scaricata.** `fetch-depth: 0` rende disponibile la cronologia Git del checkout, ma la scansione non copre repository diversi, ref inaccessibili o segreti mai committati. Nessuna scansione Gitleaks è stata eseguita durante la preparazione.
- **Nessuna build o esecuzione del workflow è stata effettuata.** Non sono stati verificati token, privacy Docker Hub, namespace, stato del tag, API runtime, YAML tramite GitHub Actions o comportamento della build. L'effetto delle esclusioni Docker resta una valutazione statica.
- **Nessuna esecuzione o pubblicazione è stata effettuata.** Questa modifica prepara il workflow nel checkout; il push su GitHub e il dispatch finale `mode: publish` con `authorize_publish: yes` sono operazioni distinte. Nessun workflow è stato eseguito, nessuna build o login Docker è avvenuta.
- **Verifiche remote aggiornate al 16 novembre 2026.** La fork è `ginevra-70/SoulSync`, il default branch è `main`, e `fix/music-assistant-scan-reliability` punta allo SHA fissato `8a94131ab8ec8e962d522fecd815d4ce29aed0ac`; il commit è unsigned.

## File della proposta

- `build-pr-2115-dockerhub.yml` — workflow manuale e fail-closed.
- `dockerignore-pr-2115.append` — esclusioni aggiuntive del contesto Docker.
- `README.md` — procedura, verifiche e limiti.
