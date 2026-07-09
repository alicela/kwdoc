# Kraftwerk - Fonctionnalités Principales (Existant, vérifié sur le code)

## Statut de ce document

Révision complète de la première passe (non vérifiée). Toutes les informations ci-dessous sont sourcées directement dans le code (`fichier:ligne`). Les zones d'incertitude sont signalées explicitement.

---

## Vue d'ensemble

Kraftwerk traite des données d'enquêtes multimodes (DDI + Lunatic) pour produire des tables exploitables statistiquement (CSV/Parquet), avec :
- un mode **fichiers locaux** (répertoire d'entrée avec `kraftwerk.json`),
- un mode **Genesis** (source de données via API REST amont),
- une **extraction JSON** simple depuis Genesis (sans le pipeline complet),
- un traitement dédié pour les **données de reporting** (suivi de collecte),
- un mode **batch** (CLI) redondant fonctionnellement avec l'API REST.

---

## 1. Inventaire complet des endpoints REST

Contrôleurs : `kraftwerk-api/src/main/java/fr/insee/kraftwerk/api/services/{KraftwerkService, ReportingDataService, MainService, StepByStepService, SplitterService, HealthcheckService}.java` (+ `services/async/`).

**Aucun `@ControllerAdvice`/`@ExceptionHandler` n'existe dans le dépôt** (vérifié par recherche exhaustive). Chaque méthode qui gère `KraftwerkException` (exception métier générique portant un `int status` + `String message`) le fait manuellement via `ResponseEntity.status(e.getStatus()).body(e.getMessage())`.

| Méthode | Chemin | Contrôleur:ligne | Dépréciation | Paramètres | Réponse | Remarques |
|---|---|---|---|---|---|---|
| PUT | `/main` | `MainService.java:91` | non | `inDirectoryParam` (requis) ; `archiveAtEnd`=false ; `withEncryption`=false ; `addStates`=false | 202, corps = jobId (UUID) | Asynchrone via `MainAsyncService.runWithoutGenesis` → `MainProcessing.runMain()`. Les erreurs de l'exécution async ne sont **pas** renvoyées sur cette réponse HTTP, seulement dans le job store. |
| PUT | `/main/file-by-file` | `MainService.java:115` | non | idem `/main` | 202, jobId | `fileByFile=true` |
| PUT | `/main/lunatic-only` | `MainService.java:140` | non | idem `/main` | 202, jobId | `withDDI=false` |
| PUT | `/main/genesis` | `MainService.java:171` | **oui**, `@Deprecated(since="3.4.1", forRemoval=true)` | `campaignId` (requis) ; `batchSize`=1000 ; `withEncryption`=false | 202, jobId | Traité par `MainProcessingGenesisLegacy` |
| PUT | `/main/genesis/by-questionnaire` | `MainService.java:190` | non | `questionnaireModelId` (requis) ; `dataMode` (optionnel) ; `batchSize`=1000 ; `withEncryption`=false ; `addStates`=false | 202, jobId | `exportJobStore.start(jobId)` appelé **avant** dispatch async (ligne 202) |
| GET | `/jobs/{jobId}` | `MainService.java:213` | non | `jobId` (path) | 200 `ExportJobResultDto` | `KraftwerkException(400)` si l'UUID est invalide ; `(404)` si inconnu. **Ces exceptions ne sont pas catchées dans la méthode** (`throws KraftwerkException`) → elles remontent comme une erreur serveur générique, pas comme le statut HTTP porté par l'exception. |
| PUT | `/main/genesis/lunatic-only` | `MainService.java:234` | **oui**, `@Deprecated(since="3.4.1", forRemoval=true)` | `campaignId` (requis) ; `batchSize`=1000 ; `withEncryption`=false | 202, jobId | Legacy, `withDDI=false` |
| PUT | `/main/genesis/by-questionnaire/lunatic-only` | `MainService.java:254` | non | `questionnaireModelId` (requis) ; `dataMode` (optionnel) ; `batchSize`=1000 ; `withEncryption`=false ; `addStates`=false | 202, jobId | **N'appelle pas** `exportJobStore.start(jobId)` avant dispatch, contrairement à l'endpoint DDI équivalent ci-dessus — incohérence de suivi de job (voir `05-anomalies-code.md`) |
| GET | `/json` | `MainService.java:277` | non | `collectionInstrumentId` (requis) ; `dataMode` (optionnel) ; `batchSize`=1000 ; `sinceDate` (Instant, optionnel) ; `localSinceDate` (LocalDateTime, optionnel) ; `withEncryption`=false ; `addStates`=false | 200 `JsonExtractionResponse{message, sinceUtc, sinceLocal}` ; 204 si pas de données | **Synchrone** (pas de job). 400 si `sinceDate`/`localSinceDate` sont fournis de façon incohérente (mutuellement exclusifs) |
| GET | `/json/replay` | `MainService.java:347` | non | idem `/json` + `untilDate`/`localUntilDate` (optionnels, défaut = maintenant) ; une des deux dates de début est **requise** | 200 `JsonReplayResponse{message, localSinceDate, localEndDate}` | 400 si date de début manquante, ou si `endDate` avant `startDate` |
| POST | `/debug/json` | `MainService.java:436` | non (mais non documenté dans Swagger — `@Operation` vide) | `collectionInstrumentId`, `dataMode`, `batchSize`=1000, `withEncryption`=false, `addStates`=false ; corps `DebugJsonExportDto{interrogationIds}` | 200 `DebugJsonExportResultDto{collectionInstrumentId, successIds, errors}` | 400 si la liste d'ids est vide/nulle ; les échecs par id sont isolés (n'interrompent pas les autres) |
| PUT | `/reportingdata/main` | `ReportingDataService.java:47` | non | `campaignId` (requis) ; `reportingDataFilePath` (requis) | 200 texte | 404 si fichier absent, 400 si extension différente de `.xml`/`.csv` |
| PUT | `/reportingdata/genesis` | `ReportingDataService.java:59` | non | `campaignId` (requis) ; `reportingDataFileName` (requis) ; `mode` (Mode, requis) | 200 texte | Un contrôle "mode not specified" existe en code mais est probablement mort (Spring rejette déjà l'absence d'un `@RequestParam` non nullable avant d'atteindre ce contrôle) |
| PUT | `/steps/buildVtlBindings` | `StepByStepService.java:68` | non | `inDirectoryParam` (requis) ; `withAllReportingData`=true (**lu mais jamais utilisé dans le corps de la méthode**) | 200 | Étape unitaire du pipeline exposée séparément pour du débogage/pilotage manuel |
| PUT | `/steps/buildVtlBindings/{dataMode}` | `StepByStepService.java:122` | non | idem + `dataMode` (path, requis) | 200 | `withAllReportingData` également non utilisé |
| PUT | `/steps/unimodalProcessing` | `StepByStepService.java:174` | non | `inDirectoryParam`, `dataMode` (requis) | 200 | `throws KraftwerkException` non catché sur l'appel final |
| PUT | `/steps/multimodalProcessing` | `StepByStepService.java:231` | non | `inDirectoryParam` | 200 | idem |
| PUT | `/steps/writeOutputFiles` | `StepByStepService.java:289` | non | `inDirectoryParam` | 200 | `throws KraftwerkException, SQLException` non catchés sur l'appel final |
| PUT | `/steps/archive` | `StepByStepService.java:351` | non | `inDirectoryParam` | délègue à `KraftwerkService.archive()` | `KraftwerkException` catchée à chacune des 3 étapes internes |
| PUT | `/split/lunatic-xml` | `SplitterService.java:32` | non | `inputFolder`, `outputFolder`, `filename`, `nbResponsesByFile` (int), `fileSystemType` (tous requis) | 200 "File split" | Signature `throws Exception` (large, non catché) |
| GET | `health-check` | `HealthcheckService.java:42` | non | aucun | 200 texte multi-lignes (statut global, version, ping Genesis, stockage) | Voir `05-anomalies-code.md` : le ping Genesis n'est pas catché, l'existence du dossier de stockage est calculée mais son résultat jamais utilisé |

### Endpoints "steps" (`/steps/**`) — usage
Ces endpoints exposent chaque étape du pipeline `MainProcessing` séparément (construction des bindings VTL, traitement unimodal, multimodal, écriture des sorties, archivage), plutôt que le seul enchaînement complet via `/main`. Utile pour du pilotage pas-à-pas / débogage, mais duplique la logique d'orchestration de `MainProcessing.runMain()`.

### Classes de service non exposées en HTTP (à ne pas confondre avec des endpoints)
- `BatchExportService` : `@Service` simple, pas de `@*Mapping`. Appelée uniquement par `KraftwerkBatch` (mode CLI).
- `MainAsyncService.getStatus` (`@GetMapping("/status/{jobId}")`) : la classe est `@Service`, **pas** `@Controller`/`@RestController`, et n'a pas de `@RequestMapping` de classe. Il est très probable que cette route ne soit **jamais réellement enregistrée par Spring** — donc que le jobId renvoyé par `/main`, `/main/file-by-file`, `/main/lunatic-only` et les endpoints Genesis dépréciés n'ait **aucun moyen de consultation de statut fonctionnel** dans le code actuel. Ce point n'a pas été vérifié à l'exécution (à confirmer en démarrant l'application).
- `OutputZipService` : `@Service` utilisé en interne pour zipper/chiffrer les sorties, pas un endpoint.

---

## 2. Suivi des jobs asynchrones

Deux systèmes **indépendants et non interopérables**, tous deux de simples `ConcurrentHashMap` en mémoire (aucune persistance — **l'état des jobs est perdu à chaque redémarrage**) :

1. **`InMemoryJobStore`** — statuts `RUNNING/DONE/FAILED`. Alimenté par `/main`, `/main/file-by-file`, `/main/lunatic-only`, `/main/genesis` (déprécié), `/main/genesis/lunatic-only` (déprécié). Censé être consultable via `GET /status/{jobId}` — qui n'est probablement pas exposé (voir ci-dessus). **Effet net probable : pour ces 5 endpoints, le jobId retourné n'a pas de point de consultation de statut fonctionnel.**
2. **`InMemoryExportJobStore`** — statuts `RUNNING/DONE/PARTIAL/FAILED`. Alimenté par `/main/genesis/by-questionnaire` et par les méthodes batch (`BatchExportService`). Consultable via `GET /jobs/{jobId}` (le seul endpoint de statut réellement exposé). `/main/genesis/by-questionnaire/lunatic-only` alimente ce même store en résultat mais **sans appeler `start()`**, ce qui ferait retourner 404 par `/jobs/{jobId}` alors que le job tourne réellement.

Exécution asynchrone via un exécuteur Spring dédié `kraftwerkExecutor` (`ThreadPoolTaskExecutor`, cœur=4, max=8, file=100). Seuls `runWithoutGenesis`, `runWithGenesis` (déprécié) et `runWithGenesisByQuestionnaire` sont `@Async` ; les endpoints `/json`, `/json/replay`, `/debug/json` s'exécutent de façon synchrone sur le thread de la requête.

---

## 3. Pipeline de traitement principal (mode fichiers locaux, `/main`)

Orchestré par `kraftwerk-api`'s `MainProcessing` (`process/MainProcessing.java`), qui pilote des classes de `kraftwerk-core/sequence/` :

1. **Initialisation** (`MainProcessing.init()`) : résolution du répertoire d'entrée, recherche du fichier `kraftwerk.json` (config utilisateur obligatoire : `survey_data`, `data_mode`, `data_file`, `data_format`, `multimode_dataset_name`), chargement des métadonnées DDI+Lunatic (ou Lunatic seul si `withDDI=false`), éventuel découpage en un `UserInputsFile` par fichier pour le mode "file-by-file", et contrôle de la **limite de taille de 400 Mio exacts** (`419430400` octets, propriété `fr.insee.postcollecte.size-limit`) — au-delà, `KraftwerkException(413, ...)`.
2. **Traitement "unimodal"** (une fois par mode de collecte présent dans les données) :
   - `BuildBindingsSequence` : parsing du fichier de données brut par un parseur spécifique au mode (`DataParserManager`), parsing des paradonnées si présentes, écriture d'un dataset Trevas via `VtlExecute`.
   - `UnimodalSequence`, dans l'ordre : contrôle de cohérence de longueur des variables (vs attendu DDI) → dérivation des variables calculées Lunatic (VTL, si un fichier Lunatic existe) → renommage des variables en "noms qualifiés" par groupe (`GroupProcessing`) → VTL automatique spécifique au mode (fichier `unimode/<mode>.vtl` — **aucun fichier de ce type n'existe actuellement sur le disque**, donc cette étape n'a aujourd'hui aucun effet réel au-delà des instructions générées automatiquement par chaque classe de traitement de mode) → modules VTL "TCM" automatiques → VTL utilisateur optionnel par mode.
3. **Traitement "multimodal"** (`MultimodalSequence`) : réconciliation (union/jointure des datasets par mode en un dataset unique `MULTIMODE`) → suppression des datasets unimodaux des bindings → transformation VTL utilisateur optionnelle → découpage en datasets par niveau d'information (racine + boucles), voir §4.
4. **Insertion en base** (`InsertDatabaseSequence`) : chaque dataset Trevas restant est chargé dans une base **DuckDB en mémoire de processus** (`jdbc:duckdb:`) sous forme de table SQL.
5. **Écriture des sorties** (`WriterSequence`) : CSV et Parquet générés par des instructions SQL `COPY ... TO` DuckDB, un fichier par dataset, plus des scripts d'import R/SAS.
6. **Erreurs / logs** : fichiers d'erreurs et de logs écrits en fin de traitement.

Un chemin parallèle, moins exploré ici, existe pour la source Genesis : `ControlInputSequenceGenesis`, `BuildBindingsSequenceGenesis`, `UserInputsGenesis`, `MetadataUtilsGenesis`, piloté depuis `MainProcessingGenesisLegacy`/`MainProcessingGenesisNew` côté API.

---

## 4. Gestion des boucles / groupes (loops)

- La structure de groupes/boucles est dérivée du DDI (enrichi par Lunatic si présent), ou de Lunatic seul si `withDDI=false` — algorithme opaque à ce dépôt, il vit dans la librairie externe `fr.insee.bpm`.
- `GroupProcessing` renomme chaque variable non-racine en un nom qualifié séparé par points (ex. `INDIVIDUALS_LOOP.CARS_LOOP.CAR_COLOR`).
- `InformationLevelsProcessing` reconstruit ensuite la hiérarchie des groupes **en reparsant ces noms de colonnes à points** du dataset multimodal (pas à partir du modèle de métadonnées original) et génère du VTL pour produire un dataset par groupe : `RACINE` (racine) + un par sous-groupe/boucle (ex. `BOUCLE_PRENOMS`).
- **Limite documentée dans le code lui-même** : cette classe ne gère "qu'un seul niveau de groupe sous la racine" (commentaire de classe explicite) — les boucles imbriquées à plus d'un niveau ne sont pas prises en charge par ce générateur.
- Chaque dataset restant (hors datasets par mode et `MULTIMODE`) devient un fichier de sortie : `<nom-campagne>_<datasetName>.csv`/`.parquet`.
- Un cas **spécifique et codé en dur** existe en dehors de ce mécanisme générique : la matrice de liens familiaux 2-à-2 de la boucle `BOUCLE_PRENOMS` est construite directement en Java (pas via VTL généré depuis les métadonnées) dans `LunaticXmlDataParser.manageTcmLiens` — avec une limite fixe de 21 liens par individu (`MAX_LINKS_ALLOWED`). Ce code est donc spécifique à une enquête (probablement l'enquête Emploi/ménages), pas générique.
- Point de vigilance structurel : deux représentations indépendantes de la hiérarchie des groupes coexistent (le modèle `MetadataModel.Group` de `fr.insee.bpm`, et sa reconstruction par parsing de noms de colonnes à points) et ne restent synchronisées que par une convention de caractère séparateur (`.`) — un nom de variable ou de groupe contenant ce caractère romprait silencieusement le parsing.

---

## 5. VTL (Validation and Transformation Language) via Trevas

Le moteur Trevas (v2.4.0) est utilisé comme `ScriptEngine` nommé `"vtl"` (`vtl/VtlExecute.java`). Aucune extension VTL custom INSEE substantielle n'a été trouvée dans `kraftwerk-core` (juste un utilitaire trivial de concaténation de chaînes).

VTL sert, dans l'ordre d'exécution :
1. **Dérivation des variables calculées**, sourcées **uniquement depuis le questionnaire Lunatic JSON** (pas depuis le DDI). Résolution des dépendances entre variables calculées par un algorithme itératif à point fixe plafonné à **100 itérations** ; les variables encore non résolues après 100 passes sont abandonnées avec un warning.
2. **Restructuration** (renommage par groupe, extraction par niveau d'information) — ce n'est pas de la validation, c'est de la réorganisation de dataset.
3. **Réconciliation multimode** (`ReconciliationProcessing`, ~300 lignes) : fusion des N datasets par mode en un seul, ajout d'une variable d'identification du mode, harmonisation des types quand ils divergent entre modes.
4. **Modules TCM automatiques** (`TCMSequencesProcessing`) : chargement de fichiers `.vtl` statiques (16 fichiers dans `src/main/resources/vtl/tcm/`) sélectionnés selon le nom des séquences DDI — ce sont des règles métier spécifiques à des enquêtes ménages/emploi INSEE, pas des règles génériques.
5. **VTL utilisateur optionnel**, injecté à 4 points définis dans `kraftwerk.json` (`mode_specifications`, `reconciliation_specifications`, `transformation_specifications`, `information_levels_specifications`).

**Il n'existe pas de phase de "validation" VTL dédiée produisant un résultat pass/fail** dans ce code — le seul contrôle apparenté à de la validation est un contrôle de longueur en Java pur (pas en VTL).

**Gestion des erreurs** : chaque instruction VTL est évaluée indépendamment ; une erreur (y compris une `Error` Java, pas seulement une `Exception`) est capturée, journalisée et enregistrée comme "erreur de transformation" — **le reste du script et du pipeline continuent** ; les échecs ne sont visibles qu'a posteriori dans le fichier d'erreurs, jamais en interrompant le traitement.

---

## 6. Paradonnées (Paradata)

- Modélise la télémétrie d'interaction par unité enquêtée (Orchestrator/Moog), un fichier JSON par unité.
- Seuls les événements dont `idOrchestrator == "orchestrator-collect"` sont conservés. Les unités avec 2 événements ou moins sont écartées.
- Calcule des sessions/orchestrateurs (intervalles) via une machine à états sur les identifiants d'événements (`init-session`, `init-orchestrator-collect`, `agree-sending-modal-button-orchestrator-collect`, `logout-close-button-orchestrator-collect`).
- **Il n'y a pas de fichier de sortie dédié aux paradonnées** : les indicateurs calculés (durées de session/orchestrateur, dates, nombre de sessions, nombre de changements de réponse — uniquement suivi pour la variable `PRENOM`, codé en dur) sont injectés comme variables de réponse ordinaires directement sur les données de l'unité enquêtée, et ressortent donc comme colonnes du fichier de sortie principal.

---

## 7. Données de reporting

Deux formats d'entrée pris en charge : **XML** et **CSV** (le CSV a un en-tête fixe à 8 colonnes contrôlé strictement : `statut,dateInfo,idUe,idContact,nom,prenom,adresse,numeroDeLot`).

- Modélise : identifiants d'échantillon INSEE, états de contact (avec dates), tentatives de contact, résultat d'identification, commentaires, cause de clôture.
- **Un seul indicateur réellement calculé** : `OUTCOME_SPOTTING` — résultat de repérage, calculé par un algorithme qui diffère selon la configuration d'identification (`IASCO/HOUSEF2F`, `INDTEL/INTELNOR`, `INDF2F/INF2FNOR`), branché sur une cascade de switch Java (identification → accès → situation → catégorie → occupant, ou statut individuel selon le mode). Ce n'est pas une formule VTL, c'est un algorithme métier codé en Java.
- **Correction après vérification par les tests Cucumber** : contrairement à ce qu'une première lecture du code laissait penser, il existe bien un **fichier de sortie séparé** `<campagne>_REPORTINGDATA.csv`/`.parquet`, confirmé par `do_we_export_reporting_data.feature` (qui teste explicitement que `<campagne>_RACINE.csv` **n'existe pas** quand seul le traitement reporting est lancé). Le mécanisme réel : `ReportingDataParser.integrateReportingDataIntoUE` crée un groupe de métadonnées dédié `REPORTINGDATA` et y rattache les variables — ce groupe est ensuite traité comme un dataset à part entière par le même mécanisme générique d'écriture par dataset que celui utilisé pour la racine et les boucles (`<campagne>_<datasetName>.csv`). Ce n'est donc pas une fusion dans le fichier principal, mais un dataset indépendant qui suit le chemin d'écriture générique — nuance importante par rapport à la formulation initiale de ce document. Les paradonnées (§6), elles, sont bien rattachées directement au niveau racine (pas de groupe dédié créé) et ressortent donc comme colonnes de `_RACINE.csv`.

---

## 8. Chiffrement

- Le chiffrement porte sur **l'archive ZIP complète des sorties**, pas sur des fichiers ou colonnes individuels : `OutputZipService.encryptAndArchiveOutputs` zippe tout le répertoire de sortie, chiffre ce ZIP via `EncryptionUtils.encryptOutputFile`, écrit `<nom>.zip.enc`, puis supprime le répertoire en clair.
- Implémentation réelle dans le module séparé `kraftwerk-encryption` (`EncryptionUtilsImpl`), qui s'appuie sur la librairie externe INSEE `lib_java_chiffrement_core` et une clé AES stockée dans Vault (chemin `chiffrement/trust_key/aes_key`). L'algorithme de chiffrement lui-même n'est pas dans le code Kraftwerk.
- Un stub no-op (`EncryptionUtilsStub`) est utilisé quand le module `kraftwerk-encryption` n'est pas sur le classpath (profil `ci-public`).
- Les tests dédiés (`kraftwerk-encryption-tests`) vérifient exactement : que le ZIP chiffré n'est pas lisible comme un ZIP brut, et que le déchiffrement via la clé Vault restitue un ZIP valide contenant le CSV racine attendu — aucun test de chiffrement partiel/colonne n'existe.

---

## 9. Découpage de fichiers volumineux (`/split/lunatic-xml`)

Découpe un fichier **Lunatic XML** en plusieurs fichiers plus petits, **par nombre d'éléments `SurveyUnit`** (paramètre `nbResponsesByFile`), pas par taille en octets. Utilise un parsing streaming StAX pour limiter la consommation mémoire sur de gros fichiers XML. **Ce mécanisme est indépendant de la limite des 400 Mio** appliquée par `MainProcessing.init()` — ce sont deux garde-fous distincts, l'un pour découper manuellement de gros fichiers en amont, l'autre pour rejeter un traitement si les données dépassent la limite au moment de l'exécution.

---

## 10. Gestion des exceptions et erreurs (vue d'ensemble)

| Classe | Hérite de | Déclenchée quand |
|---|---|---|
| `KraftwerkException` | `Exception` | Cas générique, porte un code HTTP arbitraire fixé au point de levée (valeurs observées : 204, 400, 404, 413, 500, 502) |
| `NullException` | `KraftwerkException` (statut 500 fixe) | Fichier JSON/paradonnées null ou illisible |
| `NullLunaticFileException` | `Exception` | Accès aux variables calculées alors que le fichier Lunatic est absent |
| `MissingMandatoryFieldException` | `IllegalArgumentException` | Champ obligatoire manquant/vide dans `kraftwerk.json` |
| `UnknownDataFormatException` | `IllegalArgumentException` | Format de données/config non reconnu |
| `DatasetNotFoundException` | `RuntimeException` | Dataset VTL demandé absent des bindings |
| `ColumnNotFoundException` | `Exception` | **Déclarée mais aucun point de levée trouvé dans le code actuel — probablement morte** |
| `DuckDBError` | `KraftwerkError` (marqueur, pas une exception) | Échec d'une instruction SQL DuckDB, collecté (pas levé) pour le rapport d'erreurs final |

Aucune table de correspondance centralisée exception → code HTTP : chaque site d'appel choisit son statut au cas par cas.

---

## 11. Modes de collecte

Les valeurs de l'enum `Mode` (dans `kraftwerk-core/data/model/`) déterminent le parseur utilisé et interviennent dans la réconciliation multimode et le nommage des datasets par mode ; le détail exhaustif des valeurs de l'enum n'a pas été vérifié caractère par caractère dans cette passe et devra être confirmé (à croiser avec le guide utilisateur dans la phase suivante) plutôt que repris tel quel de l'ancienne documentation.

---

## Synthèse des écarts par rapport à la documentation précédente (mistral)

- La liste des endpoints était incomplète : `KraftwerkService` (archive), `StepByStepService` (5 endpoints "pas-à-pas"), `SplitterService`, `HealthcheckService` n'étaient pas mentionnés.
- `/main/genesis` et `/main/genesis/lunatic-only` sont dépréciés par `campaignId` ; le paramètre de remplacement est `questionnaireModelId`, pas un `partitionId` — cette dernière notion n'existe pas dans le code actuel (elle apparaît dans des échanges antérieurs sur la cible V1, à ne pas confondre avec l'existant).
- Il n'y a pas de fichier de sortie séparé pour le reporting ou les paradonnées : tout est fusionné dans le fichier de sortie principal.
- Le mode batch n'utilise pas d'arguments positionnels comme indiqué dans le README, mais des arguments nommés `--clé=valeur` ; la valeur de service `LUNATIC_ONLY` n'existe pas (c'est un flag combiné avec `MAIN`/`GENESIS`).
- Il n'existe pas de persistance des jobs, et le mécanisme de statut pour les jobs `InMemoryJobStore` (5 endpoints `/main*`) semble non fonctionnel (route non exposée).
