# Kraftwerk cible — Spécification technique

**Date** : 2026-07-09  |  **Version** : 1.1 (mise à l'état de l'art)  |  **Statut** : cadrage cible

Ce document décrit **comment** est construite la cible : architecture, technologies, flux d'exécution, contrats internes. Le **quoi** fonctionnel est dans `03-specification-fonctionnelle.md`. Les garde-fous anti-dette sont dans `../1-existant/06-dette-technique-et-garde-fous.md`.

> ⚠️ **Écart assumé avec `../3-chemin-vers-la-cible/annexe-01-choix-architecturaux-initiaux.md`** (« Mistral Vibe », non vérifié) : il propose Spring WebFlux/Reactor et Spring Batch + PostgreSQL persistant. La cible retient virtual threads + DuckDB (pas de réactif) et un **suivi de jobs en mémoire, non persistant**. En revanche, la **séparation API / Batch** est bien retenue (décision client 2026-07-09). Le présent document fait foi.

> 🔄 **Révisions client 2026-07-09** intégrées : suivi de jobs **non persistant** (mémoire, rejouable) ; **séparation API / Batch** (utilité de l'API à confirmer) ; **stockage MinIO** (Kube, plus de filesystem a priori) ; **chiffrement tout au long du traitement** et **interchangeable** ; DuckDB retenu **essentiellement pour ses capacités d'export** Parquet/JSON/CSV.

> 🎯 **Principe directeur « état de l'art »** : tirer parti de la plateforme (Java 25, Spring Framework 7 / Boot 4) plutôt que d'ajouter des frameworks. On privilégie **les capacités natives et standardisées** (virtual threads, scoped values, clients HTTP déclaratifs, `ProblemDetail` RFC 9457, Micrometer Observation/OpenTelemetry, SBOM CycloneDX) et on n'introduit une dépendance que lorsqu'elle apporte une valeur nette (Resilience4j, DuckDB).

---

## 1. Décisions d'architecture

### 1.1 Socle — tranché ✅ (inchangé depuis les arbitrages du 2026-07-09)

| # | Décision | Justification |
|---|---|---|
| D-01 | **Java 25 (LTS) + Spring Boot 4.1 / Spring Framework 7 + Maven** | Décision client (`01-objectifs-refonte.md`) |
| D-02 | **Zéro VTL/Trevas** | Variables déjà calculées par Genesis (RG-ACQ-03) |
| D-03 | **Genesis unique source, par `partitionId`** | Décision client (RG-ACQ-01/90) |
| D-04 | **Concurrence : virtual threads + traitement ensembliste DuckDB** — pas de programmation réactive | DuckDB (colonnaire, spill-to-disk) porte le budget mémoire ; les virtual threads couvrent la concurrence I/O sans la complexité Reactor |
| D-05 | **Séparation API / Batch** : `kraftwerk-core` (métier partagé) + `kraftwerk-batch` (mode principal, Job Kube) + `kraftwerk-api` (**utilité à confirmer**). POM parent unique, un seul chemin de build par artefact | Décision client 2026-07-09 ; batch = exécution principale sur Kubernetes |
| D-06 | **DuckDB au cœur du traitement** (split par niveau, réconciliation, écriture) | **Retenu essentiellement pour ses capacités d'export Parquet/JSON/CSV** ; les opérations ensemblistes servent aussi le budget < 1 Go RAM |
| D-07 | **Ports & Adapters (hexagonal)** | Testabilité, découplage, remplaçabilité |
| D-08 | **Suivi de jobs unifié EN MÉMOIRE, non persistant** (derrière un port) | Décision client : traitements temporaires et **rejouables** ; un plantage perd les jobs en cours, ce n'est pas grave — **pas de persistance** |
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
| D-17 | **Suivi de jobs en mémoire** (structure concurrente unifiée), pas de base ni de migration | Aligné sur la décision « non persistant » (D-08) — inutile d'introduire PostgreSQL/Flyway pour du transitoire rejouable |
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

> 📐 Schémas **C4** dans `../schemas/` : [contexte](../schemas/cible-c4-contexte.png) (niveau 1), [conteneurs](../schemas/cible-c4-conteneurs.png) (niveau 2, `.drawio` éditable), [composants hexagonaux](../schemas/cible-c4-composants.png) (niveau 3). Le diagramme ASCII ci-dessous en est un résumé.

```
┌────────────────────────────────────────────────────────────────────────────┐
│        kraftwerk-core partagé — artefacts : kraftwerk-batch (Kube) + kraftwerk-api (?)        │
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
│  │@HttpExchange BPM        DuckDB       MinIO       Vault (opt.)  en mémoire    │
│  │+Resilience4j (DDI/Lun.) (in-proc)    (Kube)                    (non persist.)│
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
- **`kraftwerk-batch`** : **mode d'exécution principal** (Job Kubernetes) — point d'entrée CLI, adapters (client Genesis déclaratif, moteur DuckDB, écrivains, MinIO), config, observabilité.
- **`kraftwerk-api`** : exposition REST (controllers, sécurité OIDC) — **utilité à confirmer** (le batch peut suffire). Réutilise les mêmes use cases et adapters que le batch.
- **`kraftwerk-encryption`** : adapter chiffrement (Vault + lib INSEE **interne, interchangeable**), profil activable, **buildé et testé en CI**.

---

## 3. Modules Maven, build et chaîne logicielle

```
kraftwerk-parent (pom.xml unique)
├── kraftwerk-core          # domaine, use cases, ports ; DuckDB SQL ; BPM
├── kraftwerk-batch         # mode principal (Job Kube) : CLI + adapters + config + observabilité
├── kraftwerk-api           # exposition REST (utilité à confirmer) ; réutilise core + adapters
└── kraftwerk-encryption    # adapter chiffrement (Vault, interchangeable), profil activable
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
   - ANONYMISATION (option) = schéma effectif = colonnes des métadonnées MOINS les variables anonymisées
     → colonnes jamais matérialisées (exclusion EN AMONT, pas de drop en fin ; RG-ANO-03)
   - split par niveau = requêtes SQL (pas de reconstruction par parsing de noms — CL-STR-03)
3. RÉCONCILIATION MULTIMODE (SQL ensembliste) — si ≥ 2 modes
   - UNION des modes + colonne MODE_KRAFTWERK (types censés concordants — sinon erreur, RG-MULTI-04)
4. POST-SCRIPT (option) : script SQL/R sur les tables finales (RG-CSV-13)
5. ÉCRITURE (streaming DuckDB) : COPY … TO … (FORMAT PARQUET) / (FORMAT CSV, …) ; spill géré par DuckDB
   - filet : SELECT * EXCLUDE(<anonymisées>) à l'export (défense en profondeur, coût nul)
6. POST (option) : archivage ZIP, chiffrement, upload MinIO
```

**Modèle DuckDB** (schéma par niveau, aligné RG-STR) :
```sql
CREATE TABLE staging_raw(interrogation_id, usual_su_id, mode, scope, iteration, var_id, value, state, questionnaire_state, validation_date, …);
CREATE TABLE racine(interrogation_id PRIMARY KEY, usual_su_id, questionnaire_state, validation_date, <vars scope=RACINE> …);
CREATE TABLE loop_<boucle>(interrogation_id, occurrence_id, <vars boucle> …, PRIMARY KEY(interrogation_id, occurrence_id));
```
Base de travail DuckDB éphémère par job (ingestion via l'**appender** natif pour la performance). Parquet produit nativement (pas de dépendance Parquet dédiée — anomalie 08 §13).

### 4.1 Pivot « lignes de variables → tables larges par niveau » (design du cœur MVP)

> 📐 Les étapes et leurs points de parallélisation sont schématisés dans [`../schemas/cible-pipeline-parallelisme.png`](../schemas/cible-pipeline-parallelisme.png).

**Problème.** Genesis renvoie des **documents** ; chaque document contient une **liste d'interrogations×mode**, et **toutes les données d'une interrogation×mode y sont regroupées** (`varId, value, scope, iteration` + champs fixes) — une interro×mode n'est **jamais scindée** entre deux documents. La sortie attendue est, **par niveau d'information**, une **table large** : une colonne par variable, une ligne par interrogation (RACINE) ou par occurrence (boucle). Les colonnes sont **dynamiques** (elles dépendent de l'enquête).

**Principe retenu : pivot PILOTÉ PAR LES MÉTADONNÉES (BPM), pas par les données.** Le schéma de chaque niveau (liste des variables + types) vient du **modèle de métadonnées BPM**, pas des valeurs observées. Avantages : (a) **schéma stable** d'une partition à l'autre (une variable sans valeur donne une colonne `NULL`, elle ne disparaît pas) ; (b) **typage explicite** ; (c) on **ne reconstruit jamais la hiérarchie en parsant les noms à points** (résout la fragilité CL-STR-03) : le rattachement à un niveau se fait par `scope`, fourni par la donnée et validé par les métadonnées.

**Deux implémentations possibles** — le fait que **chaque interro×mode soit complète dans un document** (jamais scindée) débloque la seconde :
- **(A) Staging + pivot ensembliste** : empiler les lignes de tous les documents dans une table `staging`, puis un `GROUP BY` par niveau (SQL ci-dessous). Simple ; pivot après ingestion, parallélisé en interne par DuckDB.
- **(B) Pivot PAR INTERRO×MODE, pipeliné** *(préférée)* : chaque interro×mode étant **complète dans un document**, on la met en forme **dès réception du document** (assemblage RACINE + occurrences) puis on **append** le résultat → fetch et mise en forme **se chevauchent**, hautement parallèle, sans attendre toute la partition.

Les deux produisent le **même schéma** métadonnées-piloté et partagent le SQL-builder et le typage ci-dessous.

**Étapes (variante A).**
1. **Staging** : insertion des lignes de variables dans une table DuckDB de travail via l'**Appender** (rapide, hors heap Java).
2. **Génération du SQL par niveau** : pour chaque groupe du modèle BPM (RACINE + une boucle par groupe), un petit **SQL-builder Java** émet un `CREATE TABLE … AS SELECT` par **agrégation conditionnelle** (`… FILTER (WHERE var_id = …)`), une colonne par variable du niveau, castée au type cible.
3. **Identifiant d'occurrence** pour les boucles : calculé par fenêtre (`dense_rank`) sur `iteration`.
4. Écriture Parquet/CSV par `COPY`.

**Exemple SQL (illustratif).**
```sql
-- RACINE : GROUP BY interrogation ; colonnes = variables scope=RACINE (métadonnées)
CREATE TABLE racine AS
SELECT interrogation_id,
       any_value(usual_su_id)         AS "usualSurveyUnitId",
       any_value(questionnaire_state) AS "questionnaireState",
       any_value(validation_date)     AS "validationDate",
       CAST(max(value) FILTER (WHERE var_id='AGE')  AS BIGINT)  AS "AGE",
       CAST(max(value) FILTER (WHERE var_id='SEXE') AS VARCHAR) AS "SEXE"
       -- … une ligne générée par variable du niveau RACINE
FROM staging WHERE scope='RACINE'
GROUP BY interrogation_id;

-- BOUCLE : GROUP BY (interrogation, iteration) ; 1 ligne par occurrence
CREATE TABLE loop_BOUCLE_PRENOMS AS
SELECT interrogation_id,
       'BOUCLE_PRENOMS-' || lpad(CAST(dense_rank() OVER w AS VARCHAR), 2, '0') AS "occurrence_id",
       CAST(max(value) FILTER (WHERE var_id='PRENOM') AS VARCHAR) AS "PRENOM",
       CAST(max(value) FILTER (WHERE var_id='LIEN_1') AS VARCHAR) AS "LIEN_1"
FROM staging WHERE scope='BOUCLE_PRENOMS'
GROUP BY interrogation_id, iteration
WINDOW w AS (PARTITION BY interrogation_id ORDER BY iteration);
```

**Génération dynamique (côté Java).** À partir du `MetadataModel` BPM :
```java
String colExpr(Variable v) {                       // une colonne = une variable
  return "CAST(max(value) FILTER (WHERE var_id=" + sqlLit(v.id()) + ") AS "
       + duckType(v.type()) + ") AS " + quoteIdent(v.name());
}
// duckType : STRING→VARCHAR, INTEGER→BIGINT, NUMBER→DOUBLE, BOOLEAN→BOOLEAN, DATE→DATE
```
- **Sécurité** : les `var_id`/noms proviennent du modèle de métadonnées (allowlist) ; `quoteIdent`/`sqlLit` protègent l'injection et les caractères spéciaux (dont le `.`). Pas de SQL utilisateur ici (le script de post-traitement RG-CSV-13 est un sujet distinct).
- **États (`addStates`)** : mêmes colonnes suffixées `_STATE` par agrégation sur la colonne `state`, en excluant `FILTER_RESULT_*`/`*_MISSING` (RG-ACQ-11).
- **Anonymisation = en amont, dans ce même SQL-builder** (RG-ANO-03) : on **retire les variables anonymisées de la liste de colonnes** avant de générer les `colExpr` → les colonnes ne sont **jamais matérialisées** (ni ingérées/pivotées/réconciliées/exportées). Pas de `DROP COLUMN` en fin. Config validée en amont : **interdit d'exclure une variable identifiante/structurante** (CL-ANO-02). Filet à l'export : `COPY (SELECT * EXCLUDE(<anonymisées>))` (défense en profondeur, coût nul).
- **Erreurs de cast** (valeur non conforme au type déclaré) : collectées comme erreurs **récupérables** → statut `PARTIAL`, colonne à `NULL` (RG-EXE-31), plutôt que d'échouer tout le job.
- **Multimode** : le pivot est unimodal (staging filtré/portant `mode`) ; la réconciliation (UNION + `MODE_KRAFTWERK`) est l'étape SQL suivante (EPIC-3).

**Ingestion par lots, mémoire et parallélisme** (le `CREATE TABLE … AS SELECT` ne s'y oppose pas) :
- **Ingestion et pivot sont deux phases découplées.** Les **appels paginés Genesis alimentent `staging` au fil de l'eau** (Appender) ; le pivot lit `staging`, jamais Genesis directement. Les lots Genesis sont donc préservés.
- **`staging` vit dans DuckDB (colonnaire, sur disque, spill géré), pas dans le tas Java** → « toute la partition en staging » n'implique pas une forte RAM ; l'objectif < 1 Go (heap) tient.
- **Le pivot est parallélisé par DuckDB** (exécution morsel-driven multi-cœurs du `GROUP BY`). De plus, **les niveaux (RACINE + chaque boucle) sont indépendants** → pivotables en parallèle (lectures concurrentes). Parallélisme gros grain : **plusieurs partitions/jobs concurrents** (virtual threads), chacun sa base DuckDB de travail.
- **Écriture DuckDB = un seul writer** : on parallélise le **fetch Genesis** (I/O) et on draine les pages vers un writer unique via une file bornée (append quasi gratuit).
- **Pipelining natif (variante B)** : chaque interro×mode est **complète dans un document** → dès qu'un document est reçu, ses interro×mode sont mises en forme, **fetch et mise en forme se chevauchent** (pas de barrière « après ingestion »). Seul point de sérialisation : l'**append** dans DuckDB (writer unique), rapide.
- **Barrières réelles** : la **réconciliation multimode** (UNION des modes) et l'**export** interviennent une fois les tables par mode accumulées — étapes ensemblistes, parallélisées par DuckDB et par niveau/table.

**Alternative écartée** : le `PIVOT … ON var_id` natif de DuckDB **infère les colonnes des données** (schéma instable entre partitions, tout en `VARCHAR`) → non retenu pour la cible ; l'agrégation conditionnelle pilotée par métadonnées est préférée.

**Points à trancher** : numérotation d'occurrence — `dense_rank` (occurrences contiguës 1..N) **vs** `iteration` brut de Genesis (CL-STR-04) ; représentation des grands entiers/décimaux à l'écriture (RG-CSV-06 : `BIGINT` évite la notation scientifique).

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
interface StoragePort    { InputStream read(String p); void write(Path local, String target); boolean isAvailable(); } // MinIO en prod
interface EncryptionPort { Path encrypt(Path data); }                       // interchangeable ; no-op si profil off (D-12/RG-ANO-12)
interface JobStorePort   { JobId create(JobRequest r); void update(JobId id, JobStatus s, JobResult partial);
                           Optional<JobExecution> find(JobId id); }         // impl EN MÉMOIRE, non persistante (D-08/D-17)
```
Le cœur ne connaît que ces interfaces ; les adapters vivent dans `kraftwerk-batch`/`kraftwerk-api`/`kraftwerk-encryption`.

---

## 6. Exécution asynchrone et suivi de jobs

- **Déclenchement** : endpoint (mode API) ou lancement du Job (mode batch) → `202 Accepted` + `jobId` en API ; traitement soumis à un exécuteur **virtual-thread-per-task** borné par sémaphore (RG-EXE-01).
- **Store de jobs unifié EN MÉMOIRE** derrière `JobStorePort` (D-08/D-17) : une seule structure concurrente (`ConcurrentHashMap`) remplaçant les deux stores mémoire de l'existant. **Aucune persistance, aucune base.**
- **Suivi au fil de l'eau** : le statut et la progression sont observables pendant le traitement (RG-EXE-02) — statuts `RUNNING/DONE/PARTIAL/FAILED` (RG-EXE-03), résultat = statut + horodatages + résumé + erreurs (RG-EXE-04).
- **Au redémarrage** : les jobs en cours sont **perdus** — comportement **accepté** (traitements temporaires et **rejouables**, CL-EXE-02) : il suffit de relancer l'export.

> Conséquence Kube : le mode batch s'exécute typiquement comme un **Job** (une exécution = un pod) ; le suivi mémoire vit le temps du traitement. Si un besoin de suivi trans-exécutions apparaissait, il serait ajouté derrière le même port — mais ce n'est **pas** demandé.

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
- Paramètres clés : `batchSize`, connexion **MinIO** (backend de stockage en prod ; `filesystem` réservé au dev/test), profil chiffrement, endpoint OTLP, seuils. *(Pas de backend de persistance de jobs : suivi en mémoire.)*
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
- **Intégration** : `@SpringBootTest` ; DuckDB in-proc ; **Testcontainers (avec `@ServiceConnection`)** pour MinIO/Genesis stubbé. *(Pas de base à conteneuriser pour les jobs — suivi en mémoire.)*
- **Fonctionnels Cucumber** rebâtis sur **stubs Genesis** paramétrés par `partitionId` (cf. `05-couverture-tests-cucumber.md`).
- **Portes CI bloquantes** (doc 09 §6.2, enrichies) : couverture ≥ seuil ; **ArchUnit** (interdit `fr.insee.vtl.*`/`trevas`, `@*Mapping` hors contrôleur, dépendances de couche) ; **NullAway** (JSpecify) ; **scan vulnérabilités + SBOM** ; **PIT** (mutation) ; build unique testant le chiffrement.

---

## 11. Correspondance dette → traitement technique (doc 09)

| Dette existante | Traitement cible |
|---|---|
| Couplage VTL (99 réf./31 fichiers) | Supprimé ; SQL DuckDB ; garde-fou ArchUnit |
| `BatchExportService` clone de `MainService` | Point d'entrée unique ; batch = runner fin sur le même use case |
| Pas de `@ControllerAdvice` | `@RestControllerAdvice` unique → `ProblemDetail` (RFC 9457) |
| Job store non exposé / doublonné | `JobStorePort` **unifié en mémoire** (non persistant — suivi temporaire rejouable, décision client) |
| Divergence `pom.xml`/`pom-public.xml` | POM unique, build = livraison |
| Config en dur (chemins Windows, URL prod) | Config typée externalisée + Vault + lint CI |
| Chiffrement non testé en CI | Module compilé et testé en CI |
| Pas de scan / supply-chain | SBOM CycloneDX + scan dépendances/image bloquant |

---

## 12. Trajectoire technique

Les incréments suivent le backlog (`../3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md`). Ordre logique : **fondations** (core + batch (+ api ?), ports/adapters, client Genesis déclaratif + Resilience4j, CI + portes qualité + SBOM, CDS) → **MVP** (export JSON incrémental + replay depuis Genesis) → **Parquet/CSV/R, multimode SQL, suivi de jobs en mémoire unifié, reporting, chiffrement, observabilité OTel** → **plus tard** (anonymisation, mode sans-DDI, script de post-traitement) → **évolutions** (multimode sans réconciliation, codification, date T). L'image native (E-01) et le contract testing (E-05) sont évalués en cours de route, non requis pour la cible.
