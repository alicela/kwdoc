# Kraftwerk v2 — Écarts fonctionnels existant → cible

**Date** : 2026-07-22  |  **Statut** : Document de référence pour arbitrages  |  **Objectif** : Lister explicitement les **écarts fonctionnels** entre l'existant et la cible (v2), dans les **deux sens** — ce qui **disparaît** et ce qui **apparaît**

---

## 📌 Contexte

Ce document consolide les **décisions d'arbitrage** et les **écarts fonctionnels** entre Kraftwerk existant et Kraftwerk cible (v2). Il se lit en **deux volets** :

- **🗑️ Volet A — Ce qui disparaît** : ce qui est **supprimé**, **externalisé** ou **différé** par rapport à l'existant. Répond à : *« Qu'est-ce qui disparaît ? »*
- **✨ Volet B — Ce qui est nouveau** : les fonctionnalités **ajoutées**, **refondues** ou **notablement améliorées** que l'existant ne proposait pas. Répond à : *« Qu'est-ce qui apparaît ? »*

> 🎯 **Périmètre** : ce document se limite aux écarts **fonctionnels** (ce que fait la cible, du point de vue métier/utilisateur). Les nouveautés purement **techniques** (virtual threads, `ProblemDetail` RFC 9457, observabilité OpenTelemetry, SBOM, client Genesis déclaratif, Resilience4j, CDS/AOT…) sont traitées dans [`04-specification-technique.md`](./04-specification-technique.md).

> ⚠️ **Important** : Ces écarts (au premier chef les suppressions) sont le résultat de choix deliberés pour :
> - Simplifier l'architecture (zéro VTL, Genesis-only)
> - Réduire la dette technique
> - Améliorer la maintenabilité
> - Se concentrer sur le cœur de métier (export des données)

---

## 🗑️ Volet A — Ce qui disparaît (fonctionnalités non reprises)

> Liste consolidée des éléments **supprimés**, **externalisés** ou **différés**. Légende d'impact : 🔴 supprimé définitivement · 🟡 conservé mais externalisé/adapté · 🔄 fusionné/amélioré · ⏸️ différé (hors MVP).

### 1. 📁 **Sources de Données et Parsing**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Lecture des fichiers Coltrane/XML locaux** | `XformsDataParser`, `LunaticXmlDataParser` | Passage à Genesis-only | Données fournies par Genesis via API REST | 🔴 **Supprimé définitivement** |
| **Parsing de fichiers XML volumineux** (`SplitterService`) | `SplitterService.splitLunaticXml()` | Sans objet en Genesis-only | Streaming pagination Genesis | 🔴 **Supprimé définitivement** |
| **Mode "fichiers locaux"** | Endpoints `/main`, `/main/file-by-file`, `/main/lunatic-only` | Déprécié au profit de Genesis | Appel unique par `partitionId` | 🔴 **Supprimé définitivement** |
| **Parseur `XformsDataParser`** | `kraftwerk-core/data/parser/XformsDataParser.java` | Inutile sans fichiers locaux | Genesis fournit les données déjà parsées | 🔴 **Supprimé définitivement** |
| **Limite de 400 Mio sur les fichiers d'entrée** | `MainProcessing.init()` | Contrôle décalé vers Genesis | Genesis gère la pagination | 🔴 **Supprimé** |

### 2. ⚙️ **Calcul et Transformation des Données**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Moteur VTL/Trevas** | `fr.insee.kraftwerk.core.vtl.*`, dépendance Trevas | Externalisation validée par le client | Variables calculées fournies par Genesis (RG-ACQ-03) | 🔴 **Supprimé définitivement** |
| **Dérivation des variables calculées** | `BuildBindingsSequence`, `VtlExecute` | Genesis fournit les variables déjà calculées | Aucune logique de calcul dans Kraftwerk | 🔴 **Supprimé définitivement** |
| **Modules TCM en VTL** | `src/main/resources/vtl/tcm/*.vtl` (16 fichiers) | Spécifiques à des enquêtes, non génériques | Liens 2à2 déportés dans BPM (RG-LIEN-90) | 🔴 **Supprimé définitivement** |
| **VTL utilisateur** (scripts `.vtl`) | Injections via `kraftwerk.json` | Remplacé par des mécanismes SQL | Script SQL/R sur tables DuckDB (RG-CSV-13) | 🔴 **Supprimé définitivement** |
| **Fichiers `unimode/<mode>.vtl`** | Ressources VTL par mode | Sans objet sans VTL | Logique réécrite en SQL | 🔴 **Supprimé définitivement** |
| **Renseignement des liens 2 à 2** | `LunaticXmlDataParser.manageTcmLiens` | Déporté dans BPM | BPM fournit les variables `LIEN_1` à `LIEN_20` | 🔴 **Supprimé** (mais fonctionnalité conservée via BPM) |

