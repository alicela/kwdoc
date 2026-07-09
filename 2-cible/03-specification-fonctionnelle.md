# Kraftwerk cible — Spécification fonctionnelle

**Date** : 2026-07-09  |  **Version** : 1.0  |  **Statut** : cadrage cible

Ce document décrit **ce que fait** Kraftwerk dans sa version cible, du point de vue fonctionnel. Il consolide et met en récit les règles métier détaillées de `regles-metiers/` (notation RG/CN/CL — source faisant foi) ; il ne les réécrit pas. Le **comment** (architecture, technologies) est traité dans `04-specification-technique.md`. La couverture de tests associée est dans `05-couverture-tests-cucumber.md`.

---

## 1. Mission et positionnement

Kraftwerk est le service de **mise en forme et d'export des données d'enquête** de la filière post-collecte de l'INSEE. Il transforme les données brutes d'une **partition d'enquête** stockées dans **Genesis** en jeux de données exploitables (tableaux CSV/Parquet, flux JSON pour SI externes, fichiers de reporting de collecte).

**Ce que la cible cesse de faire** (par rapport à l'existant) :
- ❌ Elle ne **calcule plus aucune variable** : les variables collectées, externes **et calculées** arrivent déjà prêtes de Genesis (RG-ACQ-03). Le moteur VTL/Trevas disparaît.
- ❌ Elle ne lit plus de **fichiers de réponses locaux** : Genesis est l'unique source (RG-ACQ-90).
- ❌ Elle ne traite plus les **paradonnées** (durées de passation, etc.) — hors périmètre.
- ❌ Elle ne produit plus de **script d'import SAS** (RG-CSV-09) ni la matrice de **liens 2à2 en propre** (déportée dans BPM, RG-LIEN-90).

**Ce que la cible continue de faire** : acquisition Genesis, mise en forme tabulaire par niveau d'information (racine + boucles), réconciliation multimode automatique, export CSV/Parquet/JSON, données de reporting, chiffrement/archivage, stockage disque ou MinIO — le tout réexprimé **sans VTL** et avec un objectif de performance fort (< 1 Go de RAM pour 1800 interrogations).

---

## 2. Acteurs et cas d'usage

| Acteur | Besoin | Cas d'usage principal |
|---|---|---|
| **Gestionnaire d'enquête (CPIE)** | Obtenir les données mises en forme d'une enquête | Déclencher un export tabulaire (CSV/Parquet) d'une partition |
| **SI externe / aval** | Recevoir régulièrement les données à jour | Consommer l'export JSON incrémental |
| **Ordonnanceur / orchestration** | Automatiser les exports récurrents | Appeler l'API avec le rôle `SCHEDULER` |
| **Exploitant** | Superviser les traitements | Consulter le statut de job, le health-check, les logs |
| **Administrateur** | Piloter/diagnostiquer | Endpoints d'admin, export de debug |

---

## 3. Concepts métier

- **Partition (`partitionId`)** : unité d'appel de la cible. Une partition regroupe les interrogations d'une enquête à traiter ensemble. Remplace `campaignId` (déprécié) et `questionnaireModelId` (RG-ACQ-01).
- **Interrogation (`interrogationId`)** : une unité enquêtée répondante. Porte aussi `usualSurveyUnitId` (identifiant métier), `mode`, `questionnaireState`, `validationDate`, `isCapturedIndirectly` (RG-ACQ-06).
- **Mode de collecte** : WEB, TEL, F2F, PAPER… Une partition peut être mono- ou multimode (RG-ACQ-04).
- **Variable** : `varId`, `value`, `scope` (`RACINE` ou nom de boucle), `iteration`. Fournie par Genesis, jamais recalculée (RG-ACQ-03).
- **Niveau d'information** : un niveau = un groupe DDI = une table de sortie. `RACINE` (unité) + une table par boucle (RG-STR-01).
- **Boucle** : groupe répétable sous la racine (ex. individus d'un ménage). Un seul niveau de boucle sous la racine (pas d'imbrication, RG-STR-10).
- **État du questionnaire** : `FINISHED` (validé) ou `STARTED` (partiel), propagé en sortie (RG-ACQ-10).

---

## 4. Flux fonctionnel de bout en bout

```
   partitionId (+ options)
        │
        ▼
 ┌─────────────────┐   RG-ACQ-01/02   ┌──────────────┐  RG-ACQ-07
 │ 1. ACQUISITION   │ ◀─── Genesis ───▶ │  Métadonnées  │ ◀── BPM (DDI+Lunatic)
 │  par partitionId │  (paginé, latest) │  de structure │
 └────────┬────────┘                    └──────────────┘
          │ variables déjà calculées (RG-ACQ-03)
          ▼
 ┌─────────────────┐   RG-ACQ-04
 │ 2. PAR MODE       │  regroupement des interrogations par mode
 └────────┬────────┘
          ▼
 ┌─────────────────┐   RG-STR-01..06
 │ 3. MISE EN FORME  │  répartition en niveaux d'information
 │    (par niveau)   │  RACINE + une table par boucle ; liens 2à2 déjà présents (BPM)
 └────────┬────────┘
          ▼
 ┌─────────────────┐   RG-MULTI-01..04   (si ≥ 2 modes)
 │ 4. RÉCONCILIATION │  fusion multimode → MODE_KRAFTWERK
 │    MULTIMODE      │
 └────────┬────────┘
          ▼
 ┌─────────────────┐   RG-ANO-01..04 (option)
 │ 5. ANONYMISATION  │  suppression des colonnes configurées par enquête
 └────────┬────────┘
          ▼
 ┌─────────────────┐   RG-CSV-*, RG-JSON-*, RG-REP-*
 │ 6. EXPORT         │  CSV + Parquet (+ script R) / JSON incrémental / reporting
 └────────┬────────┘
          ▼
 ┌─────────────────┐   RG-ANO-10..21 (option)
 │ 7. ARCHIVAGE /    │  ZIP + chiffrement (clé Vault) ; stockage disque ou MinIO
 │    STOCKAGE       │
 └─────────────────┘

 Transverse : exécution asynchrone + suivi de job (RG-EXE-01..05), sécurité OIDC (RG-EXE-20..22),
              gestion d'erreurs centralisée (RG-EXE-30..32), logs & health-check (RG-CSV-10/11, RG-EXE-13).
```

---

## 5. Fonctionnalités par domaine

Chaque fonctionnalité renvoie à ses règles détaillées. Le **statut** synthétise l'effort cible : 🟢 repris tel quel · 🔵 adapté (réimplémenté, souvent sans VTL ou passage à `partitionId`) · 🟠 à concevoir · 🔴 supprimé.

### 5.1 Acquisition des données — `ACQ`
Kraftwerk récupère les données d'une partition via Genesis, par `partitionId` (🔵 RG-ACQ-01), de façon **paginée** pour maîtriser la mémoire (🟢 RG-ACQ-02). Genesis fournit les **valeurs** (collectées, externes, calculées — aucune dérivation côté Kraftwerk, 🔵 RG-ACQ-03) et BPM fournit la **structure** (types, groupes) via le DDI enrichi du Lunatic (🟢 RG-ACQ-07). Le traitement peut être filtré à un mode (🟢 RG-ACQ-05) ou porter sur tous les modes de la partition (🟢 RG-ACQ-04). Le **mode sans DDI** (Lunatic seul) est conservé pour les cas où le DDI manque (🟢 RG-ACQ-08). Les états `FINISHED`/`STARTED` et l'indicateur `isCapturedIndirectly` sont propagés (🟢 RG-ACQ-10/12) ; les états de variable `_STATE` sont ajoutables à la demande (`addStates`, 🟢 RG-ACQ-11). **Le mode « hors Genesis » et sa limite de 400 Mio sont supprimés** (🔴 RG-ACQ-90).

**Points ouverts** : comportement sur partition vide (CL-ACQ-01), variable rattachée à un groupe inconnu du DDI (CL-ACQ-04).

### 5.2 Structure des sorties et boucles — `STR`
Les données d'une interrogation sont réparties en **une table par niveau d'information** : `RACINE` (identifiants + variables d'unité) plus une table par boucle, chaque ligne de boucle portant un **identifiant d'occurrence** `<NOM_BOUCLE>-<NN>` (🟢 RG-STR-02/03/04). La logique de répartition, aujourd'hui en VTL, est **réexprimée sans VTL** (🔵 RG-STR-01, cf. domaine MULTI). Limitation conservée : **pas de boucles imbriquées**, un seul niveau de groupe sous la racine (🟢 RG-STR-10 — question ouverte : lever ou maintenir).

**Points ouverts** : fiabiliser la reconstruction de hiérarchie (aujourd'hui par parsing des noms qualifiés à `.`, fragile — CL-STR-03) ; occurrences non contiguës (CL-STR-04) ; détection explicite des boucles imbriquées (CL-STR-05).

### 5.3 Liens 2à2 — `LIEN`
La matrice de liens familiaux (pour chaque individu, sa relation aux autres individus du ménage) est **conservée fonctionnellement mais déportée dans BPM** (🟢 RG-LIEN-90). Kraftwerk **n'implémente aucune logique dédiée** : il conserve la dépendance `fr.insee.bpm` et consomme les variables `LIEN_1…LIEN_20` produites, comme n'importe quelles variables de boucle. L'ancien `manageTcmLiens` (codé en dur pour `BOUCLE_PRENOMS`, lié au mode fichier supprimé) n'est pas repris. **Hors périmètre MVP.**

### 5.4 Réconciliation multimode — `MULTI`
Quand une partition contient plusieurs modes, Kraftwerk **fusionne automatiquement** les jeux unimodaux en un jeu unique et ajoute une variable `MODE_KRAFTWERK` identifiant le mode d'origine de chaque ligne (🟢 RG-MULTI-02). La fusion réalise l'union des colonnes (variables présentes dans un seul mode conservées, 🔵 RG-MULTI-03) et harmonise les types divergents (🔵 RG-MULTI-04). Toute cette mécanique, aujourd'hui en VTL, est **à réécrire sans VTL** (🟠 RG-MULTI-90, piste DuckDB SQL validée).

**Points ouverts** : règle par défaut sur doublon inter-mode (CL-MULTI-03), conflits de type au-delà de entier→number (CL-MULTI-02), point d'extension « transformation multimode » (🟠 RG-MULTI-05).

### 5.5 Export CSV / Parquet — `CSV`
Chaque table (racine, boucles, reporting) est produite en **CSV et Parquet** (🟢 RG-CSV-01), un fichier par table (🔵 RG-CSV-02 — convention de préfixe à figer autour de `partitionId`). Format CSV : séparateur `;` (🟢 RG-CSV-03), tous champs entre guillemets (🟢 RG-CSV-04), booléens en `0`/`1` (🟢 RG-CSV-05), grands nombres sans notation scientifique (🟢 RG-CSV-06), en-tête unique (🟢 RG-CSV-07), UTF-8. Parquet : schéma dans les métadonnées, pas de ligne d'en-tête (🟢 RG-CSV-08). Un **script d'import R** accompagne l'export ; **le SAS est supprimé** (🟢/🔴 RG-CSV-09). Chaque exécution produit un **fichier de log** (🟢 RG-CSV-11) et un **fichier d'erreurs** (🔵 RG-CSV-10, contenu non-VTL), rangés dans un **sous-dossier horodaté** (🟢 RG-CSV-12).

**Points ouverts** : échappement des séparateurs/guillemets/sauts de ligne (CL-CSV-01/02), format des décimaux (CL-CSV-03), table vide (CL-CSV-04), null vs chaîne vide (CL-CSV-05).

### 5.6 Export JSON pour SI externes — `JSON`
Export au format JSON, à contenu équivalent au tabulaire (🟢 RG-JSON-01), structuré par bloc d'identification + `data` (RACINE + boucles) (🔵 RG-JSON-02 — `partitionId` désormais **renseigné**, plus vide). Caractère **incrémental** : par défaut seules les interrogations modifiées depuis la dernière extraction, avec mémorisation/mise à jour d'une date de dernière extraction (🟢 RG-JSON-03/04). Gestion UTC/heure locale mutuellement exclusives (🟢 RG-JSON-05). **Rejeu (`replay`)** sur un intervalle `[sinceDate, untilDate]` **sans** faire avancer la date de dernière extraction — fonctionnalité importante pour ne pas perturber la production (🟢 RG-JSON-06/07). Sortie vide si rien de neuf (🟢 RG-JSON-09). Endpoint de **debug par `interrogationIds`** (🟠 RG-JSON-08, à confirmer).

**Point critique** : la date de dernière extraction ne doit avancer **qu'en cas de succès complet** (CL-JSON-05).

### 5.7 Données de reporting — `REP`
Production d'un fichier `<préfixe>_REPORTINGDATA.csv/.parquet` **séparé** du fichier de données (🟢 RG-REP-01), à partir des données de suivi de collecte, au format d'entrée **inchangé** (XML ou CSV 8 colonnes, 🟢 RG-REP-03). Contient identifiants (dont `IDSTATINSEE` calculé, 🟢 RG-REP-05), états `STATE_i`, tentatives `ATTEMPT_i`, résultats `OUTCOME`, bloc d'identification et surtout **`OUTCOME_SPOTTING`** — calcul de repérage dépendant de `TYPE_SPOTTING` en trois familles d'algorithme (🟢 RG-REP-07, règle la plus stable, cohérente code/guide/tests, sans VTL). Nomenclatures d'états/tentatives/résultats conservées (🟢 RG-REP-09). Le reporting reste dans le périmètre (distinct des paradonnées exclues).

**Points ouverts** : unifier les deux points d'entrée (Genesis vs fichier) autour de Genesis (🔵 RG-REP-02) ; `TYPE_SPOTTING` absent → repli `HOUSEF2F` (CL-REP-01) ; schéma variable selon nb d'occurrences (CL-REP-04).

### 5.8 Anonymisation — `ANO`
Nouvelle fonctionnalité formalisée : l'anonymisation = **suppression de colonnes** des tables produites (🟠 RG-ANO-01), selon une **liste paramétrée par enquête** (🟠 RG-ANO-02), applicable à toutes les tables (racine, boucles, reporting — 🟠 RG-ANO-04), **avant** écriture et avant tout archivage/chiffrement (🟠 RG-ANO-03/CL-ANO-05). Remplace l'usage principal des anciens « scripts aval ».

**Point ouvert majeur** : format/stockage/transmission de la configuration par enquête (à concevoir). Cas : variable inexistante (CL-ANO-01), exclusion d'un identifiant (CL-ANO-02).

### 5.9 Chiffrement, archivage, stockage — `ANO`
Chiffrement **optionnel** (`withEncryption`, défaut off, 🟢 RG-ANO-10) de **l'archive ZIP entière** d'une exécution → `<date>.zip.enc`, dossier clair supprimé (🟢 RG-ANO-11), chiffrement **symétrique AES via clé Vault** et lib INSEE `lib_java_chiffrement` (🟢 RG-ANO-12). Archivage ZIP optionnel (🟢 RG-ANO-20). **Double backend de stockage** disque local ou MinIO/S3 derrière une interface commune (🟢 RG-ANO-21). Le chiffrement asymétrique + signature du guide est **hors périmètre** (relève de l'orchestration, 🔴 RG-ANO-13). L'archivage des fichiers d'entrée perd son sens en Genesis-only (🟠 RG-ANO-22, à retirer). **Hors périmètre MVP.**

### 5.10 Exécution, API, sécurité, erreurs — `EXE`
Les exports sont **asynchrones** : réponse immédiate `202 Accepted` + `jobId` (🟢 RG-EXE-01). La cible expose **un suivi de statut unifié et persistant** survivant au redémarrage (🟠 RG-EXE-02), avec statuts `RUNNING`/`DONE`/`PARTIAL`/`FAILED` (🔵 RG-EXE-03), un résultat consultable (statut, horodatages, résumé, erreurs — 🔵 RG-EXE-04) via un endpoint unique (🔵 RG-EXE-05). Surface d'API limitée aux traitements Genesis par `partitionId` ; endpoints legacy (`/main`, file-by-file, lunatic-only, `campaignId`, `/steps/**`, `/split/**`) **supprimés** (🔵/🔴 RG-EXE-10/11). Sécurité **OIDC/JWT** avec mode `NONE` possible (🟢 RG-EXE-20), rôles `ADMIN`/`USER`/`SCHEDULER` (🔵 RG-EXE-21), auth service-à-service vers Genesis (🟢 RG-EXE-22). **Gestion d'erreurs centralisée** (🟠 RG-EXE-30), distinction erreurs récupérables (collectées → `PARTIAL`) vs fatales (arrêt → `FAILED`) (🔵 RG-EXE-31), codes HTTP cohérents (🔵 RG-EXE-32). Health-check application/Genesis/stockage, corrigé de ses défauts actuels (🔵 RG-EXE-13).

---

## 6. Exigences non fonctionnelles (rappel, cf. `01-objectifs-refonte.md`)

| ID | Exigence | Cible mesurable |
|---|---|---|
| NF-001 | Mémoire | < 1 Go RAM pour 1800 interrogations (vs 5 Go) |
| NF-002 | Temps de traitement | réduction significative (parallélisation I/O, moteur ensembliste) |
| NF-003 | Migration progressive | bascule par fonctionnalité, sans big bang, rollback possible |
| NF-004 | Maintenabilité | zéro VTL, artefact unique, couplage réduit |
| NF-005 | Documentation | 100 % des fonctionnalités et endpoints documentés |
| NF-006 | Tests | couverture bloquante (≥ seuil), non-régression fonctionnelle |

---

## 7. Périmètre MVP et trajectoire (résumé)

Le **MVP** couvre le chemin fonctionnel le plus court produisant de la valeur : **acquisition Genesis par `partitionId` → mise en forme RACINE + boucles → export Parquet**, **sans** liens 2à2, chiffrement, anonymisation, JSON, reporting, ni réconciliation multimode (MVP mono-mode possible). Les fonctionnalités s'ajoutent ensuite par incréments jusqu'à la cible. Le découpage détaillé en epics et user stories priorisées est dans **`../3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md`**.

---

## 8. Évolutions fonctionnelles cibles (post-refonte, cf. `05`)

- **EV-001** Export par lot (recherche des `collectionInstrumentId` d'un lot).
- **EV-002** Export multimode **sans** réconciliation (toutes les réponses par mode apparaissent distinctement).
- **EV-003** Intégration de la codification (envoi + retours).
- **EV-004** Révision des types de variable en sortie (implique Genesis).
- **EV-005** Export de l'état à une **date T** (valeurs et états des variables à cette date).

---

## 9. Points ouverts consolidés (à trancher en conception)

| # | Point | Domaine | Impact |
|---|---|---|---|
| 1 | Réécriture sans VTL de la répartition en tables **et** de la réconciliation (DuckDB SQL) | STR, MULTI | Structurant |
| 2 | Format/stockage/transmission de la config d'anonymisation par enquête | ANO | Structurant |
| 3 | Backing store du suivi de job (PostgreSQL vs embarqué) | EXE | Structurant (tranché : port + embarqué par défaut, cf. spec technique) |
| 4 | Maintenir ou lever la limite « pas de boucles imbriquées » | STR | Moyen |
| 5 | Règle par défaut sur doublon inter-mode | MULTI | Moyen |
| 6 | Convention de préfixe de nommage des fichiers (`partitionId` ?) | CSV | Faible |
| 7 | Conserver l'endpoint de debug JSON et les reportings Sabiane uniquement | JSON, REP | Faible |
| 8 | Confirmer hors-périmètre : chiffrement asymétrique/signature, export automatisé (Bangles) | ANO | Faible |
