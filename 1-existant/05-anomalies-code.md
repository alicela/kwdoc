# Kraftwerk - Anomalies et points de vigilance internes au code

Ce document liste des incohérences **internes au code lui-même** (pas des écarts code/tests/guide utilisateur — ceux-ci seront traités dans `04-comparaison-code-tests-guide.md`, à reprendre dans une phase ultérieure). Ce sont des observations factuelles issues de la lecture du code, pas des recommandations de correction : elles sont à trancher au moment de définir le comportement cible.

Pour chaque point : où c'est dans le code, ce qui semble en découler, et pourquoi ça mérite une décision explicite pour la refonte plutôt qu'une reprise telle quelle.

---

## 1. Le mécanisme de statut de job `InMemoryJobStore` semble non fonctionnel

- `MainAsyncService.getStatus` porte l'annotation `@GetMapping("/status/{jobId}")` mais la classe est annotée `@Service`, pas `@Controller`/`@RestController`, et n'a pas de `@RequestMapping` de classe.
- Spring n'enregistre les méthodes `@*Mapping` que sur des beans porteurs d'un stéréotype contrôleur — cette route ne semble donc jamais réellement exposée en HTTP.
- **Conséquence probable** : les 5 endpoints `/main`, `/main/file-by-file`, `/main/lunatic-only`, `/main/genesis` (déprécié), `/main/genesis/lunatic-only` (déprécié) renvoient un `jobId`, mais rien ne permet d'en consulter le statut via `InMemoryJobStore`. Seul `/main/genesis/by-questionnaire` (via `InMemoryExportJobStore` + `GET /jobs/{jobId}`) a un point de consultation qui fonctionne.
- Non vérifié à l'exécution dans cette passe — à confirmer en démarrant l'application et en appelant `GET /status/{jobId}` avant de considérer ce point comme acquis pour la refonte.

## 2. Incohérence d'enregistrement de job entre deux endpoints jumeaux

- `/main/genesis/by-questionnaire` (`MainService.java:202`) appelle `exportJobStore.start(jobId)` avant de dispatcher le traitement asynchrone.
- `/main/genesis/by-questionnaire/lunatic-only` (`MainService.java:271`), qui suit pourtant le même contrat de retour (jobId consultable via `/jobs/{jobId}`), **n'appelle pas** `start()`.
- Résultat : `GET /jobs/{jobId}` renverrait 404 ("uuid inconnu") pour un job de ce second endpoint pourtant réellement en cours d'exécution.

## 3. Aucune gestion centralisée des exceptions

- Aucun `@ControllerAdvice`/`@ExceptionHandler` dans tout le dépôt.
- Chaque contrôleur reproduit le même motif `catch (KraftwerkException e) { return ResponseEntity.status(e.getStatus()).body(e.getMessage()); }`.
- Plusieurs méthodes déclarent `throws KraftwerkException`/`throws SQLException`/`throws Exception` sans l'attraper sur le chemin final du traitement (`StepByStepService.unimodalProcessing/multimodalProcessing/writeOutputFiles`, `MainService.getJobStatus`, `SplitterService.saveResponsesFromXmlFile`) : dans ces cas, le code HTTP réellement renvoyé au client ne correspond pas à `e.getStatus()` mais au comportement par défaut de Spring Boot pour une exception non gérée (généralement 500).

## 4. `HealthcheckService` : deux défauts de robustesse

- `fileStorageExists()` calcule `Files.exists(...)` mais **le résultat booléen n'est jamais utilisé** — seule une exception levée par `Paths.get`/`Files.exists` est catchée pour signaler une déconnexion. Un répertoire simplement absent (sans lever d'exception) serait donc rapporté comme `OK` à tort.
- `client.pingGenesis()` est appelé sans try/catch dans `healthcheck()` : si Genesis est injoignable, l'exception n'est pas absorbée et remonte comme une erreur serveur non gérée, au lieu du statut "FAILURE" dégradé que l'endpoit est censé produire.

## 5. Paramètres lus mais jamais utilisés

- `withAllReportingData` (paramètre des deux endpoints `PUT /steps/buildVtlBindings*`) est lu depuis la requête mais n'intervient nulle part dans le corps des méthodes correspondantes.

## 6. Contrôle probablement mort dans `ReportingDataService`

- `processReportingDataGenesis` contient un contrôle explicite "mode not specified" pour un `mode` null, mais le paramètre est déclaré `@RequestParam Mode mode` sans `required=false` — Spring rejette donc déjà la requête (erreur 400 native) avant que ce contrôle applicatif ne puisse s'exécuter.

## 7. `ColumnNotFoundException` : exception déclarée sans point de levée