### 3. 📊 **Paradonnées (Paradata)**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Traitement des paradonnées** | `ParadataProcessing`, événements Orchestrator/Moog | Hors périmètre fonctionnel | Non traité dans la cible | 🔴 **Supprimé définitivement** |
| **Variables de paradonnées** (`NB_ORCHESTRATORS`, durées de session, etc.) | Injection dans le dataset racine | Simplification du scope | Seuls les données de reporting sont conservées | 🔴 **Supprimé** |
| **Calcul des indicateurs de passation** | Algorithmes codés en dur (ex. `PRENOM`) | Non prioritaire | À étudier dans une version ultérieure | ⏸️ **Différé** |

### 4. 📤 **Sorties et Formats**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Scripts d'import SAS** | Génération automatique | Remplacés par des scripts R | Script R accompagné de l'export (RG-CSV-09) | 🔴 **Supprimé définitivement** |
| **Archivage des fichiers d'entrée** | Fonctionnalité inutilisée | Non reprise (RG-ANO-22) | — | 🔴 **Supprimé définitivement** |

### 5. 🔄 **Traitement et Pipeline**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Mode batch redondant** | `KraftwerkBatch`, CLI avec args positionnels | Séparation API/batch confirmée | Mode batch comme Job Kubernetes, API séparée | 🔴 **Supprimé** (architecture revisitée) |
| **Endpoints pas-à-pas** (`/steps/**`) | `StepByStepService` (5 endpoints) | Sans objet sans VTL | Inspection DuckDB à l'étude | 🔴 **Supprimé définitivement** |
| **Orchestration du pipeline en VTL** | `MainProcessing.runMain()` | Remplacé par un pipeline SQL | Orchestration via DuckDB | 🔴 **Supprimé** |

### 6. 📡 **Endpoints API Dépréciés ou Supprimés**

| Endpoint | Raison | Remplacement |
|----------|--------|---------------|
| `PUT /main` | Mode fichiers locaux | `POST /api/v1/export` par `partitionId` |
| `PUT /main/file-by-file` | Mode fichiers locaux | `POST /api/v1/export` |
| `PUT /main/lunatic-only` | Mode fichiers locaux | `POST /api/v1/export` |
| `PUT /main/genesis` | Déprécié (`@Deprecated(since="3.4.1")`) | `GET /api/partitions/{partitionId}` |
| `PUT /main/genesis/lunatic-only` | Déprécié | `GET /api/partitions/{partitionId}` |
| `PUT /main/genesis/by-questionnaire` | Remplacé par `partitionId` | `GET /api/partitions/{partitionId}` |
| `PUT /main/genesis/by-questionnaire/lunatic-only` | Remplacé par `partitionId` | `GET /api/partitions/{partitionId}` |
| `PUT /steps/buildVtlBindings` | Sans objet sans VTL | — |
| `PUT /steps/buildVtlBindings/{dataMode}` | Sans objet sans VTL | — |
| `PUT /steps/unimodalProcessing` | Sans objet sans VTL | — |
| `PUT /steps/multimodalProcessing` | Sans objet sans VTL | — |
| `PUT /steps/writeOutputFiles` | Sans objet sans VTL | — |
| `PUT /steps/archive` | Archivage supprimé | — |
| `PUT /split/lunatic-xml` | Découpage XML supprimé | — |
| `PUT /reportingdata/main` | Lecture des données de reporting depuis un **fichier local** | Point d'entrée à **unifier autour de Genesis** (RG-REP-02) ; lecture par fichier **différée** (version avancée, EV-006) |

> ℹ️ Les endpoints **conservés** de l'existant (non listés ci-dessus) : `GET /jobs/{jobId}` (suivi de job, unifié), `GET /json` / `GET /json/replay` (export JSON incrémental + rejeu), `POST /debug/json` (debug par `interrogationIds`, conservé — utile à la MOA), `GET /health-check` (corrigé de ses défauts). Le nouvel appel nominal est `POST` d'export **par `partitionId`**.

