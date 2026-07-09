# Kraftwerk - Architecture Globale (Existant, vérifié sur le code)

## Statut de ce document

**Révision** : cette version remplace la première passe d'analyse (générée par un agent Mistral sans vérification systématique ligne à ligne). Chaque affirmation ci-dessous a été vérifiée directement dans le code source du dépôt `Kraftwerk/` (tag `v4.1.3`), avec référence `fichier:ligne`. Quand une information n'a pas pu être confirmée avec certitude, c'est indiqué explicitement plutôt que supposé.

- **Version analysée** : 4.1.3 (`pom.xml:8`, tag git `v4.1.3`)
- **Date de vérification** : 2026-07-08

---

## 1. Modules Maven et graphe de dépendances

```
kraftwerk (parent pom, packaging=pom, version 4.1.3)
├── kraftwerk-core            # logique métier, aucune dépendance interne
├── kraftwerk-encryption      # dépend de kraftwerk-core + lib externe INSEE
├── kraftwerk-api             # dépend de kraftwerk-core (+ kraftwerk-encryption par défaut)
├── kraftwerk-functional-tests   # dépend de kraftwerk-api + kraftwerk-core
└── kraftwerk-encryption-tests   # dépend de kraftwerk-api + kraftwerk-core + kraftwerk-encryption
```

- `kraftwerk-core` : base, ne dépend d'aucun autre module Kraftwerk.
- `kraftwerk-encryption` : dépend de `kraftwerk-core` et de la librairie externe INSEE `fr.insee.lib_java_chiffrement:lib_java_chiffrement_core:1.5.6` (`kraftwerk-encryption/pom.xml:22-45`).
- `kraftwerk-api` : dépend de `kraftwerk-core`. Le profil Maven `with-encryption` (actif par défaut, `kraftwerk-api/pom.xml:102-115`) ajoute la dépendance à `kraftwerk-encryption`. Un profil alternatif `ci-public` (non actif par défaut, lignes 116-122) construit l'API **sans** `kraftwerk-encryption` — build utilisé pour les environnements où le chiffrement Vault n'est pas souhaité/possible.
- `kraftwerk-functional-tests` et `kraftwerk-encryption-tests` importent le jar `kraftwerk-api` via un classifier `app-to-import` produit par `maven-jar-plugin` (`kraftwerk-api/pom.xml:64-78`).

Le module `kraftwerk-api` n'a **pas** de sous-module "batch" séparé : le mode batch est un package (`api/batch/`) à l'intérieur du module API, pas un artefact Maven distinct.

### Packages par module

```
kraftwerk-core/src/main/java/fr/insee/kraftwerk/core/
├── data/model/            # Mode, DataFormat, etc.
├── dataprocessing/        # étapes de traitement VTL (Group, Calculated, Reconciliation, TCM, InformationLevels...)
├── encryption/            # interface EncryptionUtils + stub no-op
├── errors/                # KraftwerkError (marker) + DuckDBError
├── exceptions/            # exceptions métier
├── extradata/paradata/    # modèle + parsing des paradonnées
├── extradata/reportingdata/ # modèle + parsing des données de reporting (XML/CSV)
├── inputs/                # UserInputsFile, ModeInputs, UserInputsGenesis
├── metadata/              # MetadataUtils (DDI/Lunatic), MetadataUtilsGenesis
├── outputs/csv/           # écriture CSV
├── outputs/parquet/       # écriture Parquet
├── parsers/               # DataParserManager, LunaticXmlDataParser, LunaticJsonDataParser...
├── rawdata/                # SurveyRawData et structures brutes
├── sequence/              # orchestration : ControlInputSequence, BuildBindingsSequence, UnimodalSequence, MultimodalSequence, InsertDatabaseSequence, WriterSequence (+ variantes *Genesis)
├── utils/                 # SqlUtils (DuckDB), fichiers (FileUtilsInterface/FileSystemImpl/MinioImpl), xml, Constants, KraftwerkExecutionContext
└── vtl/                   # VtlExecute, VtlBindings, VtlJsonDatasetWriter, VtlMacros

kraftwerk-api/src/main/java/fr/insee/kraftwerk/
├── KraftwerkApi.java       # classe principale Spring Boot
└── api/
    ├── batch/              # KraftwerkBatch (ApplicationRunner), ArgsChecker, KraftwerkServiceType
    ├── client/             # GenesisClient, OidcService, GenesisAuthInterceptor
    ├── configuration/      # ConfigProperties, MinioConfig, VaultConfig, OpenApiConfiguration, AsyncConfig, TimeConfiguration, LogRequestFilter
    │   └── security/       # NoAuthConfiguration, OIDCAuthConfiguration, RoleConfiguration, ApplicationRole, SecurityTokenProperties
    ├── dto/                # DTOs de requête/réponse REST
    ├── process/            # MainProcessing, MainProcessingGenesisLegacy, MainProcessingGenesisNew, ReportingDataProcessing
    ├── services/           # contrôleurs REST (voir 02-fonctionnalites-principales.md)
    │   └── async/          # InMemoryJobStore, InMemoryExportJobStore, MainAsyncService
    └── utils/

kraftwerk-encryption/src/main/java/fr/insee/kraftwerk/encryption/
├── utils/EncryptionUtilsImpl.java   # implémentation réelle (Vault + lib_java_chiffrement)
└── VaultCallerImpl.java             # authentification Vault (AppRole)
```

