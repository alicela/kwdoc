# Kraftwerk refonte — Backlog Epics & User Stories (MVP → cible)

**Date** : 2026-07-09  |  **Version** : 1.0

Backlog priorisé pour construire la cible **de façon incrémentale**, en partant d'un **MVP à valeur démontrable** puis en ajoutant les fonctionnalités jusqu'au périmètre complet.

- Sources : `../2-cible/03-specification-fonctionnelle.md` (fonctionnel), `../2-cible/04-specification-technique.md` (technique), `../2-cible/regles-metiers/` (RG/CN/CL), `../2-cible/05-couverture-tests-cucumber.md` (tests).
- **MVP défini avec le client** : *export Parquet depuis Genesis (par `partitionId`), sans liens 2à2 ni chiffrement.*
- Convention : `EPIC-n` regroupe des `US-n.m`. Chaque US porte une **valeur**, des **critères d'acceptation** (CA), les **RG** couvertes et les **tests** cibles (`TEST-…` de la couverture Cucumber).
- Priorité : **V0** = fondations, **V1** = MVP, **V2/V3/V4** = incréments, **V5** = évolutions. Effort indicatif : S / M / L.

---

## Vue d'ensemble et ordre logique

| Vague | Epic | Objectif | Dépend de |
|---|---|---|---|
| **V0** | EPIC-0 Fondations | Squelette artefact unique, ports/adapters, CI + portes qualité, client Genesis + BPM | — |
| **V1** | EPIC-1 **MVP — Export Parquet depuis Genesis** | Acquisition `partitionId` → RACINE + boucles → Parquet, job async | EPIC-0 |
| **V2** | EPIC-2 Export CSV + scripts + logs | CSV conforme, script R, fichiers log/erreurs, sous-dossiers datés | EPIC-1 |
| **V2** | EPIC-3 Réconciliation multimode (SQL) | Fusion multimode sans VTL, `MODE_KRAFTWERK` | EPIC-1 |
| **V2** | EPIC-4 États & variables optionnelles | `STARTED/FINISHED`, `addStates`, `FILTER_RESULT`/`MISSING` | EPIC-1 |
| **V3** | EPIC-5 Suivi de jobs persistant | Store unifié persistant, statuts, reprise | EPIC-1 |
| **V3** | EPIC-6 Sécurité & API | OIDC, rôles, gestion d'erreurs centralisée, health-check | EPIC-1 |
| **V3** | EPIC-7 Export JSON SI externe | JSON incrémental + replay + debug | EPIC-2 |
| **V3** | EPIC-8 Données de reporting | Fichier reporting séparé, `OUTCOME_SPOTTING` | EPIC-2 |
| **V4** | EPIC-9 Anonymisation | Suppression de colonnes paramétrée par enquête | EPIC-2 |
| **V4** | EPIC-10 Chiffrement / archivage / MinIO | ZIP + chiffrement Vault, backend MinIO | EPIC-2, EPIC-9 |
| **V4** | EPIC-11 Liens 2à2 (via BPM) | Consommation des variables `LIEN_*` de BPM | EPIC-1 |
| **V4** | EPIC-12 Observabilité & migration | Métriques, logs structurés, feature flags, bascule | transverse |
| **V5** | EPIC-13 Évolutions | Lot, multimode sans réconciliation, codification, date T | cible atteinte |

**Chemin critique du MVP** : EPIC-0 → EPIC-1. Tout le reste est incrémental et peut être parallélisé une fois le MVP en place.

---

## EPIC-0 — Fondations (V0)
**But** : poser le socle technique et les garde-fous anti-dette avant toute fonctionnalité. *« Les portes qualité posées après coup coûtent 10× » (doc 09).*

