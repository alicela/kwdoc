# Kraftwerk cible — Spécification technique

**Date** : 2026-07-09  |  **Version** : 1.1 (mise à l'état de l'art)  |  **Statut** : cadrage cible

Ce document décrit **comment** est construite la cible : architecture, technologies, flux d'exécution, contrats internes. Le **quoi** fonctionnel est dans `03-specification-fonctionnelle.md`. Les garde-fous anti-dette sont dans `../1-existant/06-dette-technique-et-garde-fous.md`.

> ⚠️ **Écart assumé avec `../3-chemin-vers-la-cible/annexe-01-choix-architecturaux-initiaux.md`** (« Mistral Vibe », non vérifié) : il propose Spring WebFlux/Reactor, Spring Batch + PostgreSQL et 3 artefacts séparés. La cible retient d'autres choix (virtual threads + DuckDB, artefact unique, jobs derrière un port). Le présent document fait foi.

> 🎯 **Principe directeur « état de l'art »** : tirer parti de la plateforme (Java 25, Spring Framework 7 / Boot 4) plutôt que d'ajouter des frameworks. On privilégie **les capacités natives et standardisées** (virtual threads, scoped values, clients HTTP déclaratifs, `ProblemDetail` RFC 9457, Micrometer Observation/OpenTelemetry, SBOM CycloneDX) et on n'introduit une dépendance que lorsqu'elle apporte une valeur nette (Resilience4j, Flyway, DuckDB).

---

## 1. Décisions d'architecture

### 1.1 Socle — tranché ✅ (inchangé depuis les arbitrages du 2026-07-09)

| # | Décision | Justification |
|---|---|---|
| D-01 | **Java 25 (LTS) + Spring Boot 4.1 / Spring Framework 7 + Maven** | Décision client (`01-objectifs-refonte.md`) |
| D-02 | **Zéro VTL/Trevas** | Variables déjà calculées par Genesis (RG-ACQ-03) |
| D-03 | **Genesis unique source, par `partitionId`** | Décision client (RG-ACQ-01/90) |
| D-04 | **Concurrence : virtual threads + traitement ensembliste DuckDB** — pas de programmation réactive | DuckDB (colonnaire, spill-to-disk) porte le budget mémoire ; les virtual threads couvrent la concurrence I/O sans la complexité Reactor |
| D-05 | **Artefact unique** (`kraftwerk-core` + `kraftwerk-app`, API + runner batch dans un jar/image, mode au lancement) | Corrige la divergence de POM (doc 09) ; code métier partagé à 100 % |
| D-06 | **DuckDB au cœur du traitement** (split par niveau, réconciliation, écriture Parquet/CSV) | Opérations ensemblistes = clé du < 1 Go RAM |
| D-07 | **Ports & Adapters (hexagonal)** | Testabilité, découplage, remplaçabilité |
| D-08 | **Suivi de jobs derrière un port** : impl embarquée fichier par défaut, PostgreSQL activable | PostgreSQL requis dès le multi-instance |
| D-10 | **Liens 2à2 via BPM** (aucune logique Kraftwerk) | Décision client (RG-LIEN-90) |

### 1.2 État de l'art — tranché ✅ (renforcements 2026-07-09)

