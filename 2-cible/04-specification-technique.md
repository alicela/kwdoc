# Kraftwerk cible — Spécification technique

**Date** : 2026-07-09  |  **Version** : 1.0  |  **Statut** : cadrage cible

Ce document décrit **comment** est construite la cible : architecture, technologies, flux d'exécution, contrats internes. Le **quoi** fonctionnel est dans `03-specification-fonctionnelle.md`. Les garde-fous anti-dette sont dans `../1-existant/06-dette-technique-et-garde-fous.md`.

> ⚠️ **Écart assumé avec le document `annexe-01-choix-architecturaux-initiaux.md`** (rédigé par « Mistral Vibe », non vérifié). Ce dernier propose **Spring WebFlux/Reactor**, **Spring Batch + PostgreSQL** et **3 artefacts séparés (api/batch)**. Après arbitrage (2026-07-09), la cible retient **des choix différents** : virtual threads + DuckDB ensembliste (pas de réactif), artefact unique, suivi de jobs derrière un port avec impl embarquée par défaut. Le présent document fait foi ; `06-` est conservé comme trace de réflexion.

---

## 1. Décisions d'architecture (tranchées vs ouvertes)

### 1.1 Tranchées ✅

| # | Décision | Justification |
|---|---|---|
| D-01 | **Java 25 + Spring Boot 4.1.0 + Maven** | Décision client (`01-objectifs-refonte.md`) |
| D-02 | **Zéro VTL/Trevas** | Décision client ; variables déjà calculées par Genesis (RG-ACQ-03) |
| D-03 | **Genesis unique source, par `partitionId`** | Décision client (RG-ACQ-01/90) |
| D-04 | **Concurrence : virtual threads (Java 25) + traitement ensembliste DuckDB** — pas de programmation réactive | Simplicité ; le gros du calcul est poussé dans DuckDB (colonnaire, spill-to-disk) ; les virtual threads couvrent la concurrence I/O |
| D-05 | **Artefact unique** : un module métier `kraftwerk-core` + un module applicatif `kraftwerk-app` (API REST + runner batch dans le même jar/image) ; mode choisi au lancement | Corrige la divergence de POM de l'audit de dette (doc 09) ; batch et API partagent 100 % du code métier |
| D-06 | **DuckDB au cœur du traitement** (jointures, split par niveau, réconciliation, écriture) | Déjà utilisé pour l'écriture ; opérations ensemblistes = clé du budget mémoire |
| D-07 | **Ports & Adapters (hexagonal)** entre le cœur métier et les technologies externes (Genesis, stockage, chiffrement, store de jobs) | Testabilité, découplage, remplaçabilité |
| D-08 | **Suivi de jobs derrière un port**, impl **embarquée fichier par défaut (DuckDB/SQLite)**, impl **PostgreSQL** activable par config | Répond au besoin « PostgreSQL si possible embarqué, sinon DuckDB » ; PostgreSQL requis dès le multi-instance |
| D-09 | **Portes qualité en CI dès le premier commit** (couverture bloquante, test d'archi « zéro VTL », scan de vulnérabilités, POM unique) | Empêche la reconstitution de la dette (doc 09 §6.2) |
| D-10 | **Liens 2à2 via BPM** (dépendance `fr.insee.bpm` conservée, aucune logique Kraftwerk) | Décision client (RG-LIEN-90) |

### 1.2 Ouvertes (à trancher en conception détaillée)

| # | Point | Proposition par défaut |
|---|---|---|
| O-01 | Format/stockage de la config d'anonymisation par enquête | Fichier de config par enquête (YAML/JSON) chargé via le port de stockage ; à valider |
| O-02 | Lever ou maintenir « pas de boucles imbriquées » | Maintenir au MVP, détecter+signaler le cas non géré |
| O-03 | Règle par défaut sur doublon inter-mode | Conserver les 2 lignes distinguées par `MODE_KRAFTWERK` ; à valider |
| O-04 | Convention de préfixe de nommage des fichiers | `<partitionId>_<TABLE>.<ext>` ; à valider |
| O-05 | PostgreSQL réellement « embarqué » | Non standard en prod → impl embarquée = DuckDB/SQLite ; PostgreSQL = déploiement externe |

---

## 2. Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────────────┐
│                        kraftwerk-app  (1 jar / 1 image)                  │
│                                                                        │
│  Mode API (défaut)                         Mode batch (--batch …)       │
│  ┌────────────────────┐                    ┌────────────────────┐      │
│  │ REST Controllers    │                    │ CommandLineRunner   │      │
│  │ + @RestControllerAdvice                  │ (parse args → même  │      │
│  │ + Security OIDC     │                    │  use case)          │      │
│  └─────────┬──────────┘                    └─────────┬──────────┘      │
│            └───────────────┬───────────────────────────┘               │
│                            ▼                                            │
│                 ┌────────────────────────┐                             │
│                 │  Use cases (kraftwerk-core)                           │
│                 │  ExportTabularUseCase / ExportJsonUseCase /           │
│                 │  ExportReportingUseCase …                             │
│                 └───────────┬────────────┘                             │
│                             │ (ports)                                   │
│   ┌───────────┬─────────────┼───────────────┬──────────────┐           │
│   ▼           ▼             ▼               ▼              ▼           │
│ GenesisPort  MetadataPort  DuckDbEngine   StoragePort   JobStorePort    │
│   │           │             │               │              │           │
│ (adapters ↓)  │             │               │              │           │
│ Genesis REST  BPM (DDI/     DuckDB (in-proc) FS / MinIO   DuckDB|SQLite │
│  client       Lunatic)                                    | PostgreSQL  │
│                                              EncryptionPort → Vault      │
└──────────────────────────────────────────────────────────────────────┘
       ▲                                    Observabilité : Micrometer,
   OIDC (JWT)                               logs structurés + correlationId
```

- **`kraftwerk-core`** : domaine + use cases + ports (interfaces). Aucune dépendance Spring Web ni technologie externe concrète.
- **`kraftwerk-app`** : adapters (controllers REST, runner batch, clients, écrivains), configuration Spring, sécurité. Dépend de core.
- **`kraftwerk-encryption`** : adapter de chiffrement (Vault + lib INSEE), activable/désactivable (profil), buildé **et testé en CI** (correction de la dette D10/doc 09).

---

## 3. Modules Maven et build

```
kraftwerk-parent (pom.xml unique — plus de pom-public.xml divergent)
├── kraftwerk-core          # domaine, use cases, ports ; DuckDB SQL ; BPM
├── kraftwerk-app           # API REST + runner batch + adapters + config
└── kraftwerk-encryption    # adapter chiffrement (Vault), profil activable
```

- **Un seul chemin de build** : l'artefact testé en CI = l'artefact livré (fin de la divergence `pom.xml`/`pom-public.xml`). Le profil « sans chiffrement » remplace l'adapter par un no-op mais **le module reste compilé/testé**.
- **BPM** épinglé à une **version publiée** (plus de build depuis `@master` flottant).
- **Renovate** enrichi (groupes, `vulnerabilityAlerts`, dashboard).
- **Docker multi-stage**, image runtime JRE 25 rootless (base INSEE), `HEALTHCHECK`, digest-pinné.

---

## 4. Pipeline de traitement (cœur performance)

Principe : **Java orchestre l'I/O, DuckDB fait le calcul ensembliste.** Objectif < 1 Go RAM (NF-001).

```
1. ACQUISITION (I/O concurrent, virtual threads)
   - GenesisPort.streamByPartition(partitionId, mode?, batchSize)
   - pagination : une page en mémoire à la fois ; INSERT en table DuckDB de staging
   - plusieurs partitions/pages en parallèle via un pool de virtual threads borné
2. STRUCTURE (SQL ensembliste)
   - métadonnées BPM → schéma des niveaux (RACINE + boucles)
   - split par niveau = requêtes SQL (pas de reconstruction par parsing de noms — cf. CL-STR-03)
3. RÉCONCILIATION MULTIMODE (SQL ensembliste) — si ≥ 2 modes
   - UNION des modes + colonne MODE_KRAFTWERK + harmonisation de types
4. ANONYMISATION (SQL) — option
   - suppression des colonnes configurées (ex. SELECT * EXCLUDE (...)) AVANT écriture
5. ÉCRITURE (streaming DuckDB)
   - COPY … TO '<fichier>' (FORMAT PARQUET) et (FORMAT CSV, …)
   - une écriture par table ; DuckDB gère le spill disque
6. POST (option) : script R, archivage ZIP, chiffrement, upload MinIO
```

**Pourquoi pas réactif** : le budget mémoire est tenu par DuckDB (colonnaire + spill), pas par un pipeline row-by-row Java. Les virtual threads Java 25 suffisent pour la concurrence I/O (pagination Genesis, partitions parallèles) sans la complexité Reactor. La `StructuredTaskScope` (Java 25) encadre le fan-out concurrent.

**Modèle DuckDB** (schéma par niveau, aligné RG-STR) :
```sql
CREATE TABLE staging_raw(interrogation_id, usual_su_id, mode, scope, iteration, var_id, value, state, questionnaire_state, validation_date, …);
-- puis, par pivot/split :
CREATE TABLE racine(interrogation_id PK, usual_su_id, questionnaire_state, validation_date, <vars scope=RACINE> …);
CREATE TABLE loop_<boucle>(interrogation_id, occurrence_id, <vars de la boucle> …, PRIMARY KEY(interrogation_id, occurrence_id));
```
Connexion DuckDB gérée par exécution (une base de travail éphémère par job) ; pooling si besoin. Parquet produit nativement par DuckDB (pas de dépendance Parquet dédiée — cf. anomalie 08 §13).

---

## 5. Contrats de ports (esquisse)

```java
// Acquisition
interface GenesisPort {
    Stream<Interrogation> streamByPartition(String partitionId, Optional<Mode> mode, int batchSize);
    Instant getLastExtraction(String collectionInstrumentId);       // JSON incrémental
    void updateLastExtraction(String collectionInstrumentId, Instant date);
    HealthStatus ping();
}
// Métadonnées de structure (BPM)
interface MetadataPort { MetadataModel forMode(String partitionId, Mode mode); } // DDI (+Lunatic), ou Lunatic seul
// Moteur de mise en forme / écriture
interface TabularEngine {                 // impl DuckDB
    void ingest(Stream<Interrogation> data);
    void splitIntoLevels(MetadataModel md);
    void reconcileModes();                 // no-op si mono-mode
    void anonymize(List<String> columnsToDrop);
    List<OutputFile> writeParquet(Path dir); List<OutputFile> writeCsv(Path dir);
}
// Stockage (disque | MinIO) — RG-ANO-21
interface StoragePort { InputStream read(String path); void write(Path local, String target); boolean isAvailable(); }
// Chiffrement (Vault) — RG-ANO-10..12
interface EncryptionPort { Path encryptArchive(Path zip); }      // no-op si profil off
// Suivi de jobs — RG-EXE-02..05
interface JobStorePort {
    JobId create(JobRequest req); void update(JobId id, JobStatus s, JobResult partial);
    Optional<JobExecution> find(JobId id);
}
```
Le cœur (use cases) ne connaît que ces interfaces ; les adapters vivent dans `kraftwerk-app`/`kraftwerk-encryption`.

---

## 6. Exécution asynchrone et suivi de jobs

- **Déclenchement** : endpoint → `202 Accepted` + `jobId` ; traitement soumis à un exécuteur de **virtual threads** (`Executors.newVirtualThreadPerTaskExecutor()` borné par sémaphore) (RG-EXE-01).
- **Store unifié persistant** derrière `JobStorePort` (RG-EXE-02) :
  - **défaut embarqué** : table `job_execution` dans un fichier DuckDB/SQLite local (une instance) ;
  - **option PostgreSQL** (activée par config) : nécessaire dès **plusieurs répliques** (statut cohérent entre pods).
- **Statuts** `RUNNING/DONE/PARTIAL/FAILED` (RG-EXE-03) ; résultat = statut + horodatages + résumé + erreurs (RG-EXE-04).
- **Au redémarrage** : les jobs `RUNNING` orphelins sont marqués `FAILED`/interrompus (CL-EXE-02).
- Schéma minimal : `job_execution(id, type, partition_id, status, started_at, ended_at, summary_json, errors_json)`.

> Le MVP peut démarrer avec l'impl embarquée fichier (persistante) ; PostgreSQL est branché quand le déploiement passe multi-instance, sans changer le use case (même port).

---

## 7. Sécurité (RG-EXE-20..22)

- **Resource server OIDC/JWT** (Spring Security), mode `NONE` pour environnements ouverts.
- Rôles `ADMIN` ⊃ `USER`/`SCHEDULER` ; export → `USER`/`SCHEDULER`, admin → `ADMIN`. Règles recartographiées sur les nouveaux endpoints (`/exports/**`, `/jobs/**`, `/reporting/**`, `/json/**`, `/actuator/**`, `/health`).
- **Auth service-à-service vers Genesis** : jeton client-credentials, rafraîchi, rejeu une fois sur 401 (adapter conservé).
- **Tests de sécurité** (`spring-security-test`) : anonyme→401, rôle insuffisant→403, rôle requis→200 — comblent le trou de couverture identifié.

---

## 8. Configuration (fin de la config en dur — dette D9/doc 09)

- **Externalisation totale** : aucune valeur d'environnement (chemins, URL Genesis, endpoints MinIO) dans `src/main/resources` livrés. `application.yml` = valeurs par défaut non sensibles uniquement.
- **Secrets via Vault** (Genesis creds, MinIO, clé AES, DB).
- Paramètres clés : `batchSize`, backend de stockage (`filesystem|minio`), backend job-store (`embedded|postgres`), profil chiffrement, seuils.
- Lint CI vérifiant l'absence de chemin absolu / URL de prod dans les resources.

---

## 9. Observabilité et erreurs

- **Logs structurés** (JSON) + **correlationId** propagé (filtre HTTP + MDC), pour tracer un export de bout en bout.
- **Micrometer** : métriques par export (durée, nb interrogations, nb erreurs, mémoire), exposées via Actuator/Prometheus.
- **Health-check** corrigé : résultat du test de stockage réellement exploité, `ping` Genesis dans un try/catch → statut dégradé au lieu d'un plantage (RG-EXE-13, corrige 08 §4).
- **`@RestControllerAdvice` unique** (RG-EXE-30) : table exception→code HTTP centralisée (400/404/500…). Distinction **erreurs récupérables** (collectées, `errors.txt` + `PARTIAL`) vs **fatales** (`FAILED`, arrêt) — plus de capture des `Error` Java (corrige 08 §8).

---

## 10. Tests (pyramide + portes)

- **Unitaires** (JUnit 5, Mockito, AssertJ) sur use cases et logique de reporting (`OUTCOME_SPOTTING`).
- **Intégration** (`@SpringBootTest` + Testcontainers pour PostgreSQL quand activé ; DuckDB in-proc sinon).
- **Fonctionnels Cucumber** rebâtis sur **stubs Genesis** (`GenesisClientStub`) paramétrés par `partitionId` — les fixtures fichier-local sont abandonnées, leurs assertions portées (cf. `05-couverture-tests-cucumber.md`).
- **Portes CI bloquantes** (doc 09 §6.2) : couverture ≥ seuil, **ArchUnit** interdisant tout import `fr.insee.vtl.*`/`trevas` et les `@*Mapping` hors contrôleur (aurait attrapé l'anomalie du job store), scan de vulnérabilités, build unique testant le chiffrement.

---

## 11. Correspondance dette → traitement technique (rappel doc 09)

| Dette existante | Traitement cible |
|---|---|
| Couplage VTL (99 réf./31 fichiers) | Supprimé ; SQL DuckDB ; garde-fou ArchUnit |
| `BatchExportService` clone de `MainService` | Point d'entrée unique ; batch = runner fin sur le même use case |
| Pas de `@ControllerAdvice` | `@RestControllerAdvice` unique |
| Job store non exposé / non persistant | `JobStorePort` unifié + persistant |
| Divergence `pom.xml`/`pom-public.xml` | POM unique, build = livraison |
| Config en dur (chemins Windows, URL prod) | Externalisation + Vault + lint CI |
| Chiffrement non testé en CI | Module compilé et testé en CI |

---

## 12. Trajectoire technique

Les incréments techniques suivent le backlog (`../3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md`). Ordre logique : **fondations** (squelette artefact unique, ports/adapters, CI+portes qualité, client Genesis + BPM) → **MVP** (acquisition → structure → Parquet) → **CSV/R, multimode SQL, JSON, reporting, anonymisation, chiffrement/MinIO, job store persistant, sécurité/rôles, observabilité** → **évolutions** (lot, multimode sans réconciliation, codification, date T).