| US | Titre | Valeur | Effort |
|---|---|---|---|
| US-0.1 | Squelette Maven **artefact unique** (`core` + `app` + `encryption`), POM unique | Fin de la divergence POM (dette D2) | M |
| US-0.2 | Structure **Ports & Adapters** + configuration Spring Boot 4.1 / Java 25 | Découplage, testabilité | M |
| US-0.3 | Pipeline **CI + portes qualité bloquantes** : couverture, ArchUnit « zéro VTL », **SBOM CycloneDX + scan dépendances/image**, **NullAway (JSpecify)**, PIT, build unique testant le chiffrement | Empêche la reconstitution de la dette + supply-chain (doc 09 §6.2 ; spec technique D-18/D-21) | M |
| US-0.4 | **Client Genesis déclaratif** (`@HttpExchange` sur `RestClient`) + **Resilience4j** (timeout/retry/circuit-breaker) + auth service-à-service (client-credentials, rejeu sur 401) + pagination | Base de toute acquisition, résiliente (D-12/D-13) | L |
| US-0.5 | **Adapter métadonnées BPM** (`MetadataPort`, DDI + Lunatic) ; BPM épinglé à une version publiée | Structure des variables | M |
| US-0.6 | **Moteur DuckDB** de travail (ingestion via appender, base éphémère par job, écriture) — squelette | Cœur du traitement ensembliste | M |
| US-0.7 | **Docker multi-stage** rootless + `HEALTHCHECK` + **CDS/AOT cache** + config typée externalisée (zéro valeur en dur) | Déploiement propre, démarrage/mémoire (dette D9 ; D-20/D-22) | S |
| US-0.8 | **Socle observabilité & erreurs** : Micrometer Observation + tracing **OTLP**, contexte par **ScopedValue**, `@RestControllerAdvice`→**ProblemDetail (RFC 9457)**, Actuator health groups | Observabilité et contrat d'erreur dès le départ (D-14/D-15/D-16/D-23) | M |

**CA transverses** : `mvn verify` échoue si couverture < seuil ou si un import `fr.insee.vtl.*`/`trevas` est présent ; l'artefact testé en CI = l'artefact livré.

---

