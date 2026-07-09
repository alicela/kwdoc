# Kraftwerk - Comparaison Code / Tests Cucumber / Guide Utilisateur

## Statut de ce document

Remplace entièrement la première passe (non vérifiée). Trois sources ont été confrontées, chacune explorée indépendamment et exhaustivement :
- **Code** : voir `01-architecture-globale.md`, `02-fonctionnalites-principales.md`, `05-anomalies-code.md` (vérifié fichier:ligne).
- **Tests** : les 15 fichiers `.feature` de `kraftwerk-functional-tests` + `out_data_encryption.feature` de `kraftwerk-encryption-tests`, avec leurs step definitions.
- **Guide utilisateur** : les 19 fichiers Markdown du zip `guide-utilisateurs-kraftwerk-main`.

Pour chaque écart, vous trouverez ce que dit chaque source (avec référence), l'analyse de la contradiction, et une question explicite pour trancher la version à retenir pour la cible.

---

## Décisions arbitrées (client, 2026-07-08)

Les quatre questions bloquantes ont été tranchées. Elles font désormais foi pour la conception de la cible :

| Réf. | Sujet | Décision retenue | Source retenue |
|------|-------|------------------|----------------|
| **A** | Paradonnées | ❌ **Exclues** — Kraftwerk ne traite pas les paradonnées dans la cible. Le package `extradata/paradata/` et les variables `NB_ORCHESTRATORS`, `DURATION_ACTIVE_*`, etc. ne sont **pas** repris. Traitement externe futur. | **Guide** |
| **B** | Réconciliation multimode | ✅ **Fusion automatique conservée** — la cible fusionne automatiquement les données de plusieurs modes (logique de `ReconciliationProcessing`), au lieu de laisser l'utilisateur l'écrire à la main. | **Code** |
| **C** | Mode « hors Genesis » | ❌ **Supprimé** — la cible ne lit les données **que via Genesis** (par `partitionId`). Endpoints `/main`, `/main/file-by-file`, `/main/lunatic-only` et arguments batch `MAIN`/`FILE_BY_FILE` non repris. Les fixtures de test correspondantes devront être rebâties sur Genesis. | **Guide** |
| **D** | Suivi des jobs | ✅ **Suivi unifié et persistant** — tous les traitements asynchrones de la cible exposent un statut de job cohérent, survivant à un redémarrage (≠ les deux stores in-memory actuels dont un est mort). | Cible (ni code ni guide) |