> Remarque : le modèle de métadonnées DDI/Lunatic lui-même (`VariablesMap`, `Variable`, `Group`, `MetadataModel`, lecteurs DDI/Lunatic) **n'est pas dans ce dépôt** : il vit dans la dépendance externe INSEE `fr.insee.bpm:bpm:1.2.2` (`kraftwerk-core/pom.xml` lignes 20 et 132). Kraftwerk-core consomme cette librairie sans en réimplémenter la logique.

---

## 2. Point d'entrée et sélection du mode (API vs batch)

`kraftwerk-api/src/main/java/fr/insee/kraftwerk/KraftwerkApi.java` :

- Classe `@SpringBootApplication`, `@ConfigurationPropertiesScan`, `@EnableConfigurationProperties(MinioConfig.class)`, qui étend `SpringBootServletInitializer` (lignes 14-18) — l'application peut donc aussi être déployée en WAR.
- **Le choix API/batch se fait uniquement en scannant les arguments bruts** passés au `main()` (lignes 26-49) : si un argument commence par `--service=`, le mode batch est activé. Il n'y a pas de propriété Spring dédiée pour ce choix.
  - Mode batch : `app.run(args).close()` (ligne 44) — le contexte Spring démarre, exécute le traitement, puis se ferme. L'application tourne une fois puis s'arrête.
  - Mode API : `app.run(args)` (ligne 47) reste démarré comme service web classique.
- Si aucun argument ne commence par `--spring.profiles.active=`, le profil `dev` est activé par défaut (lignes 27-28, 35-37). **Aucun fichier `application-dev.properties`/`.yml` n'existe dans le dépôt** : activer ce profil n'a aujourd'hui aucun effet sur les propriétés (seul `EncryptionUtilsImpl` est conditionné par un profil, `@Profile("!ci-public")`, sans lien avec `dev`).
- Le travail effectif du mode batch est délégué à `KraftwerkBatch` (`ApplicationRunner`), pas au `main()` lui-même.

### Mode batch (`api/batch/KraftwerkBatch.java`)

Arguments CLI reconnus (constantes lignes 42-51), tous passés en `--clé=valeur` :

| Option | Description | Défaut |
|---|---|---|
| `--service=` | `MAIN`, `GENESIS`, `FILE_BY_FILE`, `JSON` (enum `KraftwerkServiceType`) | requis |
| `--with-ddi=` | traite avec DDI ou en mode Lunatic-only | `true` |
| `--reporting-data=` | active le traitement des données de reporting | — |
| `--reporting-data-file-path=` | requis si `reporting-data=true` | — |
| `--questionnaireId=` | identifiant questionnaire | requis |
| `--with-encryption=` | active le chiffrement | — |
| `--extract-json-since=` | instant ISO, utilisé seulement pour le service `JSON` | — |
| `--add-states=` | ajoute les états de traitement | — |
| `--batch-size=` | taille de lot | `1000` |
| `--mode=` | doit correspondre à l'enum `Mode` (`kraftwerk-core`) | — |

Les options inconnues sont journalisées en warning, pas rejetées. Toute valeur de `--service=` en dehors de `MAIN/GENESIS/FILE_BY_FILE/JSON` lève une `IllegalArgumentException` (`ArgsChecker.checkServiceName`).

