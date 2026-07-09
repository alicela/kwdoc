# Kraftwerk cible — Couverture Cucumber : tests manquants pour 100 % des fonctionnalités conservées

## Objet et méthode

Ce document identifie les **scénarios Cucumber manquants** pour couvrir à 100 % les fonctionnalités **conservées** dans la cible. Il croise :
- l'inventaire **exhaustif** des `.feature` **existants** — **20 fichiers** répartis sur 3 modules : `kraftwerk-functional-tests` (15), `kraftwerk-encryption-tests` (1) et **`kraftwerk-core`** (4) — et ce qu'ils vérifient réellement,
- les **règles métier cible** (`../regles-metiers/`, notation RG/CN/CL).

> 🛠️ **Révision 2026-07-09** : l'inventaire initial n'avait balayé que `functional-tests` + `encryption-tests` (16 fichiers) et **omettait les 4 `.feature` du module `kraftwerk-core`** (`do_we_aggregate`, `do_we_apply_vtl_instruction`, `do_we_apply_vtl_script`, `do_we_export_datasets`). Ils sont désormais intégrés (§1) et vérifiés ligne à ligne. Deux d'entre eux sont des actifs fonctionnels à conserver (agrégation multimode, aller-retour export/import).

**Définition retenue de « couverture à 100 % »** : chaque règle de gestion au statut 🟢 Conservé / 🔵 Adapté / 🟠 À concevoir possède **au moins un scénario nominal (CN)**, et chaque **edge case (CL)** documenté possède **un scénario** dédié. Les règles 🔴 Supprimé sont hors périmètre (leurs tests doivent être retirés).

**Convention d'identifiant** : `TEST-<DOM>-<NN>`. Statut : `EXISTANT` (déjà couvert), `À PORTER` (assertion existante à rejouer sur fixture Genesis/`partitionId`), `À ADAPTER` (existe mais logique à modifier), `NOUVEAU` (aucune couverture aujourd'hui).

> ⚠️ **Impact transversal majeur** : la suppression du mode « hors Genesis » (RG-ACQ-90) rend caduques la plupart des fixtures actuelles (fichiers locaux + `kraftwerk.json`). Leurs *assertions* (format CSV, boucles, incrément, numérique, métadonnées) restent valides et doivent être **portées sur des fixtures Genesis** (`GenesisClientStub`), pas perdues. C'est le principal poste de travail de reprise.

---

## 1. Sort des fichiers `.feature` existants