### 7. 🗃️ **Suivi des Jobs et Persistance**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **`InMemoryJobStore`** (pour `/main*`) | Suivi non persistant, route non exposée | Remplacé par un suivi unifié | `InMemoryExportJobStore` unifié (RG-EXE-03) | 🔄 **Fusionné et amélioré** |
| **Persistance des jobs** | Aucune dans l'existant | Décision client : non nécessaire | Suivi **en mémoire uniquement** (rejouable) | 🟡 **Remplacé par solution non persistante** |
| **`MainAsyncService.getStatus`** | Route probablement non enregistrée | Corrigé et unifié | Suivi de job standardisé via `GET /jobs/{jobId}` | 🔄 **Amélioré** |

### 8. 🔐 **Chiffrement et Sécurité**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Chiffrement asymétrique** | Hors scope | Non nécessaire pour la V1 | Chiffrement symétrique AES + Vault | 🔴 **Supprimé** |
| **Signature des exports** | Hors scope | Non nécessaire pour la V1 | — | 🔴 **Supprimé** |

### 9. 📦 **Fonctionnalités Spécifiques Non Génériques**

| Élément | Localisation (existant) | Raison de la suppression | Remplacement / Alternative | Impact |
|---------|------------------------|--------------------------|----------------------------|--------|
| **Algorithmes de repérage codés en dur** | `OUTCOME_SPOTTING` dans reporting | Conservé mais externalisé | Calcul dans BPM ou Genesis | 🟡 **À clarifier** (RG-REP-07 confirme le maintien) |
| **Limite de 21 liens max par individu** | `MAX_LINKS_ALLOWED` (= 21) | Contrainte métier conservée mais **recalée à 20** dans la cible (`LIEN_1…LIEN_20`, RG-LIEN-02) | Gérée dans BPM | 🟡 **Contrainte maintenue, valeur ajustée (21 → 20), implémentation externalisée** |

---

## 📋 Synthèse par Domaine

### ✅ **Ce qui est clairement supprimé** (20+ éléments)

1. **Tout le VTL** : moteur, scripts, modules TCM, transformations utilisateur
2. **Tout le parsing local** : fichiers XML/Coltrane, splitters, parseurs XForms
3. **Les scripts SAS** : génération de scripts d'import
4. **Les paradonnées** : traitement et indicateurs de passation
5. **Les endpoints legacy** : `/main*`, `/steps/**`, `/split/**`
6. **L'archivage des entrées** : fonctionnalité inutilisée
7. **La persistance des jobs** : remplacée par du suivi en mémoire

### 🟡 **Ce qui est externalisé** (fonctionnalité conservée, mais plus dans Kraftwerk)

1. **Calcul des variables** → Genesis
3. **Matrice de liens 2à2** → BPM
4. **Algorithmes de repérage** → BPM/Genesis (à confirmer)

### ⏸️ **Ce qui est différé** (hors périmètre MVP, à prévoir plus tard)

1. **Anonymisation** (RG-ANO-01..04) — V4
2. **Export Parquet/CSV** (RG-CSV-01) — V2
3. **Réconciliation multimode (TCM)** (RG-MULTI-01..04) — V2
4. **Liens 2à2** (RG-LIEN-01..04) — V2
5. **Persistance des jobs** — V2
6. **Mode sans-DDI** — Attente du nouveau modèle Lunatic
7. **Lecture des données de reporting depuis des fichiers** (RG-REP-02 / EV-006) — V3

---

## 🎯 **Impact Global**

### **Réduction de Complexité**
- **~40% de code en moins** (estimation) : suppression de tout le code VTL, parsing XML, gestion des fichiers locaux
- **Dépendances supprimées** : Trevas, librairies de parsing XML complexes
- **Maintenabilité améliorée** : architecture plus simple, plus focalisée

### **Risques et Points de Vigilance**
- **Dépendance accrue à Genesis** : Kraftwerk suppose que Genesis fournit des données **complètes et valides** (variables calculées, structure cohérente)
- **Migration des scripts utilisateur** : Les scripts VTL existants devront être **réécrits en SQL/R**
- **Adaptation des consommateurs** : Les SI aval devront s'adapter à l'absence de scripts SAS