| # | Décision | Pourquoi c'est l'état de l'art |
|---|---|---|
| D-11 | **Modèle de domaine en `record` + `sealed interface` + pattern matching** (switch sur types scellés) | Immutabilité, exhaustivité vérifiée par le compilateur, moins de boilerplate (Java 21→25) |
| D-12 | **Client Genesis en interface HTTP déclarative** (`@HttpExchange` sur `RestClient` / `HttpServiceProxyFactory`, Spring 7) | Remplace le client écrit à la main ; contrat typé, testable, sans WebClient réactif |
| D-13 | **Résilience des appels sortants via Resilience4j** (timeout, retry avec backoff, circuit breaker, bulkhead) sur `GenesisPort` | Standard actuel de tolérance aux pannes ; métriques exposées à Micrometer |
| D-14 | **Erreurs HTTP normalisées `ProblemDetail` (RFC 9457)** via `@RestControllerAdvice` unique | Format d'erreur standardisé et interopérable ; remplace les corps d'erreur ad hoc (dette 08 §3) |
| D-15 | **Observabilité unifiée : Micrometer Observation API + Micrometer Tracing → OpenTelemetry (OTLP)** ; métriques + **traces distribuées** (traceId/spanId) | Traçabilité de bout en bout export→Genesis→stockage ; standard OTel vendor-neutral (remplace le simple correlationId) |
| D-16 | **Propagation de contexte par `ScopedValue`** (stable en Java 25), pas de `ThreadLocal`/MDC | `ThreadLocal` ne passe pas à l'échelle avec des millions de virtual threads ; `ScopedValue` est le mécanisme conçu pour cela |
| D-17 | **Job store en Spring Data JDBC + migrations Flyway** (PostgreSQL) ; schéma versionné | Persistance légère et explicite (pas d'ORM lourd) ; migrations reproductibles et auditées |
| D-18 | **Null-safety JSpecify** (annotations `@Nullable`/`@NonNull`) + vérification au build (NullAway / Error Prone) | Spring 7 adopte JSpecify ; détection des NPE à la compilation |
| D-19 | **Versionnement d'API natif Spring MVC 7** (négociation de version) | Fait évoluer l'API (p. ex. `partitionId`) sans casser les clients, sans bricolage d'URL |
| D-20 | **Démarrage/mémoire : CDS + cache AOT (Project Leyden)** activés par défaut | Démarrage plus rapide et RSS réduit — utile pour le mode batch (cold start) et le budget mémoire, sans les contraintes du natif |
| D-21 | **Chaîne logicielle : SBOM CycloneDX généré au build + scan dépendances/image (Trivy/Grype/OWASP) + builds reproductibles** | Exigence de sécurité supply-chain moderne (SLSA-friendly) |
| D-22 | **Config typée et validée** (`@ConfigurationProperties` + `jakarta.validation`), secrets via Vault, `spring.config.import` config-tree pour secrets montés (K8s) | Config sûre, testable, sans valeur en dur (dette D9) |
| D-23 | **Runtime cloud-native** : Actuator **health groups** liveness/readiness, **graceful shutdown**, JVM `MaxRAMPercentage`, export OTLP | Alignement Kubernetes standard |

### 1.3 À évaluer (bénéfice réel à confirmer)

| # | Piste | Réserve |
|---|---|---|
| E-01 | **Image native GraalVM** pour le profil batch (démarrage ~ms, RSS bas) | DuckDB (JDBC **JNI**) + réflexion → configuration native non triviale ; **CDS (D-20) donne déjà l'essentiel du gain** sans ce risque. À benchmarker avant adoption |
| E-02 | **Construction d'image via Cloud Native Buildpacks** (`spring-boot:build-image`) au lieu du Dockerfile | À arbitrer avec la contrainte d'image de base interne INSEE (registry rootless) |
| E-03 | **Structured Concurrency** (`StructuredTaskScope`) pour le fan-out d'acquisition | *Preview* en Java 25 → baseline = exécuteur virtual-thread (stable) ; bascule quand l'API est finalisée |
| E-04 | **Signature d'artefacts / provenance** (cosign, attestations SLSA) | Selon la politique de la chaîne CI/CD INSEE |
| E-05 | **Contract testing** du client Genesis (Spring Cloud Contract / Pact) | Utile si Genesis évolue souvent ; coût d'outillage à peser |

### 1.4 Points fonctionnels ouverts (conception détaillée)

| # | Point | Proposition par défaut |
|---|---|---|
| O-01 | Format/stockage de la config d'anonymisation par enquête | Config par enquête (YAML/JSON) chargée via le port de stockage |
| O-02 | Lever/maintenir « pas de boucles imbriquées » | Maintenir au MVP, détecter+signaler le cas non géré |
| O-03 | Règle par défaut sur doublon inter-mode | Conserver 2 lignes distinguées par `MODE_KRAFTWERK` |
| O-04 | Préfixe de nommage des fichiers | `<partitionId>_<TABLE>.<ext>` |

---

## 2. Vue d'ensemble

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        kraftwerk-app  (1 jar / 1 image)                        │
│                                                                              │
│  Mode API (défaut)                              Mode batch (--batch …)        │
│  ┌───────────────────────────┐                 ┌────────────────────┐        │
│  │ REST Controllers (MVC 7,   │                 │ CommandLineRunner   │        │
│  │  versionnés) + Security OIDC│                 │ (args → même        │        │
│  │ @RestControllerAdvice →    │                 │  use case)          │        │
│  │ ProblemDetail (RFC 9457)   │                 └─────────┬──────────┘        │
│  └─────────────┬─────────────┘                           │                   │
│                └──────────────────┬────────────────────────┘                 │
│                                   ▼                                           │
│                    ┌──────────────────────────────┐                          │
│                    │  Use cases (kraftwerk-core)    │  domaine : records +     │
│                    │  ExportTabular / ExportJson /  │  sealed + pattern match  │
│                    │  ExportReporting …             │  (aucune dépendance web) │
│                    └───────────────┬───────────────┘                          │
│                                    │ ports                                     │
│   ┌───────────┬──────────┬─────────┼──────────┬───────────┬──────────────┐   │
│   ▼           ▼          ▼         ▼          ▼           ▼              ▼   │
│ GenesisPort MetadataPort TabularEngine StoragePort EncryptionPort JobStorePort │
│  │@HttpExchange BPM        DuckDB       FS/MinIO    Vault (opt.)  SpringDataJDBC│
│  │+Resilience4j (DDI/Lun.) (in-proc)                              embarqué|PG   │
│  ▼                                                                             │
│ Genesis REST                                                                   │
│                                                                              │
│  Transverse : Micrometer Observation + Tracing → OTLP ; logs JSON ;           │
│               contexte via ScopedValue ; Actuator (health groups, métriques)  │
└────────────────────────────────────────────────────────────────────────────┘
        ▲
   OIDC (JWT)
```

- **`kraftwerk-core`** : domaine (records/sealed) + use cases + ports. Aucune dépendance Spring Web ni techno externe concrète. Annoté JSpecify.
- **`kraftwerk-app`** : adapters (controllers, runner batch, client Genesis déclaratif, moteur DuckDB, écrivains), configuration Spring, sécurité, observabilité.
- **`kraftwerk-encryption`** : adapter chiffrement (Vault + lib INSEE), profil activable, **buildé et testé en CI**.

---

## 3. Modules Maven, build et chaîne logicielle

```
kraftwerk-parent (pom.xml unique)
├── kraftwerk-core          # domaine, use cases, ports ; DuckDB SQL ; BPM
├── kraftwerk-app           # API + runner batch + adapters + config + observabilité
└── kraftwerk-encryption    # adapter chiffrement (Vault), profil activable
```

- **Un seul chemin de build** : artefact testé en CI = artefact livré (fin de la divergence `pom.xml`/`pom-public.xml`). Le profil « sans chiffrement » substitue un no-op mais **le module reste compilé/testé**.
- **Jackson 3** (`tools.jackson`, standard Boot 4) unifié — plus de mélange de versions.
- **Null-safety** : JSpecify + NullAway (Error Prone) en échec de build sur violation (D-18).
- **SBOM CycloneDX** généré au build (`cyclonedx-maven-plugin` / support Boot), publié comme artefact ; **scan** dépendances + image (Trivy/Grype/OWASP) bloquant sur sévérité haute (D-21).
- **BPM** épinglé à une **version publiée** (fin du build depuis `@master`).
- **Renovate** enrichi (groupes, `vulnerabilityAlerts`, dashboard).
- **Image** : multi-stage rootless (base INSEE), digest-pinné, `HEALTHCHECK`, **CDS/AOT cache** intégré au layer (D-20) ; Buildpacks à évaluer (E-02).

---

## 4. Pipeline de traitement (cœur performance)

Principe : **Java orchestre l'I/O, DuckDB fait le calcul ensembliste.** Objectif < 1 Go RAM (NF-001).

```
1. ACQUISITION (I/O concurrent, virtual threads)
   - GenesisPort.streamByPartition(partitionId, mode?, batchSize)  [client @HttpExchange + Resilience4j]
   - pagination : une page en mémoire ; appender DuckDB en table de staging
   - fan-out borné (sémaphore) ; StructuredTaskScope quand finalisé (E-03)
2. STRUCTURE (SQL ensembliste)
   - métadonnées BPM → schéma des niveaux (RACINE + boucles)
   - split par niveau = requêtes SQL (pas de reconstruction par parsing de noms — CL-STR-03)
3. RÉCONCILIATION MULTIMODE (SQL ensembliste) — si ≥ 2 modes
   - UNION des modes + colonne MODE_KRAFTWERK + harmonisation de types
4. ANONYMISATION (SQL) — option : SELECT * EXCLUDE (...) AVANT écriture
5. ÉCRITURE (streaming DuckDB) : COPY … TO … (FORMAT PARQUET) / (FORMAT CSV, …) ; spill géré par DuckDB
6. POST (option) : script R, archivage ZIP, chiffrement, upload MinIO
```

**Modèle DuckDB** (schéma par niveau, aligné RG-STR) :
```sql
CREATE TABLE staging_raw(interrogation_id, usual_su_id, mode, scope, iteration, var_id, value, state, questionnaire_state, validation_date, …);
CREATE TABLE racine(interrogation_id PRIMARY KEY, usual_su_id, questionnaire_state, validation_date, <vars scope=RACINE> …);
CREATE TABLE loop_<boucle>(interrogation_id, occurrence_id, <vars boucle> …, PRIMARY KEY(interrogation_id, occurrence_id));
```
Base de travail DuckDB éphémère par job (ingestion via l'**appender** natif pour la performance). Parquet produit nativement (pas de dépendance Parquet dédiée — anomalie 08 §13).

---

## 5. Contrats de ports (esquisse)

```java
// Acquisition — client HTTP déclaratif (D-12) + résilience (D-13)
@HttpExchange("/api")
interface GenesisPort {
    @GetExchange("/partitions/{partitionId}/interrogations")
    Stream<Interrogation> streamByPartition(@PathVariable String partitionId,
                                            @RequestParam Optional<Mode> mode,
                                            @RequestParam int batchSize);
    Instant getLastExtraction(String collectionInstrumentId);   // JSON incrémental
    void   updateLastExtraction(String collectionInstrumentId, Instant date);
    HealthStatus ping();
}
// Domaine immuable (D-11)
record Interrogation(String interrogationId, String usualSurveyUnitId, Mode mode,
                     QuestionnaireState state, Instant validationDate,
                     List<Variable> collected, List<Variable> external) {}
sealed interface OutputTarget permits DiskTarget, MinioTarget {}

interface MetadataPort   { MetadataModel forMode(String partitionId, Mode mode); }
interface TabularEngine  { void ingest(Stream<Interrogation> d); void splitIntoLevels(MetadataModel md);
                           void reconcileModes(); void anonymize(List<String> drop);
                           List<OutputFile> writeParquet(Path dir); List<OutputFile> writeCsv(Path dir); }
interface StoragePort    { InputStream read(String p); void write(Path local, String target); boolean isAvailable(); }
interface EncryptionPort { Path encryptArchive(Path zip); }                 // no-op si profil off
interface JobStorePort   { JobId create(JobRequest r); void update(JobId id, JobStatus s, JobResult partial);
                           Optional<JobExecution> find(JobId id); }         // impl Spring Data JDBC (D-17)
```
Le cœur ne connaît que ces interfaces ; les adapters vivent dans `kraftwerk-app`/`kraftwerk-encryption`.

---

## 6. Exécution asynchrone et suivi de jobs

- **Déclenchement** : endpoint → `202 Accepted` + `jobId` ; traitement soumis à un exécuteur **virtual-thread-per-task** borné par sémaphore (RG-EXE-01).
- **Store unifié persistant** derrière `JobStorePort`, en **Spring Data JDBC** avec **schéma versionné Flyway** (D-17) :
  - **défaut embarqué** : fichier local DuckDB/SQLite (mono-instance) ;
  - **option PostgreSQL** : dès plusieurs répliques (statut cohérent entre pods).
- **Statuts** `RUNNING/DONE/PARTIAL/FAILED` (RG-EXE-03) ; résultat = statut + horodatages + résumé + erreurs (RG-EXE-04).
- **Au redémarrage** : jobs `RUNNING` orphelins marqués `FAILED`/interrompus (CL-EXE-02).
- Schéma : `job_execution(id, type, partition_id, status, started_at, ended_at, summary_json, errors_json)` (migration Flyway `V1__job_execution.sql`).

> Le MVP démarre sur l'impl embarquée (persistante) ; PostgreSQL se branche au passage multi-instance sans changer le use case (même port).

---

## 7. Sécurité (RG-EXE-20..22)

- **Resource server OIDC/JWT** (Spring Security), mode `NONE` pour environnements ouverts.
- **Method security** (`@PreAuthorize`) + règles par endpoint ; rôles `ADMIN` ⊃ `USER`/`SCHEDULER`. Export → `USER`/`SCHEDULER`, admin → `ADMIN`. Surface : `/exports/**`, `/jobs/**`, `/reporting/**`, `/json/**`, `/actuator/**`, `/health`.
- **Auth service-à-service vers Genesis** : client-credentials, jeton rafraîchi, rejeu une fois sur 401 (via l'adapter du client déclaratif).
- **Versionnement d'API** natif (D-19) pour faire évoluer les contrats sans rupture.
- **Tests de sécurité** (`spring-security-test`) : anonyme→401, rôle insuffisant→403, rôle requis→200.

---

## 8. Configuration (fin de la config en dur — dette D9)

- **Externalisation totale** : aucune valeur d'environnement dans `src/main/resources` livrés ; `application.yml` = défauts non sensibles.
- **Config typée + validée** (`@ConfigurationProperties` + `jakarta.validation`) (D-22).
- **Secrets via Vault** ; secrets montés en fichiers lus via `spring.config.import=configtree:` (K8s).
- Paramètres clés : `batchSize`, backend stockage (`filesystem|minio`), backend job-store (`embedded|postgres`), profil chiffrement, endpoint OTLP, seuils.
- Lint CI : absence de chemin absolu / URL de prod dans les resources.

---

## 9. Observabilité et erreurs (état de l'art)

- **Micrometer Observation API** : une observation par export (et par appel Genesis), donnant **à la fois métriques et traces** sans double instrumentation.
- **Traces distribuées** via **Micrometer Tracing → OpenTelemetry (OTLP)** (D-15) : `traceId`/`spanId` propagés jusqu'à Genesis ; export vers le collector OTel.
- **Contexte porté par `ScopedValue`** (D-16), compatible virtual threads (fin du `ThreadLocal`/MDC) ; le `traceId` enrichit automatiquement les logs.
- **Logs structurés JSON** (Actuator/Logback ou `structured-logging` de Boot), corrélés au `traceId`.
- **Métriques** exposées via Actuator/Prometheus (durée, nb interrogations, nb erreurs, mémoire) + métriques Resilience4j.
- **Health-check** : Actuator **health groups** liveness/readiness ; indicateurs custom Genesis/stockage corrigés (résultat exploité, `ping` en try/catch → statut dégradé, pas de plantage — corrige 08 §4 ; RG-EXE-13).
- **Erreurs** : `@RestControllerAdvice` unique produisant des **`ProblemDetail` (RFC 9457)** (D-14) ; table exception→statut centralisée (400/404/409/500…). Distinction **récupérables** (collectées, `errors.txt` + `PARTIAL`) vs **fatales** (`FAILED`, arrêt) — plus de capture des `Error` Java (corrige 08 §8).

---

## 10. Tests (pyramide + portes qualité)

- **Unitaires** (JUnit 5, Mockito, AssertJ) sur use cases et logique de reporting (`OUTCOME_SPOTTING`).
- **Intégration** : `@SpringBootTest` + **Testcontainers avec `@ServiceConnection`** (PostgreSQL quand activé ; DuckDB in-proc sinon).
- **Fonctionnels Cucumber** rebâtis sur **stubs Genesis** paramétrés par `partitionId` (cf. `05-couverture-tests-cucumber.md`).
- **Portes CI bloquantes** (doc 09 §6.2, enrichies) : couverture ≥ seuil ; **ArchUnit** (interdit `fr.insee.vtl.*`/`trevas`, `@*Mapping` hors contrôleur, dépendances de couche) ; **NullAway** (JSpecify) ; **scan vulnérabilités + SBOM** ; **PIT** (mutation) ; build unique testant le chiffrement.

---

## 11. Correspondance dette → traitement technique (doc 09)

| Dette existante | Traitement cible |
|---|---|
| Couplage VTL (99 réf./31 fichiers) | Supprimé ; SQL DuckDB ; garde-fou ArchUnit |
| `BatchExportService` clone de `MainService` | Point d'entrée unique ; batch = runner fin sur le même use case |
| Pas de `@ControllerAdvice` | `@RestControllerAdvice` unique → `ProblemDetail` (RFC 9457) |
| Job store non exposé / non persistant | `JobStorePort` (Spring Data JDBC + Flyway), unifié + persistant |
| Divergence `pom.xml`/`pom-public.xml` | POM unique, build = livraison |
| Config en dur (chemins Windows, URL prod) | Config typée externalisée + Vault + lint CI |
| Chiffrement non testé en CI | Module compilé et testé en CI |
| Pas de scan / supply-chain | SBOM CycloneDX + scan dépendances/image bloquant |

---

## 12. Trajectoire technique

Les incréments suivent le backlog (`../3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md`). Ordre logique : **fondations** (artefact unique, ports/adapters, client Genesis déclaratif + Resilience4j, CI + portes qualité + SBOM, CDS) → **MVP** (acquisition → structure → Parquet) → **CSV/R, multimode SQL, JSON, reporting, anonymisation, chiffrement/MinIO, job store persistant (Flyway), sécurité/rôles, observabilité OTel** → **évolutions** (lot, multimode sans réconciliation, codification, date T). L'image native (E-01) et le contract testing (E-05) sont évalués en cours de route, non requis pour la cible.