**Conséquences en cascade sur les autres points :**
- **F (limite 400 Mio)** → **sans objet** : cette limite ne s'appliquait qu'au mode « hors Genesis » supprimé (C).
- **C impacte massivement les tests** : la majorité des `.feature` actuels s'appuient sur des fixtures fichiers locaux ; ils devront être portés sur des scénarios Genesis (`GenesisClientStub`) lors de la définition des cas de test cible.
- **B + suppression du vocabulaire VTL** (cf. `02-decisions-client-arbitrages.md`) : la fusion multimode automatique est **conservée fonctionnellement**, mais son implémentation cible ne devra plus s'exprimer en VTL/Trevas — c'est un point de conception à traiter (comment reproduire l'union/jointure multimode sans moteur VTL, p. ex. en SQL DuckDB).
- **A n'affecte pas le reporting** : l'exclusion porte sur les *paradonnées* uniquement ; les *données de reporting* (`_REPORTINGDATA`, `OUTCOME_SPOTTING`) ne sont pas concernées par cette décision et restent à arbitrer séparément si besoin lors de l'extraction des règles métier.

---

## A. Paradonnées : traitées ou pas par Kraftwerk ? 🔴 Bloquant → ❌ Exclues (guide)

- **Guide** (`pages/exports/para.md`, intro) : affirme explicitement **"Kraftwerk ne traite pas les paradonnées"** à ce jour — Kraftwerk ne ferait que déposer les fichiers sur l'applishare, à charge pour l'utilisateur de les exploiter lui-même (exemple de script R fourni). Le guide annonce un traitement automatisé "à venir".
- **Code** : un package complet `kraftwerk-core/extradata/paradata/` existe, avec parsing des événements, machine à états pour reconstituer sessions/orchestrateurs, et calcul de plusieurs variables (`NB_ORCHESTRATORS`, `DURATION_ACTIVE_ORCHESTRATORS`, `NB_SESSIONS`, `DURATION_ACTIVE_SESSIONS`, `DATE_COLLECTE`, `CHANGES_PRENOM`) injectées directement dans les données de l'unité enquêtée (`02-fonctionnalites-principales.md` §6).
- **Tests** : `do_we_get_paradata.feature` exerce réellement ce parsing et vérifie des valeurs précises (`NB_ORCHESTRATORS=6`, durées au format `"0 jours, 01:04:20"`, etc.) — donc la fonctionnalité existe et fonctionne au moins jusqu'au niveau du modèle en mémoire.

**Analyse** : contradiction frontale. Soit le guide décrit une version antérieure de Kraftwerk (le traitement des paradonnées aurait été ajouté au code depuis, sans mise à jour du guide), soit le code contient une fonctionnalité qui n'est jamais réellement invoquée en production (aucun endpoint REST ni argument batch documenté dans `02-fonctionnalites-principales.md` ne déclenche explicitement `ParadataParser` — il est appelé automatiquement dès qu'un fichier de paradonnées est présent dans le dossier d'entrée, via `BuildBindingsSequence`, sans commutateur dédié). Le test Cucumber ne teste que la brique interne, pas un endpoint utilisateur.

**Question pour vous** : la cible doit-elle intégrer le traitement des paradonnées (calcul de durées de session/passation) comme une fonctionnalité de Kraftwerk, ou bien confirmer la vision du guide (Kraftwerk n'en fait rien, ce sera un traitement externe futur) ?

---

## B. Réconciliation multimode : automatique ou pas ? 🔴 Bloquant → ✅ Fusion automatique conservée (code)

- **Guide** (`pages/fonctionnement/vtl.md`, section "Réconciliation multimode") : affirme explicitement **"la réconciliation multimode n'est pas encore permise par Kraftwerk à ce jour"** — la gestion des doublons entre modes serait manuelle, via un script VTL utilisateur (`gestion_doublons.vtl`) que le concepteur doit lui-même écrire.
- **Code** : `dataprocessing/ReconciliationProcessing.java` (~300 lignes) génère **automatiquement** l'union/jointure VTL de tous les datasets par mode en un dataset `MULTIMODE`, y compris la détection des mesures communes/absentes entre modes et l'harmonisation des types quand ils divergent (`02-fonctionnalites-principales.md` §3, `05-anomalies-code.md` point 10 pour le contexte plus large TCM).
- **Tests** : `do_we_apply_vtl.feature` teste uniquement l'injection d'un script VTL **utilisateur** au point d'extension "reconciliation_specifications" (calcul d'une variable `RECONCILIATION_VAR`) — cela ne prouve ni n'infirme que la fusion automatique des modes fonctionne réellement de bout en bout ; aucun test ne vérifie le résultat de l'union/jointure automatique elle-même sur un cas multimode réel.

**Analyse** : le guide et le code se contredisent sur un point structurant (est-ce que Kraftwerk sait déjà fusionner plusieurs modes de collecte automatiquement, ou est-ce entièrement à la charge de l'utilisateur ?). Sans test qui exerce réellement un cas à 2+ modes de collecte simultanés, il est impossible de trancher côté tests si le code de `ReconciliationProcessing` produit un résultat correct en pratique — c'était déjà l'une des 9 questions du premier passage ("La réconciliation automatique est-elle vraiment implémentée ?").

**Question pour vous** : la fusion automatique multimode (hors TCM, hors liens 2à2 déjà tranchés) est-elle une fonctionnalité qui doit être reprise dans la cible, ou le guide a-t-il raison de dire qu'elle n'est pas fiable/pas utilisée aujourd'hui et qu'elle doit rester un point d'extension manuel ?

---

## C. Mode "hors Genesis" (traitement de fichiers locaux) : à garder ou déjà en voie de disparition ? 🔴 Bloquant → ❌ Supprimé (guide)

- **Guide** (`pages/exports/enqhorsgenesis.md`) : le titre de la page est explicitement **"3.7 Les enquêtes hors Genesis (DEPRECATED)"**, avec la mention que ce mode "est en voie de disparition" — Platine Collecte ne produit plus de fichiers XML, les données transitent désormais toujours via Genesis.
- **Code** : les endpoints `/main`, `/main/file-by-file`, `/main/lunatic-only`, ainsi que le mode batch `--service=MAIN`/`FILE_BY_FILE`, sont pleinement implémentés, actifs, non dépréciés dans le code (pas de `@Deprecated`), et couverts par une bonne partie des tests Cucumber (`do_we_export_data.feature`, `do_we_export_data_file_by_file.feature`, `do_we_export_data_lunatic_only.feature`, `do_we_increment.feature`, etc.).
- **Tests** : couvrent abondamment ce mode — c'est même la majorité des scénarios de test du dépôt.

**Analyse** : ce n'est pas une contradiction de fait (le code n'affirme jamais que ce mode restera indéfiniment), mais un écart de statut : le guide utilisateur annonce une dépréciation fonctionnelle qui n'est reflétée nulle part dans le code (pas d'annotation `@Deprecated`, endpoints toujours actifs) ni dans les tests (qui continuent d'en dépendre massivement comme moyen principal de test, y compris pour des fonctionnalités non liées à Genesis comme les boucles ou les variables numériques). C'est directement lié à la première question du tout premier passage d'analyse ("Faut-il supprimer le traitement local de fichiers sans passer par Genesis ?").

**Question pour vous** : confirmez-vous la suppression du mode "hors Genesis" dans la cible (cohérent avec le guide et avec le passage à `partitionId`/Genesis-only déjà acté pour l'appel de données), ce qui impliquerait de réécrire une bonne partie des scénarios de test actuels sur des fixtures Genesis plutôt que sur des fichiers locaux ?

---

## D. Suivi de statut des jobs asynchrones : incomplet des deux côtés 🔴 Bloquant → ✅ Suivi unifié et persistant (cible)

- **Guide** : ne documente un mécanisme de suivi de job (`HTTP 202` + `jobId`, puis `GET /jobs/{jobId}`) que pour un seul endpoint, `PUT /main/genesis/by-questionnaire` (`pages/exports/enqgenesis.md`). Aucune mention de suivi de statut pour `/main`, `/main/file-by-file`, `/main/lunatic-only`, `/json`, `/json/replay`.
- **Code** : confirme qu'il existe deux stores de jobs distincts et non interopérables ; seul celui utilisé par `/main/genesis/by-questionnaire` (`InMemoryExportJobStore` + `GET /jobs/{jobId}`) semble fonctionnel. L'autre (`InMemoryJobStore`, utilisé par `/main`, `/main/file-by-file`, `/main/lunatic-only`) semble ne jamais être exposé en HTTP (voir `05-anomalies-code.md` point 1). Aucun des deux stores n'est persistant.
- **Tests** : aucun scénario Cucumber ne teste `GET /jobs/{jobId}` ni le comportement asynchrone en général (les tests appellent directement les classes de traitement de manière synchrone, en contournant la couche REST/async).

**Analyse** : ici, code et guide sont "d'accord" par omission (aucun des deux ne prétend qu'un suivi de statut existe pour les endpoints hors Genesis-by-questionnaire), mais cela signifie que ce point n'a jamais été un problème testé ni documenté — juste un point mort silencieux. Pour une cible construite autour de traitements asynchrones par `partitionId`, ce sujet devient structurant.

**Question pour vous** : la cible doit-elle fournir un suivi de statut de job unifié et fonctionnel pour **tous** les modes de traitement asynchrones (avec persistance, pour survivre à un redémarrage), plutôt que de reproduire le système actuel à deux stores dont un est cassé ?

---

## E. Endpoint reporting Genesis : chemin différent entre guide et code 🟡 Important

- **Guide** (`pages/exports/reporting.md`, section 4) : documente l'endpoint sous le nom **`/reportingdata/main/genesis`**.
- **Code** : l'endpoint réel est **`PUT /reportingdata/genesis`** (`ReportingDataService.java:59`), sans le segment `main`.

**Analyse** : écart de documentation simple (nom d'URL), sans ambiguïté sur le code à privilégier. Ne nécessite pas d'arbitrage — le nom exact sera de toute façon à redéfinir pour la cible.

**Proposition par défaut** : corriger le guide sur ce point n'a plus d'intérêt (la cible aura de nouveaux noms d'endpoints) ; à signaler seulement si vous voulez que je corrige aussi le guide existant en parallèle de la refonte.

---

## F. Limite de taille de 400 Mio : absente du guide utilisateur 🟡 Important → ⚪ Sans objet (mode hors Genesis supprimé, cf. C)

- **Code** : limite stricte de `419430400` octets (400 Mio exacts) sur la taille des fichiers/dossiers traités par le pipeline `MainProcessing` (mode "hors Genesis"), avec une erreur HTTP 413 explicite au-delà (`02-fonctionnalites-principales.md` §9, confirmé par la constante et son usage).
- **Guide** : ne mentionne cette limite nulle part, dans aucun des 19 fichiers. Seule la notion de "taille de lot" (nombre d'interrogations par batch, 100 à 5000 selon les pages) est documentée, ce qui est un concept différent (nombre d'enregistrements, pas taille en octets).
- **Tests** : aucun scénario ne teste le déclenchement du 413.

**Analyse** : si le mode "hors Genesis" est supprimé (cf. point C), cette limite disparaît d'elle-même puisqu'elle ne s'applique qu'à ce mode. Si le mode est conservé, il s'agit d'un comportement non documenté aux utilisateurs, qui pourraient être surpris par un rejet de traitement.

**Proposition par défaut** : ce point devient sans objet si la question C est tranchée en faveur de la suppression du mode fichiers locaux. Sinon, la limite devra être documentée dans le guide cible.

---

## G. `/contexts/schedules/v2` (exports automatisés) : hors périmètre du code Kraftwerk 🟡 Important

- **Guide** (`pages/exports/expauto.md`) : documente en détail un mécanisme de planification d'exports automatiques via des endpoints `POST /contexts/schedules/v2`, `PUT /contexts/{collectionInstrumentId}/schedules/v2/{scheduleUuid}`, `GET /contexts/schedules/v2`, orchestré deux fois par jour par un flux nommé `export-automatisation-workflow-pd-cron-main` et par Rundeck.
- **Code** : recherche exhaustive dans `kraftwerk-core` et `kraftwerk-api` — **aucune trace** de ces endpoints, de cette terminologie (`ContextSchedule`, `schedules/v2`) ni de ce workflow.

**Analyse** : ce n'est probablement pas un écart de documentation mais un **problème de périmètre du guide lui-même** — ces endpoints appartiennent vraisemblablement à Genesis ou à un service d'orchestration distinct (mentionné ailleurs dans le guide sous le nom "Bangles"), pas à Kraftwerk. Le guide utilisateur "Kraftwerk" documente donc, au moins pour cette section, un service qui n'est pas Kraftwerk.

**Question pour vous** : la planification d'exports récurrents fait-elle partie du périmètre de la refonte Kraftwerk (auquel cas il faudra concevoir cette fonctionnalité, absente du code actuel), ou reste-t-elle bien un service externe (Genesis/Bangles) hors périmètre, comme semble l'indiquer l'absence totale de trace dans le code Kraftwerk actuel ?

---

## H. Chiffrement : le guide décrit un mécanisme plus riche que le code n'en implémente 🟡 Important

- **Guide** (`pages/exports/expauto.md`) : décrit un chiffrement **symétrique** (usage interne INSEE) et **asymétrique** (obligatoire pour les envois externes, combiné au symétrique, clé publique nommée stockée dans Vault), plus une **signature électronique** optionnelle.
- **Code** : implémente uniquement un chiffrement symétrique de **l'archive ZIP complète** des sorties, via une clé AES stockée dans Vault (`02-fonctionnalites-principales.md` §8). Aucune trace de chiffrement asymétrique ni de signature.
- **Tests** : `out_data_encryption.feature` ne couvre que le cas symétrique, archive entière.

**Analyse** : cohérent avec le point G — la partie "chiffrement asymétrique + signature" documentée dans `expauto.md` concerne vraisemblablement le même mécanisme d'export automatisé externe à Kraftwerk, pas Kraftwerk lui-même. Le chiffrement propre à Kraftwerk (`withEncryption` sur les endpoints `/main*`) reste, lui, cohérent entre code, tests et le reste du guide (chiffrement symétrique, archive entière, sur demande).

**Proposition par défaut** : traiter ce point comme réglé par la réponse à la question G — si les exports planifiés sont hors périmètre Kraftwerk, le chiffrement asymétrique/signature l'est aussi. Sinon, à intégrer explicitement dans les règles métier de chiffrement de la cible.

---

## I. Données partielles (`questionnaireState` = `STARTED`) : documenté et présent en code, mais jamais testé 🟢 Mineur (gap de test, pas de contradiction)

- **Guide** : documente en détail le mécanisme `questionnaireState` (`FINISHED` vs `STARTED`) permettant de repérer les réponses partielles/non validées dans les exports (`pages/fonctionnement/entree.md`, `pages/fonctionnement/intgenesis.md`, `pages/exports/enqgenesis.md`, `pages/exports/jsonSIexterne.md`).
- **Code** : le champ existe bel et bien — `Constants.QUESTIONNAIRE_STATE_NAME`, porté par `SurveyUnitUpdateLatest.questionnaireState`, exploité aussi bien dans l'export JSON (`JsonWriterSequence.java:62`) que dans le chemin CSV/Parquet (mentionné dans la Javadoc d'`InformationLevelsProcessing` comme identifiant fixe à propager). Cohérent avec le guide.
- **Tests** : recherche exhaustive dans toute la suite Cucumber — **aucune occurrence** de `STARTED`, `FINISHED` ni de `questionnaireState`. Cette fonctionnalité documentée et présente en code n'est vérifiée par aucun test automatisé.

**Analyse** : pas une contradiction entre les trois sources — code et guide s'accordent — mais un vrai trou de couverture de tests sur une fonctionnalité qui semble importante (gestion des réponses partielles), à ne pas perdre de vue pour la cible.

**Proposition par défaut** : conserver cette fonctionnalité dans la cible (guide et code s'accordent) et prévoir des cas de test dédiés dès la conception, plutôt que de la redécouvrir plus tard.

---

## J. Nommage du dossier/fichier de sortie du reporting 🟢 Mineur

- **Guide** : le dossier de sortie est nommé `out\NOM_ENQUETE\date_REPORTING_DATA_ONLY` (`pages/exports/reporting.md`).
- **Code/Tests** : le(s) fichier(s) de sortie sont nommés `<campagne>_REPORTINGDATA.csv`/`.parquet` (sans les underscores intermédiaires du guide), confirmé par `do_we_export_reporting_data.feature`.

**Proposition par défaut** : écart de nommage sans enjeu fonctionnel — à harmoniser librement dans la convention de nommage de la cible.

---

## K. Incohérence interne au guide sur la taille de lot recommandée 🟢 Mineur

- `pages/exports/enqgenesis.md` recommande d'**augmenter** la taille de lot (jusqu'à 5000) pour les grosses enquêtes.
- `pages/exports/expauto.md` recommande au contraire de **réduire** la taille de lot (50 au lieu de 100) pour les grosses enquêtes.

**Analyse** : ce n'est pas un écart code/guide mais une incohérence interne au guide, sans lien avec le code. Sans enjeu pour la cible (le comportement optimal en taille de lot dépendra de la nouvelle architecture par `partitionId`), à noter simplement pour la reprise de la documentation utilisateur.

---

## L. Points confirmés cohérents entre les trois sources (pas d'action requise)

Pour rassurer sur la fiabilité globale du code : plusieurs points importants sont cohérents entre code, tests et guide, et n'appellent pas de décision :
- **`campaignId` (déprécié) / `questionnaireModelId` (actuel)** pour l'appel Genesis : cohérent entre code (`@Deprecated(since="3.4.1")`) et guide (mention only du `questionnaireModelId` comme paramètre courant, `campaignName` cantonné à une procédure d'intégration manuelle déjà marquée dépréciée côté Genesis). `partitionId` apparaît déjà comme un champ vide/inutilisé dans le schéma JSON du guide et dans `JsonWriterSequence.java` — cohérent avec la trajectoire déjà validée avec vous.
- **Algorithme `OUTCOME_SPOTTING`** : les valeurs et la cascade logique documentées dans le guide (`pages/exports/reporting.md` section 2) correspondent exactement à ce que le code implémente et à ce que les 9 scénarios de test vérifient.
- **Absence de boucles imbriquées** : le guide l'affirme explicitement ("il ne peut pas y avoir de boucles imbriquées"), et le code le confirme par une limitation documentée dans la Javadoc d'`InformationLevelsProcessing` ("un seul niveau de groupe sous la racine").
- **Chiffrement symétrique de l'archive entière** (hors périmètre "exports automatisés", cf. point H) : cohérent entre guide (`enqgenesis.md`/`enqhorsgenesis.md`, option "crypter le résultat"), code et tests.
- **Dépréciation de `/main/genesis/lunatic-only`** : annoncée dans le guide ("cette option sera bientôt supprimée") et déjà marquée `@Deprecated(since="3.4.1", forRemoval=true)` dans le code.

---

## Synthèse et suite

Les points bloquants A, B, C, D ont été tranchés (voir « Décisions arbitrées » en tête de document). Restent ouverts, mais non bloquants pour démarrer l'extraction des règles métier :
- **G / H** (exports automatisés `/contexts/schedules/v2` + chiffrement asymétrique/signature) : à confirmer comme **hors périmètre Kraftwerk** (probablement Genesis/Bangles) — hypothèse par défaut retenue tant qu'aucun élément contraire n'apparaît.
- **E / J / K** : simples écarts de nommage/documentation, sans arbitrage nécessaire, à absorber dans les conventions de la cible.
- **I** (données partielles `STARTED`/`FINISHED`) : conservé, à couvrir par des cas de test dédiés.

Prochaine étape : **extraction des règles métier, cas nominaux et edge cases**, en intégrant les décisions ci-dessus (périmètre Genesis-only par `partitionId`, liens 2à2 conservés, fusion multimode automatique conservée, paradonnées exclues, pas de vocabulaire VTL).