## EPIC-1 — MVP : Export Parquet depuis Genesis (V1) ⭐
**But** : produire les tables Parquet (RACINE + boucles) d'une partition Genesis, de façon asynchrone. **Sans** liens 2à2, chiffrement, anonymisation, CSV, JSON, reporting, ni réconciliation multimode (MVP mono-mode accepté).
**Démo cible** : `POST /exports/parquet?partitionId=…` → `202 + jobId` → fichiers `<partitionId>_RACINE.parquet` (+ tables de boucle) déposés sur disque.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-1.1 | Acquérir une partition Genesis par `partitionId` (paginé, virtual threads) | RG-ACQ-01/02/06 | TEST-ACQ-01 | L |
| US-1.2 | Ingérer les variables reçues **sans aucun calcul** (collectées/externes/calculées telles quelles) | RG-ACQ-03 | TEST-ACQ-08 | M |
| US-1.3 | Construire le schéma des niveaux d'information depuis les métadonnées BPM (RACINE + boucles) | RG-ACQ-07, RG-STR-01/06 | TEST-STR-01 | L |
| US-1.4 | Répartir les variables en table **RACINE** (identifiants + variables d'unité) — en **SQL DuckDB** | RG-STR-02/05 | TEST-STR-01 | M |
| US-1.5 | Répartir les variables de boucle en **tables de boucle** avec identifiant d'occurrence `<BOUCLE>-<NN>` | RG-STR-03/04 | TEST-STR-02/03 | M |
| US-1.6 | Écrire chaque table en **Parquet** (`COPY … TO … FORMAT PARQUET`) dans un sous-dossier d'exécution | RG-CSV-01/08/12 | TEST-CSV-02 | M |
| US-1.7 | Déclencher l'export en **asynchrone** (`202` + `jobId`) sur exécuteur virtual-threads ; statut consultable (en mémoire au MVP) | RG-EXE-01/03/05 | TEST-EXE-01/02 | M |

**Hors MVP explicitement** : liens 2à2 (EPIC-11), chiffrement (EPIC-10), anonymisation (EPIC-9), CSV (EPIC-2), multimode (EPIC-3), persistance job (EPIC-5).
**Dette assumée au MVP** : suivi de job **en mémoire** (non persistant) — remboursée par EPIC-5.

---

## EPIC-2 — Export CSV, scripts d'import, logs (V2)
**But** : compléter le MVP Parquet par le CSV conforme et les artefacts d'exécution.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-2.1 | Écrire chaque table en **CSV** : séparateur `;`, guillemets systématiques, booléens 0/1, UTF-8, en-tête unique | RG-CSV-03/04/05/07 | TEST-CSV-01 | M |
| US-2.2 | Grands nombres **sans notation scientifique** ; format des décimaux figé | RG-CSV-06 / CL-CSV-03 | TEST-CSV-03 | S |
| US-2.3 | Cohérence CSV ↔ Parquet (mêmes colonnes, N−1 lignes en Parquet) | RG-CSV-08 | TEST-CSV-02 | S |
| US-2.4 | Produire le **script d'import R** ; **ne pas** produire de SAS | RG-CSV-09 | TEST-CSV-05 | M |
| US-2.5 | Produire le **fichier de log d'exécution** et le **fichier d'erreurs** (contenu non-VTL) par sous-dossier daté | RG-CSV-10/11/12 | TEST-EXE-06/07 | M |
| US-2.6 | Échappement CSV (`;`, `"`, saut de ligne), null vs chaîne vide, table vide | CL-CSV-01/02/04/05 | TEST-CSV-06/08/10 | M |

---

## EPIC-3 — Réconciliation multimode sans VTL (V2)
**But** : fusionner automatiquement plusieurs modes en SQL DuckDB (remplace la logique VTL). Point de **non-régression prioritaire**.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-3.1 | Fusionner les jeux unimodaux en un jeu unique (UNION SQL) + colonne `MODE_KRAFTWERK` | RG-MULTI-01/02 | TEST-MULTI-01 | L |
| US-3.2 | Union des variables présentes dans un seul mode (valeurs nulles pour les autres) | RG-MULTI-03 | TEST-MULTI-02 | M |
| US-3.3 | Harmonisation des types divergents (règles de promotion spécifiées) | RG-MULTI-04 / CL-MULTI-02 | TEST-MULTI-03 | M |
| US-3.4 | Règle par défaut sur **doublon inter-mode** (décision O-03) | CL-MULTI-03 | TEST-MULTI-04 | M |
| US-3.5 | Comportement mono-mode (`MODE_KRAFTWERK` présent ou non — à figer) | CN-MULTI-02 | TEST-MULTI-05 | S |

---

## EPIC-4 — États et variables optionnelles (V2)
**But** : couvrir les données partielles et les variables d'état/filtre/non-réponse.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-4.1 | Propager `questionnaireState` (`FINISHED`/`STARTED`) en sortie | RG-ACQ-10 / CL-ACQ-02 | TEST-ACQ-04/05 | S |
| US-4.2 | Option `addStates` : variables `_STATE`, en excluant `FILTER_RESULT_*`/`*_MISSING` | RG-ACQ-11 | TEST-ACQ-06 | M |
| US-4.3 | Restituer `FILTER_RESULT_*` / `*_MISSING` reçues de Genesis (non recalculées) | RG-ACQ-03 | TEST-ACQ-07 | S |
| US-4.4 | Restituer `isCapturedIndirectly` ; gérer `validationDate` absente | RG-ACQ-12 / CL-ACQ-03 | TEST-ACQ-13/10 | S |
| US-4.5 | Mode **sans DDI** (Lunatic seul) via Genesis | RG-ACQ-08 | TEST-ACQ-09 | M |

---

## EPIC-5 — Suivi de jobs persistant unifié (V3)
**But** : rembourser la dette du MVP (job en mémoire) — store unifié persistant survivant au redémarrage.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-5.1 | `JobStorePort` en **Spring Data JDBC** + impl **embarquée fichier (DuckDB/SQLite)** par défaut | RG-EXE-02/03/04 (D-17) | TEST-EXE-02 | M |
| US-5.2 | Persistance survivant au redémarrage ; jobs `RUNNING` orphelins → `FAILED` | RG-EXE-02 / CL-EXE-02 | TEST-EXE-04 | M |
| US-5.3 | Impl **PostgreSQL** activable par config + **migrations Flyway** versionnées (multi-instance) | RG-EXE-02 (D-17) | TEST-EXE-04 | M |
| US-5.4 | Statut `PARTIAL` sur erreurs non bloquantes ; distinction récupérable/fatale | RG-EXE-03/31 | TEST-EXE-03/10 | M |
| US-5.5 | `jobId` invalide → 400 / inconnu → 404 (réponses structurées) | RG-EXE-32 / CL-EXE-01 | TEST-EXE-05 | S |

---

## EPIC-6 — Sécurité, API, erreurs, health-check (V3)
**But** : durcir la surface API et centraliser les erreurs.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-6.1 | Auth **OIDC/JWT** (+ mode `NONE`) sur la surface cible ; endpoints legacy retirés | RG-EXE-10/20 | — | M |
| US-6.2 | Rôles `ADMIN/USER/SCHEDULER` recartographiés sur les nouveaux endpoints | RG-EXE-21 | TEST-EXE-09 | M |
| US-6.3 | **`@RestControllerAdvice` unique** → réponses **`ProblemDetail` (RFC 9457)** + table exception→code HTTP | RG-EXE-30/32 (D-14) | — | M |
| US-6.4 | Health-check application/Genesis/stockage **corrigé** (pas de plantage, statut dégradé) | RG-EXE-13 / CL-EXE-03 | TEST-EXE-08 | S |
| US-6.5 | Documentation OpenAPI complète (plus de `@Operation` vides) | RG-EXE-12 | — | S |

---

## EPIC-7 — Export JSON SI externe (V3)
**But** : export JSON incrémental + replay (aucune couverture aujourd'hui).

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-7.1 | Export JSON structure complète (bloc identification + `data` RACINE + boucles), `partitionId` **renseigné** | RG-JSON-01/02 | TEST-JSON-01/02 | L |
| US-7.2 | Incrémental par `sinceDate` / dernière extraction ; MAJ date **au succès seulement** | RG-JSON-03/04 / CL-JSON-05 | TEST-JSON-03/09 | M |
| US-7.3 | **Replay** `[sinceDate, untilDate]` sans faire avancer la date d'extraction | RG-JSON-06/07 | TEST-JSON-04/05 | M |
| US-7.4 | UTC/local exclusifs ; sortie vide/204 si rien de neuf | RG-JSON-05/09 | TEST-JSON-06/08 | S |
| US-7.5 | Debug par `interrogationIds` (échecs isolés) — si conservé (O-07) | RG-JSON-08 | TEST-JSON-07 | M |

---

## EPIC-8 — Données de reporting (V3)
**But** : fichier reporting séparé, avec `OUTCOME_SPOTTING`.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-8.1 | Produire `<préfixe>_REPORTINGDATA.csv/.parquet` **séparé** ; entrée XML/CSV 8 colonnes inchangée | RG-REP-01/03 | TEST-REP (existants) | M |
| US-8.2 | Variables produites (états, tentatives, `IDSTATINSEE`, dates, nomenclatures) | RG-REP-04/05/08/09 | TEST-REP | M |
| US-8.3 | Calcul **`OUTCOME_SPOTTING`** 3 familles (`HOUSEF2F`/`IASCO`, `INDTEL`, `INDF2F`) | RG-REP-07 | TEST-REP-10 | L |
| US-8.4 | Repli `TYPE_SPOTTING` absent → `HOUSEF2F` ; bloc identification incomplet → valeur définie | CL-REP-01/02 | TEST-REP-11/12 | S |
| US-8.5 | Unifier le point d'entrée reporting sur Genesis/`partitionId` (retrait chemin fichier) | RG-REP-02 | — | M |

---

## EPIC-9 — Anonymisation (V4)
**But** : suppression de colonnes paramétrée par enquête (nouvelle fonctionnalité).

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-9.1 | Concevoir le **format de config d'anonymisation par enquête** (décision O-01) | RG-ANO-02 | — | M |
| US-9.2 | Supprimer les colonnes configurées de **toutes** les tables (SQL), **avant** écriture | RG-ANO-01/03/04 | TEST-ANO-01/02 | M |
| US-9.3 | Cas : variable inexistante (ignorer/signaler), exclusion d'un identifiant (interdire ?) | CL-ANO-01/02 | TEST-ANO-04 | S |

---

## EPIC-10 — Chiffrement, archivage, MinIO (V4)
**But** : sécuriser et router les sorties.

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-10.1 | Archivage ZIP optionnel (`archiveAtEnd`) des sorties d'une exécution | RG-ANO-20 | — | S |
| US-10.2 | Chiffrement **de l'archive entière** (`withEncryption`) → `.zip.enc`, dossier clair supprimé, clé Vault | RG-ANO-10/11/12 | TEST-ANO-05 (portage) | M |
| US-10.3 | Anonymisation appliquée **avant** chiffrement (ordre garanti) | CL-ANO-05 | TEST-ANO-03 | S |
| US-10.4 | Backend **MinIO/S3** via `StoragePort` ; Vault/MinIO indisponible → échec propre | RG-ANO-21 / CL-ANO-03/04 | TEST-ANO-06/07 | M |

---

## EPIC-11 — Liens 2à2 via BPM (V4)
**But** : consommer les variables de liens produites par BPM (aucune logique Kraftwerk).

| US | Titre | RG | Tests | Effort |
|---|---|---|---|---|
| US-11.1 | Restituer les colonnes `LIEN_1…LIEN_20` de la table de boucle telles que produites par BPM | RG-LIEN-01/04/90 | TEST-LIEN-01 | S |
| US-11.2 | Vérifier sentinelles `0`/`99` et cas ménage 1/N individus (validation des données BPM) | RG-LIEN-03 / CN-LIEN-01/02 | TEST-LIEN-02/03 | S |

> Aucun calcul dans Kraftwerk : simple consommation. Effort faible côté Kraftwerk ; toute évolution (borne 20, autres enquêtes) est portée dans **BPM**.

---

## EPIC-12 — Observabilité & migration (transverse, V4)
**But** : rendre la cible exploitable et permettre la bascule progressive.

| US | Titre | RG/Réf | Effort |
|---|---|---|---|
| US-12.1 | Logs structurés JSON corrélés au `traceId` (contexte **ScopedValue**) | spec technique §9 (D-16) | M |
| US-12.2 | **Micrometer Observation + tracing OTLP** (métriques + traces distribuées jusqu'à Genesis) + métriques Resilience4j | spec technique §9 (D-15) | M |
| US-12.3 | **Feature flags** de bascule ancienne/nouvelle version par fonctionnalité (Strangler Fig) | NF-003 | M |
| US-12.4 | Benchmark mémoire/temps par vague : valider **< 1 Go / 1800 interrogations** ; évaluer image native (E-01) vs CDS | NF-001/002 | M |

---

## EPIC-13 — Évolutions fonctionnelles (V5, post-cible)
**But** : évolutions demandées, une fois la cible atteinte (cf. `01-objectifs-refonte.md`).

| US | Titre | Réf | Effort |
|---|---|---|---|
| US-13.1 | Export **par lot** (résolution des `collectionInstrumentId` d'un lot) | EV-001 | L |
| US-13.2 | Export **multimode sans réconciliation** (réponses par mode distinctes) | EV-002 | M |
| US-13.3 | Intégration **codification** (envoi + retours) | EV-003 | L |
| US-13.4 | Révision des **types de variable** en sortie (implique Genesis) | EV-004 | L |
| US-13.5 | Export de l'état à une **date T** | EV-005 | L |

---

## Jalons

| Jalon | Contenu | Critère de sortie |
|---|---|---|
| **M0 — Socle** | EPIC-0 | CI verte + portes qualité actives ; client Genesis + BPM opérationnels |
| **M1 — MVP** | EPIC-1 | Parquet RACINE+boucles d'une partition réelle, async ; démo client |
| **M2 — Export complet** | EPIC-2/3/4 | CSV+R, multimode SQL, états — parité fonctionnelle « export » |
| **M3 — Service robuste** | EPIC-5/6 | jobs persistants, sécurité, erreurs centralisées, health-check |
| **M4 — Parité cible** | EPIC-7/8/9/10/11/12 | JSON, reporting, anonymisation, chiffrement/MinIO, liens BPM ; **< 1 Go validé** ; bascule prod |
| **M5 — Évolutions** | EPIC-13 | selon priorisation client |

## Notes de priorisation
- **Non-régression prioritaire** : EPIC-3 (multimode réécrit sans VTL) et EPIC-8 (`OUTCOME_SPOTTING`) sont les logiques métier les plus à risque — à couvrir tôt par tests.
- **Zones sans aucune couverture aujourd'hui** (cf. couverture Cucumber) : JSON (EPIC-7), jobs async (EPIC-5/6), multimode réel (EPIC-3), anonymisation (EPIC-9) — prévoir l'effort de test associé.
- Ce backlog **remplace** la roadmap indicative de `annexe-02-synthese-executive-initiale.md` (rédigée avant les arbitrages du 2026-07-09 : réactif → virtual threads, 3 artefacts → artefact unique). En cas de divergence, ce document fait foi.