| Fichier existant | Fonctionnalité | Sort en cible |
|---|---|---|
| `do_we_export_data_genesis.feature` | Export Genesis (campaignId + questionnaireModelId) | 🔵 **À ADAPTER** → `partitionId` |
| `do_we_export_loop_genesis.feature` | Boucles via Genesis (TABESA, TABOFATS) | 🔵 **À ADAPTER** → `partitionId` (bon socle) |
| `do_we_export_reporting_data.feature` | Reporting (10 scénarios, OUTCOME_SPOTTING…) | 🟢 **CONSERVER** (adapter chemins d'entrée) |
| `out_data_encryption.feature` | Chiffrement archive (4 scénarios) | 🔵 **À ADAPTER** → `partitionId` |
| `do_we_get_metadata_correctly.feature` | Modèle de métadonnées DDI vs Lunatic | 🟢 **CONSERVER** (métadonnées via BPM maintenues) |
| `do_we_generate_logs.feature` | Fichier de log | 🔵 **À PORTER** sur Genesis |
| `do_we_export_data.feature` | Structure CSV/Parquet racine | 🔵 **À PORTER** (assertions format → Genesis) |
| `do_we_export_numeric.feature` | Grands nombres sans notation scientifique | 🔵 **À PORTER** |
| `do_we_increment.feature` | En-tête unique / incrément | 🔵 **À PORTER** |
| `do_we_get_loops.feature` | Boucles (B_PRENOMREP) | 🔵 **À PORTER** (redondant partiel avec loop_genesis) |
| `do_we_export_data_lunatic_only.feature` | Mode sans DDI | 🔵 **À PORTER** sur Genesis (mode sans DDI conservé, RG-ACQ-08) |
| `do_we_export_data_file_by_file.feature` | Traitement fichier par fichier | 🔴 **SUPPRIMER** le mode ; porter les assertions loops. « file-by-file » contournait la limite 400 Mo (supprimée) |
| `do_we_generate_errors.feature` | Fichier d'erreurs (déclencheur = VTL invalide) | 🔵 **À ADAPTER** — nouvelle condition d'erreur non-VTL |
| `do_we_apply_vtl.feature` | Application de scripts VTL (2 scénarios : `calc`, `drop root`) | 🔴 **SUPPRIMER** (VTL retiré) — voir report des capacités ci-dessous |
| `do_we_execute_TCM_script.feature` | Script TCM — **intégralement commenté** (fichier mort, aucun `Feature:` actif) | 🔴 **SUPPRIMER** |
| `do_we_get_paradata.feature` | Paradonnées | 🔴 **SUPPRIMER** (paradonnées exclues, RG A) |
| `core/do_we_aggregate.feature` | Agrégation de 2 `VTLBindings` (CAWI + PAPI) en un jeu agrégé | 🔵 **À ADAPTER** → **amorce de la réconciliation multimode** ; conserver l'intention (fusionner 2 modes), réécrire sans `VTLBindings`/VTL → alimente **TEST-MULTI-01** |
| `core/do_we_apply_vtl_instruction.feature` | Instruction VTL unique (`keep`/`calc`/`rename`/`drop`), comptage de variables | 🔴 **SUPPRIMER** (VTL retiré) — capacités reportées ci-dessous |
| `core/do_we_apply_vtl_script.feature` | Enchaînement de scripts VTL, dont `union(...)` | 🔴 **SUPPRIMER** (VTL retiré) — `union` → réconciliation (**TEST-MULTI-01/02**) |
| `core/do_we_export_datasets.feature` | Aller-retour export→réimport d'un `SurveyRawData` (COLEMAN, PAPER), intégrité des valeurs | 🔵 **À PORTER** → test d'intégrité round-trip (écriture/relecture DuckDB), à conserver sans `VTLBindings` |

**Report des capacités des features VTL supprimées** (`do_we_apply_vtl.feature` + `core/do_we_apply_vtl_instruction.feature` + `core/do_we_apply_vtl_script.feature`) — VTL supprimé mais les besoins subsistent, réexprimés sans VTL :
- `calc` (variable calculée) → désormais fournie par Genesis → couvert par **TEST-ACQ-08**.
- `drop` (suppression de variable) → désormais anonymisation → couvert par **TEST-ANO-01**.
- reconciliation / transformation → **TEST-MULTI-01/02**.
- niveaux d'information (`RACINE` keep/drop) → **TEST-STR-01/06**.

---

## 2. Tests manquants par domaine

### ACQ — Acquisition Genesis

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-ACQ-01 | Export d'une partition par `partitionId` | récupération + sortie correcte à partir d'un `partitionId` | RG-ACQ-01, CN-ACQ-01 | NOUVEAU | **P1** |
| TEST-ACQ-02 | Partition multimode (WEB+TEL) | toutes les interrogations des 2 modes traitées | RG-ACQ-04, CN-ACQ-02 | NOUVEAU | **P1** |
| TEST-ACQ-03 | Filtre `dataMode` précisé | seul le mode demandé est extrait/traité | RG-ACQ-05, CN-ACQ-03 | NOUVEAU | P2 |
| TEST-ACQ-04 | Questionnaire `FINISHED` en sortie | colonne `questionnaireState=FINISHED` présente | RG-ACQ-10 | NOUVEAU | **P1** |
| TEST-ACQ-05 | Questionnaire partiel `STARTED` en sortie | l'interrogation partielle apparaît avec `STARTED` | RG-ACQ-10, CL-ACQ-02 | NOUVEAU | **P1** |
| TEST-ACQ-06 | `addStates=true` | colonnes `_STATE` ajoutées, sauf pour `FILTER_RESULT_*`/`*_MISSING` | RG-ACQ-11 | NOUVEAU | P2 |
| TEST-ACQ-07 | Variables `FILTER_RESULT_*` / `*_MISSING` reçues de Genesis | présentes en sortie telles quelles (non recalculées) | RG-ACQ-03 | NOUVEAU | P2 |
| TEST-ACQ-08 | Variable calculée fournie par Genesis | présente en sortie sans dérivation Kraftwerk | RG-ACQ-03 | NOUVEAU | **P1** |
| TEST-ACQ-09 | Mode sans DDI via Genesis | sortie correcte à partir du seul modèle Lunatic | RG-ACQ-08 | À PORTER | P2 |
| TEST-ACQ-10 | `validationDate` absente | cas `null` géré sans erreur | CL-ACQ-03 | NOUVEAU | P3 |
| TEST-ACQ-11 | Partition vide | réponse/sortie « aucune donnée » propre (pas d'erreur) | CL-ACQ-01 | NOUVEAU | P2 |
| TEST-ACQ-12 | Variable d'un groupe absent des métadonnées | ignorée/tracée explicitement (pas de sortie corrompue) | CL-ACQ-04 | NOUVEAU | P3 |
| TEST-ACQ-13 | `isCapturedIndirectly` en sortie | indicateur restitué | RG-ACQ-12 | NOUVEAU | P3 |

### STR — Structure / boucles

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-STR-01 | Table `RACINE` seule (sans boucle) | 1 ligne/interrogation, identifiants fixes | RG-STR-02, CN-STR-01 | À PORTER | **P1** |
| TEST-STR-02 | Table de boucle (valeurs + itérations) | N lignes/interrogation, valeurs par itération | RG-STR-03, CN-STR-02 | À ADAPTER (loop_genesis) | **P1** |
| TEST-STR-03 | Identifiant d'occurrence `<BOUCLE>-<NN>` | présence + format de l'identifiant d'occurrence | RG-STR-04 | NOUVEAU | P2 |
| TEST-STR-04 | Plusieurs boucles au même niveau | une table par boucle, toutes filles de racine | CN-STR-03 | NOUVEAU | P2 |
| TEST-STR-05 | Boucle sans occurrence pour une interrogation | pas de ligne fantôme dans la table de boucle | CL-STR-01 | NOUVEAU | P2 |
| TEST-STR-06 | Itérations non contiguës | numérotation d'occurrence correcte | CL-STR-04 | NOUVEAU | P3 |
| TEST-STR-07 | Détection de boucle imbriquée | rejet/signalement explicite (pas de résultat faux) | RG-STR-10, CL-STR-05 | NOUVEAU | P2 |
| TEST-STR-08 | Nom de variable contenant `.` | pas de rupture silencieuse de la hiérarchie | CL-STR-03 | NOUVEAU | P3 |

### LIEN — Liens 2à2 (calculés par BPM, consommés par Kraftwerk)

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-LIEN-01 | Matrice de liens en sortie | colonnes `LIEN_1..LIEN_20` présentes dans la table de boucle individus | RG-LIEN-01/04 | NOUVEAU | P2 |
| TEST-LIEN-02 | Ménage de N individus | valeurs réelles + sentinelle `"0"` (soi-même) + `"99"` (hors matrice) | RG-LIEN-03, CN-LIEN-01 | NOUVEAU | P2 |
| TEST-LIEN-03 | Ménage d'1 individu | `LIEN_1="0"`, reste `"99"` | CN-LIEN-02 | NOUVEAU | P3 |

> Ces tests valident que Kraftwerk **restitue correctement** la matrice produite par BPM. Le calcul lui-même relève des tests de BPM.

### MULTI — Réconciliation multimode (réécrite sans VTL)

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-MULTI-01 | Fusion de 2 modes | jeu unique + colonne `MODE_KRAFTWERK` valorisée | RG-MULTI-01/02, CN-MULTI-01 | À ADAPTER (`core/do_we_aggregate` — amorce à réécrire sans VTL + à hisser en partition Genesis réelle) | **P1** |
| TEST-MULTI-02 | Variable présente dans un seul mode | union avec valeurs nulles pour l'autre mode | RG-MULTI-03, CL-MULTI-01 | NOUVEAU | **P1** |
| TEST-MULTI-03 | Conflit de type entre modes | harmonisation de type déterministe | RG-MULTI-04, CL-MULTI-02 | NOUVEAU | P2 |
| TEST-MULTI-04 | Doublon d'interrogation inter-modes | règle de gestion par défaut appliquée | CL-MULTI-03 | NOUVEAU | P2 |
| TEST-MULTI-05 | Partition mono-mode | comportement de `MODE_KRAFTWERK` défini (présent ? ) | CN-MULTI-02 | NOUVEAU | P3 |

### CSV — Export CSV / Parquet

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-CSV-01 | Format CSV (guillemets, séparateur `;`, booléens 0/1) | conformité de format | RG-CSV-03/04/05 | À PORTER | **P1** |
| TEST-CSV-02 | Cohérence CSV/Parquet | mêmes colonnes, N−1 lignes en Parquet | RG-CSV-08 | À PORTER | **P1** |
| TEST-CSV-03 | Grand entier sans notation scientifique | `19800340` et non `1.98E7` | RG-CSV-06 | À PORTER | P2 |
| TEST-CSV-04 | En-tête unique après retraitement | pas de ré-ajout d'en-tête | RG-CSV-07 | À PORTER | P2 |
| TEST-CSV-05 | Script d'import R produit, SAS **absent** | présence R, absence SAS | RG-CSV-09 | NOUVEAU | P2 |
| TEST-CSV-06 | Valeur contenant `;`, `"` ou saut de ligne | échappement correct | CL-CSV-01/02 | NOUVEAU | P2 |
| TEST-CSV-07 | Formatage des décimaux | séparateur/précision/absence d'exposant | CL-CSV-03 | NOUVEAU | P3 |
| TEST-CSV-08 | Valeur nulle vs chaîne vide | distinction en CSV (`""`) et Parquet (`NULL`) | CL-CSV-05 | NOUVEAU | P3 |
| TEST-CSV-09 | Caractères non-ASCII (UTF-8) | encodage préservé en sortie | CL-CSV-06 | NOUVEAU | P3 |
| TEST-CSV-10 | Table vide | fichier avec en-tête seul ou absence — comportement figé | CL-CSV-04 | NOUVEAU | P3 |

### JSON — Export SI externe *(aucune couverture Cucumber aujourd'hui)*

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-JSON-01 | Export JSON structure complète | bloc d'identification + `data` (RACINE + boucles) | RG-JSON-01/02, CN-JSON-01 | NOUVEAU | **P1** |
| TEST-JSON-02 | `partitionId` valorisé dans le JSON | plus vide, cohérent avec l'appel | RG-JSON-02 | NOUVEAU | **P1** |
| TEST-JSON-03 | Export incrémental depuis dernière extraction | seules les interrogations modifiées ; date mise à jour | RG-JSON-03/04 | NOUVEAU | **P1** |
| TEST-JSON-04 | Replay ne met pas à jour la date d'extraction | date de dernière extraction inchangée | RG-JSON-06, CN-JSON-02 | NOUVEAU | **P1** |
| TEST-JSON-05 | Bornes de replay invalides | `untilDate<sinceDate` → 400 | RG-JSON-07, CL-JSON-02 | NOUVEAU | P2 |
| TEST-JSON-06 | Aucune donnée nouvelle | sortie vide/204, pas d'erreur | RG-JSON-09, CL-JSON-01 | NOUVEAU | P2 |
| TEST-JSON-07 | Debug par `interrogationIds` | isolation des échecs par id | RG-JSON-08, CL-JSON-03 | NOUVEAU | P3 |
| TEST-JSON-08 | Dates UTC/local mutuellement exclusives | 400 si incohérence | RG-JSON-05 | NOUVEAU | P3 |
| TEST-JSON-09 | Échec après export : date non avancée | pas de perte de données au prochain export | CL-JSON-05 | NOUVEAU | P2 |

### REP — Données de reporting *(bien couvert — quelques compléments)*

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-REP-01..09 | (existants) fichier séparé, OUTCOME_SPOTTING, états, tentatives, dates, nouvelles variables | — | EXISTANT | — |
| TEST-REP-10 | `OUTCOME_SPOTTING` branche `INDF2F`/`INF2FNOR` avec `NOTREAT` | couverture de la 3ᵉ famille d'algorithme | RG-REP-07(c) | NOUVEAU | P2 |
| TEST-REP-11 | `TYPE_SPOTTING` inconnu/absent | repli sur cascade `HOUSEF2F` par défaut | CL-REP-01 | NOUVEAU | P3 |
| TEST-REP-12 | Bloc identification incomplet | `OUTCOME_SPOTTING` = valeur définie (vide), pas d'échec | CL-REP-02 | NOUVEAU | P3 |
| TEST-REP-13 | En-tête CSV reporting non conforme | erreur explicite | CL-REP-03 | NOUVEAU | P3 |

### ANO — Anonymisation, chiffrement, stockage

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-ANO-01 | Suppression de colonnes paramétrée par enquête | colonnes configurées absentes des sorties | RG-ANO-01/02, CN-ANO-02 | NOUVEAU | **P1** |
| TEST-ANO-02 | Suppression sur plusieurs tables | colonne retirée en RACINE + boucles + reporting | RG-ANO-04 | NOUVEAU | P2 |
| TEST-ANO-03 | Anonymisation puis chiffrement | colonnes exclues absentes même après déchiffrement | CL-ANO-05 | NOUVEAU | P2 |
| TEST-ANO-04 | Colonne à supprimer inexistante | ignorée/signalée sans échec | CL-ANO-01 | NOUVEAU | P3 |
| TEST-ANO-05 | Chiffrement archive complète | `.zip.enc` illisible brut, déchiffrable via clé Vault | RG-ANO-11/12 | À ADAPTER (existe, 4 scénarios) | — |
| TEST-ANO-06 | Sortie vers MinIO | fichiers écrits dans le bucket (au lieu du disque) | RG-ANO-21 | NOUVEAU | P2 |
| TEST-ANO-07 | Vault indisponible | job en échec propre, pas de sortie en clair | CL-ANO-03 | NOUVEAU | P3 |

### EXE — Exécution (jobs), API, sécurité, erreurs *(quasi aucune couverture Cucumber)*

| ID | Scénario proposé | Vérifie | Règle | Statut | Prio |
|----|------------------|---------|-------|--------|------|
| TEST-EXE-01 | Export asynchrone → 202 + jobId | réponse immédiate avec identifiant de job | RG-EXE-01, CN-EXE-01 | NOUVEAU | **P1** |
| TEST-EXE-02 | Consultation statut RUNNING→DONE + résumé | suivi de job avec nb d'interrogations | RG-EXE-04/05 | NOUVEAU | **P1** |
| TEST-EXE-03 | Job PARTIAL sur erreurs non bloquantes | statut PARTIAL + liste d'erreurs | RG-EXE-03/31, CN-EXE-02 | NOUVEAU | P2 |
| TEST-EXE-04 | Persistance du statut après redémarrage | job antérieur toujours consultable | RG-EXE-02, CL-EXE-02 | NOUVEAU (⚠ intégration) | **P1** |
| TEST-EXE-05 | `jobId` invalide / inconnu | 400 (format) / 404 (inconnu) propres | CL-EXE-01 | NOUVEAU | P2 |
| TEST-EXE-06 | Fichier d'erreurs sur erreur de traitement (non-VTL) | fichier d'erreurs produit avec contenu attendu | RG-CSV-10, RG-EXE-31 | À ADAPTER (do_we_generate_errors) | P2 |
| TEST-EXE-07 | Fichier de log par exécution | log produit par exécution | RG-CSV-11 | À PORTER (do_we_generate_logs) | P3 |
| TEST-EXE-08 | Health-check nominal / Genesis KO / stockage KO | statut global + dégradations sans plantage | RG-EXE-13, CL-EXE-03 | NOUVEAU | P2 |
| TEST-EXE-09 | Autorisation par rôle (USER autorisé, sans rôle 403) | contrôle d'accès OIDC | RG-EXE-21, CN-EXE-03 | NOUVEAU | P2 |
| TEST-EXE-10 | Distinction erreur fatale vs récupérable | erreur fatale → FAILED + arrêt ; récupérable → PARTIAL | RG-EXE-31 | NOUVEAU | P2 |

---

## 3. Synthèse

### Décompte des tests manquants (hors « existants » déjà couverts)

| Domaine | À PORTER | À ADAPTER | NOUVEAU | Total manquant |
|---------|:-------:|:--------:|:------:|:-------------:|
| ACQ | 1 | — | 12 | 13 |
| STR | 1 | 1 | 6 | 8 |
| LIEN | — | — | 3 | 3 |
| MULTI | — | — | 5 | 5 |
| CSV | 4 | — | 6 | 10 |
| JSON | — | — | 9 | 9 |
| REP | — | — | 4 | 4 |
| ANO | — | 1 | 6 | 7 |
| EXE | 1 | 1 | 8 | 10 |
| **Total** | **7** | **4** | **59** | **~69 scénarios** |

### Zones de couverture aujourd'hui totalement absentes (priorité de comblement)

1. **Export JSON** (`JSON`) — aucun test Cucumber n'existe pour `/json`, `/json/replay`, `/debug/json` : 9 scénarios à créer, dont le comportement critique « replay n'avance pas la date d'extraction ».
2. **Exécution asynchrone & suivi de jobs** (`EXE`) — les tests actuels appellent les classes de traitement en direct, en contournant toute la couche REST/async/sécurité : 10 scénarios.
3. **Réconciliation multimode réelle** (`MULTI`) — seule une amorce unitaire existe (`core/do_we_aggregate.feature`, agrégation CAWI+PAPI via `VTLBindings`) ; aucun test n'exerce une vraie partition Genesis à 2+ modes de bout en bout. La logique étant réécrite sans VTL, c'est un point de non-régression prioritaire : 5 scénarios (dont MULTI-01 à adapter depuis l'amorce).
4. **Anonymisation** (`ANO`) — fonctionnalité nouvelle, aucune couverture : 7 scénarios.
5. **Acquisition par `partitionId` + données partielles `STARTED`** (`ACQ`) — paramètre cible et fonctionnalité documentée jamais testés.

### Recommandation de séquencement

- **Vague 1 (P1, ~14 scénarios)** : socle cible non couvert — `partitionId` (ACQ-01/04/05/08), multimode (MULTI-01/02), JSON (JSON-01/02/03/04), anonymisation (ANO-01), jobs (EXE-01/02/04), structure racine+boucles (STR-01/02), format CSV (CSV-01/02).
- **Vague 2 (P2)** : edge cases importants + robustesse (types multimode, échappement CSV, health-check, rôles, MinIO, reporting INDF2F).
- **Vague 3 (P3)** : edge cases de robustesse fine (nulls, non-ASCII, itérations non contiguës, configs par défaut).

### Prérequis technique de test

La reprise impose de **rebâtir les fixtures autour de `GenesisClientStub`** (déjà utilisé par `do_we_export_data_genesis.feature`) plutôt que des dossiers `in/` + `kraftwerk.json`. Le stub Genesis devient le point d'injection de données pour la quasi-totalité des scénarios cibles, paramétré par `partitionId`.

---

## 4. Réserve méthodologique

Inventaire des `.feature` **vérifié ligne à ligne** (révision 2026-07-09, skill `tech-debt-manager` chargée) : les 20 fichiers des 3 modules ont été relus, y compris les 4 features `kraftwerk-core` initialement omises. Le décompte « ~69 scénarios » reste une estimation de dimensionnement, pas un contrat : beaucoup de scénarios sont des *Scenario Outline* couvrant plusieurs cas via `Examples`. Deux amorces existantes (`core/do_we_aggregate`, `core/do_we_export_datasets`) réduisent marginalement le « NOUVEAU » au profit de l'« À ADAPTER ». Les priorités et le séquencement (§3) sont inchangés.