**⚠️ Écart documentation/code** : le `README.md` (lignes 26-34 et son miroir français 53-61) décrit des arguments **positionnels** (`service, archive, integrate-all-reporting-data, campaign-name`) et une valeur `LUNATIC_ONLY` qui n'existe plus dans l'enum. Le code actuel utilise exclusivement des arguments **nommés** `--clé=valeur`, et le mode "Lunatic uniquement" s'obtient via `--service=MAIN` (ou `GENESIS`) + `--with-ddi=false`, pas via une valeur de service dédiée. Le README est donc obsolète par rapport au code — à traiter dans la phase de comparaison code/doc.

Sur échec (statut non-2xx ou exception), le batch fait `System.exit(1)`, sauf si le profil Spring `test` est actif (l'exception est alors relancée, `KraftwerkBatch.java:294-301`).

En batch comme en API, le backend de stockage (disque local `FileSystemImpl` ou `MinioImpl`) est choisi selon `minioConfig.isEnable()`.

---

## 3. Packaging et exécution (Docker)

`Kraftwerk/Dockerfile` :
- Image de base : `gitlab-registry.insee.fr/kubernetes/images/run/java:25.0.3_9-jre-rootless` — JRE 25 interne INSEE, exécution non-root.
- `JAVA_TOOL_OPTIONS="-XX:+UseZGC -XX:MaxRAMPercentage=75"` : GC ZGC, heap plafonné à 75 % de la mémoire du conteneur.
- `EXPOSE 8080` ; le port effectif dépend en réalité de la configuration externe injectée (`spring.config.import`) — `application.properties` ne fixe pas `server.port` (seul `application-test.properties` le fait).
- `ENTRYPOINT ["java","-jar","/kraftwerk.jar"]` — un commentaire du Dockerfile confirme : sans paramètre = mode API REST, avec paramètres batch = mode batch. Cohérent avec la logique de `KraftwerkApi.main()`.

---

## 4. Sécurité

Authentification exclusivement **OIDC/JWT Bearer** (pas de Basic Auth, pas de session/login) :

- `fr.insee.kraftwerk.security.authentication=NONE` → `NoAuthConfiguration` : tout est permis, CSRF désactivé.
- `fr.insee.kraftwerk.security.authentication=OIDC` → `OIDCAuthConfiguration` :
  - Session stateless, `oauth2ResourceServer().jwt()`.
  - `/main/**` requiert le rôle `USER` ou `SCHEDULER` ; `/steps/**` et `/split/**` requièrent `ADMIN` ; le reste requiert simplement d'être authentifié (sauf chemins whitelistés).
  - Rôles extraits du JWT via un chemin de claim configurable (`oidcClaimRole`, ex. `realm_access.roles`) et mappés vers `ApplicationRole` (`ADMIN, USER, SCHEDULER`) par `RoleConfiguration`, à partir de trois listes de propriétés (`app.role.admin.claims`, etc.).
  - Hiérarchie de rôles : `ADMIN` inclut `USER` et `SCHEDULER`.
- Pas de configuration CORS trouvée dans `api/configuration` (aucune classe `WebMvcConfigurer`/CORS).
- Deux clients OIDC distincts existent : un côté ressource (vérification des JWT entrants) et un côté client (`OidcService`, client-credentials) pour authentifier les appels **sortants** vers Genesis (`GenesisAuthInterceptor` ajoute `Authorization: Bearer` et rejoue une fois en cas de 401).

---

## 5. Configuration

Aucun fichier YAML : uniquement des `.properties`.

- `kraftwerk-api/src/main/resources/application.properties` : configuration socle (voir table de propriétés ci-dessous), y compris un `spring.config.import` qui importe en plus un fichier `kraftwerk.properties` externe (classpath ou répertoire Tomcat webapps) — c'est le point d'injection de la configuration spécifique à l'environnement (Kubernetes/OPS).
- `application-test.properties` : ajoute les propriétés nécessaires aux tests (port, OIDC, Vault, rôles...).
- `kraftwerk_example.properties` : gabarit des valeurs attendues côté exploitation (toutes valorisées à `***`).
- Pas de `application-dev.properties`/`application-prod.properties`.

### Principales clés de configuration (relevées dans le code)

| Domaine | Clés |
|---|---|
| Général | `spring.application.name`, `fr.insee.kraftwerk.lang`, `fr.insee.kraftwerk.version` |
| Limites | `fr.insee.postcollecte.size-limit` (= `419430400`, soit exactement 400 Mio) |
| CSV | `fr.insee.postcollecte.csv.output.quote` |
| Stockage MinIO | `fr.insee.postcollecte.minio.{endpoint,access_key,secret_key,enable,bucket_name}` |
| Genesis | `fr.insee.postcollecte.genesis.api.url` |
| Sécurité | `fr.insee.kraftwerk.security.authentication`, `fr.insee.kraftwerk.security.token.oidc-claim-role`, `fr.insee.kraftwerk.security.token.oidc-claim-username`, `fr.insee.kraftwerk.security.whitelist-matchers`, `spring.security.oauth2.resourceserver.jwt.issuer-uri` |
| Vault/chiffrement | `fr.insee.kraftwerk.encryption.vault.{uri, app-role.role-id, app-role.secret-id}` |
| Rôles | `app.role.{admin,user,user-kraftwerk,user-platine,reader,collect-platform,scheduler}.claims` |
| DuckDB | `fr.insee.kraftwerk.duckdb.use-memory` |
| Monitoring | `management.health.probes.enabled`, `management.endpoint.health.show-details` |
| Swagger/OpenAPI | `springdoc.swagger-ui.path`, `springdoc.api-docs.resolve-schema-properties`, `springdoc.swagger-ui.tagsSorter`, `springdoc.swagger-ui.oauth2RedirectUrl`, `springdoc.swagger-ui.oauth.{scopes,client-id}` |

---

## 6. Intégrations externes

### Genesis (`api/client/GenesisClient.java`)
API REST amont fournissant les données d'enquête. Endpoints appelés côté Genesis :
`GET /health-check`, `GET /interrogations/by-questionnaire`, `GET /collection-instruments/{id}/interrogations/all`, `GET /collection-instruments/{id}/interrogations?since=...[&until=...]`, `GET /modes/by-campaign`, `GET /modes/by-questionnaire`, `POST /responses/{collectionInstrumentId}[?recordedBefore=...]`, `GET /questionnaires/by-campaign`, `GET /questionnaires/by-interrogation`, `GET/POST /questionnaire-metadata`, `PUT/GET /collection-instruments/{id}/extractions/json/last`.
Toutes les erreurs (hors health-check) sont enveloppées en `KraftwerkException(500, ...)`.

### Vault
Deux points d'intégration :
1. `VaultConfig` côté API : expose les propriétés (`uri`, `app-role.role-id/secret-id`) mais ne fait pas l'appel Vault lui-même.
2. `EncryptionUtilsImpl` (module `kraftwerk-encryption`) : appel réel via la librairie externe INSEE `lib_java_chiffrement_core`, chemin Vault `chiffrement/trust_key/aes_key`, mount `filiere_enquetes`. Ce composant est `@Primary` et actif sur tous les profils sauf `ci-public` — le build `ci-public` exclut le module `kraftwerk-encryption` et utilise un stub no-op (`EncryptionUtilsStub`).

### MinIO
Backend de stockage optionnel (`fr.insee.postcollecte.minio.enable`). Implémenté par `MinioImpl` (dans `kraftwerk-core/utils/files/`, pas dans le module API), qui respecte la même interface `FileUtilsInterface` que le stockage disque local `FileSystemImpl` — les deux sont utilisés de façon interchangeable dans tout le code métier.

### Trevas / VTL
`kraftwerk-core` embarque le moteur VTL Trevas (`fr.insee.trevas:vtl-engine`/`vtl-jackson`, version 2.4.0) comme `ScriptEngine` nommé `"vtl"`. Voir le détail des usages dans `02-fonctionnalites-principales.md`.

---

## 7. Monitoring / Observabilité

- `spring-boot-starter-actuator` est présent, avec `management.health.probes.enabled=true` et `management.endpoint.health.show-details=always`. Les endpoints actuator restent par ailleurs à leur configuration par défaut (une ligne commentée dans `application.properties` montre qu'ils ne sont pas restreints explicitement).
- Un endpoint applicatif dédié `GET health-check` (`HealthcheckService`) existe **en plus** d'Actuator : il vérifie la version de l'app, l'accessibilité de Genesis (`pingGenesis`) et l'existence du répertoire de stockage local. Voir les anomalies relevées dans `05-anomalies-code.md` (le résultat du test de fichier est calculé mais jamais utilisé, et un échec Genesis fait planter la requête au lieu de renvoyer un statut dégradé).
- `LogRequestFilter` journalise chaque requête/réponse (méthode, URI, paramètres, corps jusqu'à 100 Ko caché / 100 caractères journalisés), avec exclusion explicite de `/actuator/health` et masquage des corps Swagger.
- Pas de logs structurés JSON, pas de corrélation d'ID, pas de métriques Micrometer/Prometheus identifiées dans le code actuel (uniquement `spring-boot-starter-logging`/Logback classique).

---

## 8. Dépendances techniques clés (versions relevées dans les POM)

| Bibliothèque | Version | Usage |
|---|---|---|
| Spring Boot (parent) | 4.1.0 | framework |
| Java | 25 | langage |
| Cucumber | 7.34.4 | tests fonctionnels |
| Tomcat | 10.1.55 (override sécurité) | serveur embarqué |
| Trevas (`vtl-engine`/`vtl-jackson`) | 2.4.0 | moteur VTL |
| DuckDB JDBC | 1.5.4.0 | base SQL embarquée pour la génération des sorties |
| MinIO client | 8.6.0 | stockage objet |
| OkHttp | 4.12.0 (épinglé, la 5.x casse MinIO) | client HTTP |
| opencsv | 5.12.0 | lecture/écriture CSV |
| Saxon-HE | 13.0 | XSLT |
| commons-compress | 1.28.0 | archives |
| `fr.insee.bpm` | 1.2.2 | modèle de métadonnées DDI/Lunatic (externe) |
| `lib_java_chiffrement_core` | 1.5.6 | chiffrement + Vault (externe, module `kraftwerk-encryption` uniquement) |
| springdoc-openapi | 3.0.3 | documentation API Swagger/OpenAPI 3 |

Aucune dépendance Parquet explicite n'a été trouvée directement dans `kraftwerk-core/pom.xml` — l'écriture Parquet passe très probablement par DuckDB (`COPY ... TO ... (FORMAT PARQUET)`), à confirmer si besoin lors d'une inspection plus poussée de `outputs/parquet/`.

---

## 9. Historique récent pertinent (CHANGELOG.md)

- **4.1.0** (2026-06-18) : montée de version Spring Boot.
- **4.0.0** (2026-06-15) : passage à Trevas v2 — **breaking change** signalé pour d'anciens scripts VTL (impossibilité de surcharger des tables ou de garder/supprimer des identifiants comme avant).
- **3.16.3** (2026-05-27) : passage du mode batch en exécution synchrone pour éviter la fermeture prématurée du contexte Spring — cohérent avec le `app.run(args).close()` actuel.
- **3.13.0** (2026-03-20) : ajout des paramètres batch `BatchSize` et `Mode`.
- **3.12.0** (2026-03-05) : passage de Java 21 à 25.
- **3.4.1** : dépréciation des endpoints basés sur `campaignId` au profit de `questionnaireModelId` (cf. `@Deprecated(since="3.4.1")` sur `/main/genesis` et `/main/genesis/lunatic-only`).

---

## Points forts identifiés (observation factuelle, pas jugement de valeur)

1. Séparation modulaire claire core / API / chiffrement / tests.
2. Le chiffrement et le stockage objet sont conçus comme des implémentations interchangeables d'une interface (`EncryptionUtils`, `FileUtilsInterface`), avec fallback no-op pour les builds sans ces capacités (`ci-public`).
3. Tests fonctionnels Cucumber couvrant plusieurs familles de scénarios (voir document dédié à la comparaison code/tests/guide, à produire dans la phase suivante).
4. Documentation API via Swagger/OpenAPI, avec variante de sécurité (NONE/OIDC) branchée dynamiquement.

## Points de vigilance identifiés dans l'architecture (faits, détaillés dans 05-anomalies-code.md)

- Deux systèmes de suivi de jobs asynchrones coexistent, non interopérables (`InMemoryJobStore` et `InMemoryExportJobStore`), aucun des deux persistant (perte totale au redémarrage).
- Un mécanisme de statut de job (`/status/{jobId}`) semble ne jamais être exposé en HTTP (classe annotée `@Service` et non `@Controller`).
- Aucun `@ControllerAdvice` centralisé : chaque contrôleur convertit `KraftwerkException` en `ResponseEntity` à la main, avec des oublis (`throws` non catchés sur certains endpoints `/steps/**`, `/split/**`, `/jobs/{jobId}`).
- README et code batch divergent sur la forme des arguments et les valeurs de service acceptées.
