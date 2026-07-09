# Kraftwerk — Audit de dette technique & garde-fous pour la refonte

**Date** : 2026-07-09  |  **Périmètre** : code existant `v4.1.3` (`/home/onyxia/work/Kraftwerk`)  |  **Health Score : 44/100** (santé faible — refonte justifiée)  |  **Tendance** : ↘ (dette structurelle, non résorbée par les correctifs de version)

> Méthodologie : chaque constat est ancré dans le code par `fichier:ligne` (cf. `feedback-kraftwerk-analysis-methodology`). Les métriques (longueur de méthode, profondeur d'imbrication, nombre de méthodes/champs) proviennent d'heuristiques de comptage d'accolades — traiter ±1 comme du bruit. Ce document ne prescrit pas de corriger le code existant : il **quantifie la dette pour justifier la refonte** et **dérive les garde-fous** empêchant de la reconstituer dans la cible.

---

## ⚠️ Correction préalable d'une analyse antérieure

Le document `03-choix-architecturaux-et-problemes.md` (rédigé par « Mistral Vibe », non vérifié ligne à ligne) recommande en **priorité urgente** de « passer sur Spring Boot 3.x LTS » et « JDK 21 LTS ». **Cette recommandation contredit une décision client confirmée** (Java 25 + Spring Boot 4.1.0, cf. `project-kraftwerk-refonte-scope` et `3-chemin-vers-la-cible/annexe-02-synthese-executive-initiale.md`). Elle **ne doit pas être reprise**. La stack moderne est un choix assumé ; le présent audit la traite comme un acquis, pas comme une dette. Les autres constats de `03-` restent pertinents et sont réutilisés ci-dessous après vérification.

---

## 1. Résumé exécutif

Kraftwerk est un projet Maven 5 modules (~27 000 lignes Java, dont ~16 000 dans `kraftwerk-core`). La dette n'est **pas** dans la stack technique (récente, saine) mais dans **trois foyers structurels** : (1) un couplage VTL/Trevas diffusé dans tout le cœur métier, (2) une couche API dupliquée, mal outillée en gestion d'erreurs et au suivi de jobs partiellement non fonctionnel, (3) une base de tests fonctionnels arrimée au disque local — un mode que la cible supprime.

La bonne nouvelle : **la majorité de cette dette est déjà adressée par des décisions de refonte confirmées** (suppression VTL, Genesis-only, job tracking persistant unifié). Le risque n'est donc pas de « ne pas savoir quoi corriger » mais de **reconstituer la même dette faute de garde-fous**. C'est l'objet de la partie 2.

**Health Score : 44/100.** Un score bas est ici *attendu et cohérent* : il chiffre l'écart que la réécriture doit combler. Il sert de ligne de base pour mesurer la cible.

---

## 2. Radar par catégorie

| Catégorie | Poids | Sévérité dette (/100) | Pénalité | Constat dominant |
|---|---|---|---|---|
| Code | 25 % | 55 | 13,75 | Couplage VTL (99 réf. / 31 fichiers), duplication API, 27 méthodes > 50 lignes |
| Architecture | 20 % | 65 | 13,0 | Couplage fort `MainProcessing`, double mode API/batch, job stores non persistants (dont 1 mort) |
| Tests | 20 % | 60 | 12,0 | Aucun seuil de couverture, tests fonctionnels couplés au disque, modules entiers exclus de la mesure |
| Dépendances | 15 % | 55 | 8,25 | Deux POM racine divergents (SB 4.1.0 vs 4.0.6), Trevas à retirer, `json-simple` abandonnée |
| Documentation | 10 % | 40 | 4,0 | README batch obsolète, config sécurité non documentée |
| Infrastructure | 10 % | 55 | 5,5 | CI ne teste pas le chiffrement, pas de scan de vulnérabilités, config en dur |
| **Total** | **100 %** | — | **56,5** | **Health = 100 − 56,5 ≈ 44/100** |

---

## 3. Registre de dette priorisé

`Score = Impact × Probabilité ÷ Effort` (Impact/Proba/Effort sur 1–5 ; score élevé = agir tôt). Colonne **Refonte** : 🔴 supprimé par décision, 🔵 adapté/à re-concevoir sans VTL, 🟢 conservé, 🟠 garde-fou à créer. *L'effort est celui du traitement dans la cible.*

| ID | Catégorie | Dette (ancrage) | I | P | E | Score | Refonte |
|---|---|---|---|---|---|---|---|
| **D9** | Infra | Config en dur dans les *resources* de prod : `C:\Temp\kraftwerk\...` (`kraftwerk-api/.../application.properties:45-46`), URL Genesis prod en clair `http://api-reponses-enquetes.insee.fr` (`:49`) | 3 | 3 | 1 | **9,0** | 🟠 |
| **D2** | Deps | Deux POM racine divergents : `pom.xml` (SB **4.1.0**, 5 modules) vs `pom-public.xml` (SB **4.0.6**, 3 modules) — versions jacoco/pitest/sonar/cucumber désalignées ; la CI build `pom-public.xml`, l'image Docker build `pom.xml` | 4 | 4 | 2 | **8,0** | 🟠 |
| **D3** | Archi | Aucune gestion centralisée d'exceptions (0 `@ControllerAdvice`) ; le code HTTP réellement renvoyé ne correspond pas à `e.getStatus()` sur plusieurs chemins (`05-anomalies §3`) | 4 | 4 | 2 | **8,0** | 🔵 |
| **D1** | Code | **Couplage VTL/Trevas** : 11 imports `fr.insee.vtl.*` (8 fichiers), type `VtlBindings` référencé **99×** dans **31 fichiers**, package `core/vtl/` dédié, 2 dépendances `fr.insee.trevas:*:2.4.0` (`kraftwerk-core/pom.xml:81-90`) | 5 | 5 | 4 | **6,25** | 🔴🔵 |
| **D8** | Tests | Aucun seuil de couverture (pas de goal `check`) ; `api/client`, `api/configuration`, modules `functional-tests` & `encryption-tests` exclus de la mesure (`pom.xml:31-36`) ; ratio effectif API ≈ 0,12 | 4 | 3 | 2 | **6,0** | 🟠 |
| **D10** | Infra | CI sans scan de vulnérabilités dédié ; BPM reconstruit depuis `@master` flottant (`maven.yml:33`) ; upload couverture `fail-on-error:false` ; **modules de chiffrement non buildés/testés en CI** | 4 | 3 | 2 | **6,0** | 🟠 |
| **D15** | Code | Robustesse `HealthcheckService` (booléen `Files.exists` ignoré ; `pingGenesis()` sans try/catch) ; paramètre `withAllReportingData` lu mais jamais utilisé ; contrôle mort `mode==null` (`05-anomalies §4-6`) | 2 | 3 | 1 | **6,0** | 🔵 |
| **D4** | Archi | Suivi de jobs partiellement **non fonctionnel** : `InMemoryJobStore` jamais exposé en HTTP (`@Service` sans stéréotype contrôleur) ; endpoint `by-questionnaire/lunatic-only` n'appelle pas `start()` → 404 (`05-anomalies §1-2`) ; 2 stores en mémoire, non persistants | 4 | 4 | 3 | **5,3** | 🔵 |
| **D5** | Tests | Tests fonctionnels arrimés au disque : `TestConstants` en chemins relatifs (`kraftwerk-functional-tests/.../TestConstants.java:11-17`), `MainDefinitions.java:226` pilote le `MainProcessing` fichier-local **supprimé en cible** → 15 features à reconstruire sur stubs Genesis | 4 | 5 | 4 | **5,0** | 🔴🟠 |
| **D11** | Deps | Dépendances à risque : `json-simple:1.1.1` (abandonnée ~2012, `kraftwerk-core/pom.xml:60-64`), `okhttp` bloquée en 4.x pour minio (`:114`), `tomcat` surchargée manuellement « for security reasons » (`pom.xml:22-23`), `Saxon-HE:13.0` invérifiable | 3 | 3 | 2 | **4,5** | 🟠 |
| **D14** | Code | Code obsolète/mort : `manageTcmLiens` (`BOUCLE_PRENOMS` en dur, superseded par BPM), 16 `.vtl` TCM, `ColumnNotFoundException` jamais levée, log permanent `"Lunatic data parser being implemented!"` (`05-anomalies §7,10,11`) | 2 | 2 | 1 | **4,0** | 🔴 |
| **D6** | Code | `BatchExportService` = clone structurel de `MainService` : 6 paires de méthodes quasi-identiques + blocs de setup répétés 6-7× (`BatchExportService.java:137,202,266,330,388,445`) | 3 | 3 | 3 | **3,0** | 🔵 |
| **D7** | Code | `MainService.java` 554 lignes, constructeur 9 paramètres (`:70`) ; 12 méthodes/constructeurs à ≥ 7 paramètres (`WriterSequence`, `OutputFiles`, `MainAsyncService`) | 3 | 3 | 3 | **3,0** | 🔵 |
| **D12** | Code | Points chauds de complexité : `XformsDataParser.java:87` imbrication à 7 niveaux ; méthodes-parseurs longues (`ReportingDataParser.addReportingDataUEToQuestionnaire` 125 lignes `:168`) | 3 | 2 | 3 | **2,0** | 🔵 |
| **D13** | Archi | Deux représentations concurrentes de la hiérarchie groupes/boucles, synchronisées seulement par la convention du séparateur `.` ; `InformationLevelsProcessing` limitée à un seul niveau (`05-anomalies §9`) | 3 | 2 | 3 | **2,0** | 🔵 |

---

## 4. Matrice de priorisation

```
                 IMPACT ÉLEVÉ                    │           IMPACT FAIBLE
   ──────────────────────────────────────────────┼──────────────────────────────────
E  │  QUICK WINS (faire tôt / verrouiller)        │  AMÉLIORATION CONTINUE (backlog)
F  │   D9  config en dur                          │   D14 code mort/obsolète
A  │   D2  divergence POM                         │   D15 robustesse healthcheck
I  │   D3  gestion d'erreurs centralisée          │
B  │   D8  seuil de couverture                    │
L  │   D10 scan sécu CI + chiffrement en CI       │
E  ─────────────────────────────────────────────┼──────────────────────────────────
   │  PROJETS STRATÉGIQUES (planifier)            │  À NE PAS FAIRE ISOLÉMENT
É  │   D1  découplage VTL  ★ pilier de la refonte │   (traiter en flux de réécriture :
L  │   D4  job tracking persistant unifié         │    D6 duplication, D7 god class,
E  │   D5  reconstruction tests sur Genesis       │    D12 complexité, D13 hiérarchie)
V  │                                              │
É  │                                              │
```

### Top 5 (lecture stratégique, pas seulement score brut)

1. **D1 — Découplage VTL/Trevas** *(pilier)* : c'est l'objectif n°1 de la refonte et la dette la plus diffuse (99 réf. / 31 fichiers). Tout le reste s'articule autour. Réexpression en DuckDB SQL confirmée pour la réconciliation multimode et le découpage par niveau d'information.
2. **D9 — Config en dur** : ROI immédiat, effort minimal. Chemins Windows et URL prod ne doivent jamais réapparaître dans les *resources* livrées.
3. **D2 — Divergence des POM** : deux baselines Spring Boot (4.1.0 / 4.0.6) et un chiffrement non couvert par la CI = risque de « ça marche chez moi, pas en prod ». Un POM unique en cible.
4. **D3 — Gestion d'erreurs centralisée** : contrat HTTP actuellement faux sur plusieurs routes ; `@ControllerAdvice` unique = quick win à haute valeur, à poser dès le squelette API.
5. **D4 — Suivi de jobs** : l'implémentation actuelle est en partie inopérante (route jamais exposée, 404 possible) ; la cible impose déjà un suivi persistant unifié — à concevoir proprement, pas à migrer.

---

## 5. Détail par catégorie (ancré `fichier:ligne`)

### 5.1 Code (25 %)
- **VTL/Trevas (D1)** — imports directs : `core/outputs/ImportScript.java:7`, `core/outputs/TableScriptInfo.java:4`, `core/dataprocessing/ReconciliationProcessing.java:8-9`, `core/utils/SqlUtils.java:8-9`, `core/vtl/VtlExecute.java:10-11`, `core/vtl/VtlBindings.java:8-10`. Package `core/vtl/` entièrement orienté Trevas (`VtlBindings`, `VtlExecute`, `VtlJsonDatasetWriter`, `VtlScript`, `VtlMacros`, `ErrorVtlTransformation`). `VtlBindings` traverse les signatures publiques de `WriterSequence`, `OutputFiles`, tous les `*Processing` et `*Sequence` → **plus fort fan-out du dépôt, principal verrou de la réécriture**.
- **Duplication (D6)** — `BatchExportService` reproduit `MainService` sur 6 paires (`mainServiceBatch`≈`mainService`, `mainFileByFileBatch`≈`mainFileByFile`, etc.) ; bloc setup `getFileUtilsInterface()/getMainProcessing(...)` répété 6× (`MainService.java:101,126,150` ; `BatchExportService.java:147,212,276`), bloc Genesis 7×.
- **Méthodes longues (D12)** — 27 méthodes > 50 lignes ; pires : `ReportingDataParser.java:168` (125 l.), `ReconciliationProcessing.java:80` (109 l.), `MainProcessingGenesisNew.java:212` (107 l.).
- **Trop de paramètres (D7)** — 26 méthodes > 5 params ; pires : `MainService.java:70` (9), `MainAsyncService.java:53,92` (8), série `WriterSequence.java:35,48,64,82` (7).
- **Imbrication (D12)** — `XformsDataParser.java:87` atteint 7 niveaux de contrôle.
- **Nombres magiques** — codes HTTP en dur passés à `KraftwerkException` (`FileSystemImpl.java:87,169,185…`), constantes SAS non centralisées (`SASImportScript.java:45` `lrecl=13106`, `:61,80,97` `32`).
- **Code mort (D14)** — `manageTcmLiens` (obsolète, cf. BPM), `ColumnNotFoundException` jamais levée, log `"Lunatic data parser being implemented!"`. *Note : le scan de champs « non utilisés » est faussé par Lombok — ne pas traiter la liste brute comme du mort-code sans passe compilateur.*

### 5.2 Architecture (20 %)
- **Gestion d'erreurs (D3)** — aucun `@ControllerAdvice`/`@ExceptionHandler` ; motif `catch (KraftwerkException e)` recopié dans chaque contrôleur ; plusieurs `throws` non rattrapés sur le chemin final (`StepByStepService`, `MainService.getJobStatus`, `SplitterService`) → 500 par défaut au lieu du statut voulu (`05-anomalies §3`).
- **Jobs (D4)** — `MainAsyncService.getStatus` porte `@GetMapping` sur une classe `@Service` (jamais routée) ; `by-questionnaire/lunatic-only` omet `start()` → 404 (`05-anomalies §1-2`) ; deux stores en mémoire non persistants (`03- §4`).
- **Couplage `MainProcessing`** — dépendances directes vers `ControlInputSequence`, `BuildBindingsSequence`, `UnimodalSequence`, `MultimodalSequence`, `InsertDatabaseSequence`, `WriterSequence` ; traitement séquentiel, tout en mémoire (`03- §5`).
- **Double mode API+batch** dans un seul artefact (`03- §3`) ; **double modèle de hiérarchie** synchronisé par le séparateur `.` (D13, `05-anomalies §9`).

### 5.3 Tests (20 %)
- **Ratios** — core 0,42 (113/48) ; api 0,12 effectif (6 vraies classes de test / 49) ; `kraftwerk-encryption` 0 test en module.
- **Couplage disque (D5)** — `TestConstants.java:11-17` (chemins relatifs `functional_tests/in|out|temp`), `MainDefinitions.java:226` pilote le `MainProcessing` local ; ~82 Mo de fixtures versionnées ; **15 features en mode fichier-local → à reconstruire sur stubs Genesis** (`GenesisClientStub`).
- **Fragilité** — tests ordonnés à état partagé (`CsvOutputFilesTest.java:44` `@TestMethodOrder`), idiome « premier sous-répertoire » répété ~16× (`MainDefinitions.java:318,341,365…`), écritures en `target/tmp/` dépendantes du répertoire courant.
- **Outillage (D8)** — JaCoCo + PIT + Sonar configurés mais **aucun seuil** ; exclusions larges (`pom.xml:31-36`) — et la liste d'exclusions présente des virgules manquantes susceptibles de casser le parsing (à vérifier).

### 5.4 Dépendances (15 %)
- **Divergence POM (D2)** — `pom.xml:11-16` parent SB **4.1.0** vs `pom-public.xml:11-16` SB **4.0.6** ; jacoco 0.8.15/0.8.14, pitest 1.25.5/1.23.1, sonar 5.7/5.6, cucumber 7.34.4/7.34.3 ; CI build `pom-public.xml` (§5.5) mais Docker build `pom.xml`.
- **VTL (D1)** — `fr.insee.trevas:vtl-engine` + `vtl-jackson:2.4.0` (`kraftwerk-core/pom.xml:81-90`) à retirer.
- **À risque (D11)** — `json-simple:1.1.1` abandonnée (`:60-64`), `okhttp:4.12.0` bloquée pour minio (`:114`, commentaire « 5.x breaks minio »), `tomcat:10.1.55` surcharge manuelle sécurité (`pom.xml:22-23`), `Saxon-HE:13.0` (`:28-32`) invérifiable.
- **Gestion** — `lib_java_chiffrement.version` redéclarée dans deux modules au lieu du parent ; aucune version SNAPSHOT (bon point) ; `renovate.json` réduit au preset `config:recommended` (aucune règle projet).

### 5.5 Infrastructure (10 %)
- **CI (D10)** — 4 workflows (build/test/couverture/release automatisée). `maven.yml` build **uniquement `pom-public.xml`** → `kraftwerk-encryption`/`-tests` **jamais buildés ni testés en CI** ; BPM cloné/rebuild depuis `actions/checkout@master` (réf. flottante) ; upload couverture `fail-on-error:false`. **Aucun scan de vulnérabilités dédié** (OWASP/Trivy/Snyk/CodeQL) — repose sur Dependabot + Sonar + Renovate.
- **Docker** — `Dockerfile:2` image interne INSEE pinnée (rootless, bon point) mais tag non digest-pinné, mono-stage, pas de `HEALTHCHECK`.
- **Config (D9)** — `application.properties:45-46` chemins `C:\Temp\kraftwerk\...`, `:49` URL Genesis prod en clair (http), `:22` `size-limit=419430400` (400 Mo en dur — devient caduc en Genesis-only).

### 5.6 Documentation (10 %)
- README documente des arguments batch positionnels et une valeur `LUNATIC_ONLY` **qui n'existent plus** (args nommés `--clé=valeur`, enum `MAIN/GENESIS/FILE_BY_FILE/JSON`) (`05-anomalies §12`).
- Configuration de sécurité (OAuth2 resource server, endpoints protégés) non documentée (`03- §9`). *Point positif : la documentation de refonte (`documentation-kraftwerk/`) est, elle, en bonne voie.*

---

## 6. Partie 2 — Garde-fous anti-dette pour la refonte

La refonte étant décidée, l'enjeu est de **ne pas reconstituer** la dette ci-dessus. Chaque foyer se traduit en règle de conception + point de contrôle automatisé.

### 6.1 Correspondance dette → traitement cible

| Dette | Décision de refonte qui la traite | Garde-fou à mettre en place |
|---|---|---|
| D1 VTL/Trevas | 🔴 VTL externalisé — DuckDB SQL pour réconciliation & niveaux d'info | **Règle « zéro VTL »** : test d'archi interdisant tout import `fr.insee.vtl.*` / `fr.insee.trevas` ; aucune dépendance Trevas au POM |
| D3 gestion d'erreurs | 🔵 API réécrite | `@RestControllerAdvice` **unique** dès le squelette ; contrat HTTP couvert par tests `MockMvc` par code de statut |
| D4 job tracking | 🔵 suivi persistant unifié (décidé) | Store unique persistant survivant au restart ; test d'intégration `POST job → GET status` sur tous les endpoints asynchrones |
| D5 tests fichier-local | 🔴 mode « Hors Genesis » supprimé | Fixtures rebâties sur **stubs Genesis** ; interdiction de chemins disque relatifs dans les steps |
| D6/D7 duplication API + god class | 🔵 point d'entrée unique, batch = `CommandLineRunner` fin | Batch et API partagent le **même service** ; objet de contexte/builder au lieu de constructeurs à 7-9 params |
| D2 divergence POM | 🟠 (aucune décision ne la traite) | **POM racine unique** ; supprimer `pom-public.xml` ou le générer ; CI et image sur la **même** définition |
| D9 config en dur | 🟠 | Toute valeur d'environnement **externalisée** (properties + Vault) ; aucun chemin absolu ni URL prod dans les *resources* livrées |
| D8/D10 outillage qualité & sécu | 🟠 | Voir portes qualité §6.2 |

### 6.2 Portes qualité permanentes (à câbler dans la CI du nouveau dépôt)

1. **Seuil de couverture bloquant** — goal JaCoCo `check` avec minimum par module (proposition : lignes ≥ 70 % cœur métier), build **rouge** en dessous. Supprimer les exclusions larges héritées ; ne plus exclure les modules de tests de la *mesure* sans justification.
2. **Test d'architecture** (ArchUnit ou équivalent) — interdit : import `fr.insee.vtl.*`/`trevas`, dépendance d'une couche `infra`→`api`, `@GetMapping`/`@PostMapping` sur une classe non-contrôleur (le bug D4 aurait été attrapé), cycles de packages.
3. **Scan de vulnérabilités + SBOM (supply-chain)** — génération d'un **SBOM CycloneDX** au build + scan dépendances/image (OWASP dependency-check / Trivy / Grype / CodeQL) en CI, bloquant sur sévérité haute ; supprimer `fail-on-error:false` sur la couverture. Voir la spec technique cible (D-21).
4. **Un seul chemin de build** — CI build exactement l'artefact livré (fin de la divergence `pom.xml`/`pom-public.xml`) et **teste le chiffrement**. Épingler BPM à une **version publiée**, pas à `@master`.
5. **Aucune valeur d'environnement en dur** — lint/test vérifiant l'absence de chemins absolus et d'URL de prod dans `src/main/resources`.
6. **Politique de dépendances** — `renovate.json` enrichi (groupes, `vulnerabilityAlerts`, `dependencyDashboard`) ; sortir `json-simple`, réévaluer la contrainte `okhttp`↔minio, documenter chaque pin de sécurité (tomcat) avec date de revue.

### 6.3 Anti-patterns à ne pas rejouer
- Ne pas dériver `Batch*` en clone de `Main*` : un seul service, deux adaptateurs minces.
- Ne pas propager un type de framework (hier `VtlBindings`, demain un équivalent) dans 30+ signatures publiques : isoler le moteur SQL derrière une frontière étroite.
- Ne pas maintenir deux modèles de la hiérarchie groupes/boucles synchronisés par convention textuelle : **un seul** modèle de métadonnées (celui de BPM) porté de bout en bout.
- Ne pas versionner des dizaines de Mo de fixtures pilotant des E2E disque : jeux de tests légers sur stubs Genesis.

---

## 7. Plan d'action

Séquencé selon le cycle de réécriture (pas de « sprints de remédiation » sur le legacy, voué au remplacement).

### Avant / au démarrage du squelette cible (verrouillage — quick wins)
- Poser **POM racine unique**, `@RestControllerAdvice` unique, externalisation config, POM CI = POM livré (D2, D3, D9).
- Câbler les **portes qualité** §6.2 dès le premier commit (seuil couverture, ArchUnit « zéro VTL », scan sécu). *Les poser après coup coûte 10×.*

### Pendant la réécriture (projets structurants)
- **Découplage VTL → DuckDB SQL** (D1) : réconciliation multimode + découpage par niveau d'information, derrière une frontière étroite. Critère de succès : 0 import `fr.insee.vtl.*`.
- **Job tracking persistant unifié** (D4) : store unique, survit au restart, testé sur tous les endpoints asynchrones.
- **Reconstruction des tests fonctionnels sur stubs Genesis** (D5) : au fil de chaque fonctionnalité réécrite, pas en fin de projet.
- Point d'entrée unique API/batch (D6/D7) ; modèle de métadonnées unique (D13).

### Portes permanentes (après mise en service)
- Renovate enrichi + revue trimestrielle des pins de sécurité (tomcat, okhttp) (D11).
- Suivi du Health Score cible sur cette même grille comme indicateur de non-régression de la dette.

### Critères de succès mesurables
| Indicateur | Base (v4.1.3) | Cible |
|---|---|---|
| Imports `fr.insee.vtl.*` / réf. `VtlBindings` | 11 / 99 | **0 / 0** |
| Seuil de couverture bloquant | absent | présent (≥ 70 % cœur) |
| Baselines Spring Boot concurrentes | 2 (4.1.0 / 4.0.6) | **1** |
| Modules non testés en CI | 2 (chiffrement) | **0** |
| Scan de vulnérabilités bloquant | absent | présent |
| Config env. en dur dans `src/main/resources` | ≥ 2 (chemin, URL) | **0** |

---

## Annexe — Métriques brutes

- **Volumétrie** : kraftwerk-core 161 fichiers / 15 961 l. ; kraftwerk-api 58 / 7 003 ; kraftwerk-encryption 2 / 111 ; kraftwerk-encryption-tests 9 / 1 676 ; kraftwerk-functional-tests 14 / 2 234. Marqueurs `TODO/FIXME/HACK/XXX` : 12.
- **Code** : 27 méthodes > 50 l. ; 1 fichier > 500 l. (`MainService.java` 554) ; 6 god classes ; 26 méthodes > 5 params ; imbrication max 7 niveaux.
- **Tests** : 20 fichiers `.feature` ; ratios test/source core 0,42 / api 0,12 ; ~82 Mo de fixtures ; 7 `@SpringBootTest` ; 0 `@Disabled`.
- **Dépendances** : ~24 déclarées ; 0 SNAPSHOT ; 4 internes INSEE (trevas, bpm, lib_java_chiffrement, modules internes).
- **Sources de l'audit** : exploration code parallèle ancrée `fichier:ligne` (dimensions Code / Deps+Infra / Tests) + `1-existant/03-` (corrigé, cf. avertissement) et `05-anomalies-code.md`.