### **Bénéfices Attendus**
- **Moins de bugs** : suppression de code complexe et source d'erreurs (VTL, parsing XML)
- **Meilleure performance** : traitement direct des données JSON, pas de parsing lourd
- **Évolutivité** : architecture plus facile à faire évoluer
- **Coût de maintenance réduit** : moins de code, moins de dépendances

---

## ✨ Volet B — Ce qui est nouveau (fonctionnalités ajoutées ou refondues)

Ce volet liste les capacités **fonctionnelles** que l'existant ne proposait pas, ou qu'il proposait d'une façon si différente qu'elles sont **refondues**. Légende de statut :

- 🆕 **Nouveau** : n'existait pas dans l'existant.
- ⭐ **Refondu / notablement amélioré** : existait mais retravaillé en profondeur (comportement, portée ou ergonomie sensiblement différents).
- 🔮 **Évolution prévue** : planifiée après la cible (post-refonte), pas dans le périmètre initial.

> Les gains **techniques** associés (mémoire < 1 Go, traitement parallèle, MinIO/Kubernetes, résilience, observabilité) sont décrits dans [`04-specification-technique.md`](./04-specification-technique.md) et rappelés en NFR dans [`03-specification-fonctionnelle.md`](./03-specification-fonctionnelle.md) (§6).

### 1. 🧩 **Nouveaux concepts et conventions**

| Nouveauté | Description | Règle / réf | Statut |
|-----------|-------------|-------------|--------|
| **Notion de partition (`partitionId`)** | Nouvelle **unité d'appel et d'export** : une partition regroupe des `interrogationId` et peut contenir **plusieurs instruments de collecte** (ex. web + téléphone). Remplace `campaignId` (déprécié) et `questionnaireModelId`. | RG-ACQ-01/04 | 🆕 **Nouveau concept** |
| **Export par partition en cumulant les modes** | Cas **nominal** de la cible : un seul appel exporte toutes les interrogations d'une partition, tous modes confondus (ex-évolution « export par lot », désormais standard). | RG-ACQ-01/04 | ⭐ **Généralisé** |
| **`shortLabel` pour le nommage des fichiers** | Les fichiers de sortie sont préfixés par le **libellé court de la partition**, et non plus par le nom de campagne. | RG-CSV-02 | ⭐ **Nouveau nommage** |
| **Variable `MODE_KRAFTWERK`** | Variable ajoutée par la réconciliation pour **identifier le mode d'origine** de chaque ligne du jeu fusionné. | RG-MULTI-02 | ⭐ **Formalisé / renommé** |
| **`partitionId` renseigné dans l'export JSON** | Le bloc d'identification JSON porte désormais le `partitionId` (auparavant **laissé vide**). | RG-JSON-02 | ⭐ **Corrigé / enrichi** |

### 2. 🚀 **Nouvelles fonctionnalités métier**