- La classe existe dans `kraftwerk-core/exceptions/` mais aucun `throw new ColumnNotFoundException(...)` n'a été trouvé dans le dépôt. À confirmer si elle est réellement mort-code ou levée via un mécanisme non détecté par recherche textuelle (réflexion, génération de code...).

## 8. VTL : la boucle d'évaluation absorbe même les `Error` Java

- `VtlExecute.evalVtlScript` catche `Exception` **et `Error`** par instruction VTL, journalise, et continue l'exécution du reste du pipeline. Une portée de `catch` incluant `Error` est inhabituelle (elle capturerait par exemple un `OutOfMemoryError` partiel) — à documenter comme un choix conscient ou à revoir explicitement pour la cible.

## 9. Deux représentations indépendantes de la hiérarchie de groupes/boucles

- Le modèle de métadonnées original (`MetadataModel.Group`, fourni par la librairie externe `fr.insee.bpm`) et sa reconstruction ad hoc par `InformationLevelsProcessing`/`VtlBindings.getDatasetVariablesMap`, qui reparse les noms de colonnes séparés par des points pour retrouver la structure de groupes.
- Ces deux représentations ne restent synchronisées que par une convention de caractère séparateur (`.`, `Constants.METADATA_SEPARATOR`). Un nom de variable ou de groupe métier contenant ce caractère romprait silencieusement le parsing, sans erreur explicite.
- `InformationLevelsProcessing` documente elle-même sa propre limite : elle ne gère qu'un seul niveau de groupe sous la racine.

## 10. Logique spécifique à une enquête mêlée à du code générique

- `LunaticXmlDataParser.manageTcmLiens` code en dur la gestion de la boucle `BOUCLE_PRENOMS` et de sa matrice de liens familiaux 2-à-2 (limite fixe de 21 liens), à l'intérieur d'un parseur par ailleurs générique. Ce n'est pas un bug, mais un couplage fort entre une enquête particulière (probablement Emploi/ménages) et le moteur générique. **Mise à jour (client, 2026-07-08)** : ce traitement a été **déporté dans le module BPM** ; ce code Kraftwerk est donc obsolète et remplacé par BPM. En cible Genesis-only, la matrice de liens est fournie via BPM et ce couplage disparaît du périmètre Kraftwerk (voir `../regles-metiers/03-liens-2a2.md`).
- De même, les 16 fichiers `.vtl` de `TCMSequencesProcessing` (`src/main/resources/vtl/tcm/`) encodent des règles métier propres à certaines enquêtes, chargées automatiquement selon le nom de séquence DDI.

## 11. `LunaticJsonDataParser` : message de log permanent suspect

- Le parseur JSON Lunatic (par opposition au parseur XML) logue systématiquement `"Lunatic data parser being implemented!"` à chaque exécution et ne semble pas gérer les boucles/groupes de la même façon que `LunaticXmlDataParser.addGroupVariables` (seulement des réponses à plat). Ambigu : reflète peut-être une fonctionnalité réellement incomplète, ou un message de log obsolète resté dans le code. À clarifier avec l'équipe avant la refonte plutôt que de sur-interpréter.

## 12. README vs comportement réel du mode batch

- Le README documente des arguments positionnels et une valeur de service `LUNATIC_ONLY` qui n'existe plus dans le code (arguments nommés `--clé=valeur`, enum `KraftwerkServiceType` = `MAIN, GENESIS, FILE_BY_FILE, JSON`, "Lunatic-only" obtenu via `--with-ddi=false`). Ceci est un écart documentation/code à traiter formellement dans `04-comparaison-code-tests-guide.md`, mais il est mentionné ici aussi parce qu'il concerne un fichier interne au dépôt (le README), pas le guide utilisateur externe.

## 13. Absence de dépendance Parquet explicite identifiée

- Aucune dépendance Maven Parquet dédiée n'a été trouvée dans `kraftwerk-core/pom.xml`. L'écriture Parquet passe très vraisemblablement par les capacités natives de DuckDB (`COPY ... TO ... (FORMAT PARQUET)`), mais ce point mérite une vérification ciblée de `outputs/parquet/ParquetOutputFiles.java` si le format Parquet est un enjeu pour la cible (contraintes de schéma, compatibilité avec les outils avals).

---

## Pourquoi ce document existe séparément

Ces anomalies sont indépendantes des questions de choix cible ou de comparaison avec le guide utilisateur : ce sont des faits vérifiables dans le code actuel. Elles sont utiles à connaître avant de décider, pour chaque comportement, s'il doit être :
- reproduit à l'identique dans la refonte (si un usage réel en dépend, même de façon non documentée),
- corrigé silencieusement (bug clair, sans valeur métier),
- ou explicitement retiré (couplage spécifique à une enquête qu'on ne veut plus généraliser).

Chacun de ces points devra être validé avec vous avant la phase de conception cible.