| Nouveauté | Description | Règle / réf | Statut |
|-----------|-------------|-------------|--------|
| **Post-traitement par script SQL/R sur DuckDB** | Possibilité de faire passer un **script utilisateur (SQL, voire R) sur les tables finales** dans DuckDB avant écriture. Remplace fonctionnellement l'usage des anciens « scripts aval » et offre un point d'extension. | RG-CSV-13 | 🆕 **Nouveau** |
| **Anonymisation formalisée** | Fonctionnalité formalisée : **suppression de colonnes** selon une **liste paramétrée par enquête**, appliquée **en amont** (colonnes retirées du schéma, jamais matérialisées), avec filet `EXCLUDE` à l'export et interdiction d'exclure un identifiant. | RG-ANO-01..04 | 🆕 **Nouveau** (⏸️ hors MVP) |
| **Inspection de la base DuckDB de travail** | Piste à l'étude pour le diagnostic : **visualiser les tables intermédiaires** de la base de travail. Remplace le pas-à-pas VTL supprimé. | RG-EXE (nouveau) | 🆕 **À concevoir** |
| **Point d'extension « transformation multimode »** | Nouveau point d'extension permettant d'injecter une transformation lors de la réconciliation multimode. | RG-MULTI-05 | 🆕 **À concevoir** |
| **Divergence de type multimode → erreur lisible** | En cas de types divergents entre modes pour une même variable, la cible produit une **erreur explicite** (voire deux colonnes distinctes en solution d'attente) plutôt qu'une **harmonisation silencieuse**. | RG-MULTI-04 | ⭐ **Comportement nouveau** |

### 3. 🔭 **Suivi, exécution, sécurité (refondus / améliorés)**

| Nouveauté | Description | Règle / réf | Statut |
|-----------|-------------|-------------|--------|
| **Suivi de job unifié « au fil de l'eau »** | Un **seul** store de suivi (contre deux non interopérables dans l'existant), avec **progression observable en cours de traitement** et statuts `RUNNING`/`DONE`/`PARTIAL`/`FAILED`. Dans l'existant, 5 endpoints renvoyaient un `jobId` **sans point de consultation fonctionnel**. | RG-EXE-03/04 | ⭐ **Refondu** |
| **Chiffrement bout-en-bout** | Le chiffrement optionnel sécurise les données **tout au long du traitement**, pas seulement l'archive ZIP finale. | RG-ANO-11 | ⭐ **Portée étendue** |
| **Implémentation de chiffrement interchangeable** | Chiffrement isolé derrière une interface (`EncryptionPort`) : un tiers clonant le projet peut **brancher sa propre implémentation** (la lib INSEE restant interne). | RG-ANO-12 | 🆕 **Nouveau** |
| **Gestion d'erreurs centralisée** | Traitement d'erreurs **centralisé** distinguant erreurs **récupérables** (collectées → `PARTIAL`) et **fatales** (arrêt → `FAILED`), avec **codes HTTP cohérents**. L'existant choisissait le statut au cas par cas, sans handler centralisé. | RG-EXE-30/31/32 | ⭐ **Refondu** |
| **Rôle `SCHEDULER`** | Nouveau rôle de sécurité dédié à l'**orchestration/ordonnancement** des exports récurrents (en plus de `ADMIN`/`USER`). | RG-EXE-21 | 🆕 **Nouveau** |
| **Séparation API / Batch (batch = Job Kubernetes)** | Le **mode batch**, déclenché comme un **Job Kubernetes**, devient le mode d'exécution **principal** ; l'API est séparée (son utilité reste à confirmer). L'existant mêlait CLI batch et API de façon redondante. | RG-EXE-10/11 | ⭐ **Refondu** |

### 4. 🔮 **Évolutions prévues après la cible** (post-refonte)

| Évolution | Description | Réf |
|-----------|-------------|-----|
| **Export multimode sans réconciliation** | Restituer toutes les réponses par mode **distinctement**, sans fusion. | EV-002 |
| **Reporting depuis des fichiers à part** | Lecture des données de reporting depuis des fichiers (en plus de Genesis). | EV-006 / RG-REP-02 |
| **Intégration de la codification** | Envoi vers la codification + récupération des retours. | EV-003 |
| **Révision des types de variable en sortie** | Meilleur typage des variables exportées (implique Genesis). | EV-004 |
| **Export de l'état à une date T** | Valeurs et états des variables **tels qu'à une date donnée**. | EV-005 |

### 🎁 **Bénéfices apportés (nouveautés)**
- **Ouverture / extensibilité** : points d'extension (post-script SQL/R, transformation multimode) et chiffrement interchangeable ouvrent la solution à d'autres usages et à des tiers.
- **Observabilité fonctionnelle** : suivi de job « live » et gestion d'erreurs centralisée rendent les traitements **pilotables** par la MOA.
- **Protection des données** : anonymisation en amont et chiffrement bout-en-bout **minimisent l'exposition** des données sensibles.
- **Alignement Genesis** : la partition comme unité d'appel simplifie et **unifie** les cas d'usage multimode.

---

## 📚 **Références**

**Pour le volet A (ce qui disparaît)** :
- [Spécification fonctionnelle cible](./03-specification-fonctionnelle.md) — §1 « Ce que la cible cesse de faire »
- [Fonctionnalités principales existantes](../1-existant/02-fonctionnalites-principales.md)
- [Décisions client et arbitrages](./02-decisions-client-arbitrages.md)

**Pour le volet B (ce qui est nouveau)** :
- [Spécification fonctionnelle cible](./03-specification-fonctionnelle.md) — §5 « Fonctionnalités par domaine » et §8 « Évolutions »
- [Règles métier cible par domaine (RG/CN/CL)](./regles-metiers/README.md) — source faisant foi
- [Backlog MVP → cible](../3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md) · [Spécification technique cible](./04-specification-technique.md)

---

*Document généré le 2026-07-22 — À valider avec le client et l'équipe technique.*
