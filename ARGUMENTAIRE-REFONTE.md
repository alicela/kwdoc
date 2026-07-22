# Kraftwerk v2 — Argumentaire de la refonte (note de synthèse)

**Date** : 2026-07-22  |  **Public** : décideurs / arbitrage  |  **Objectif** : justifier la refonte par les **écarts** (macro) et les **estimations** de coût

> Note de décision **chiffrée** : que gagne-t-on, à quel coût, avec quels risques ? Pour le pitch d'équipe (1 page) voir [`PRESENTATION.md`](./PRESENTATION.md) ; pour le détail des écarts voir [`2-cible/ecarts-existant-cible.md`](./2-cible/ecarts-existant-cible.md).

---

## 1. En bref

Kraftwerk remplit sa mission mais coûte **de plus en plus cher à faire tourner et à faire évoluer**, pour des raisons **structurelles** que des correctifs de version ne résorbent pas :

- **~5 Go de RAM** pour 1800 interrogations (objectif cible **< 1 Go**).
- **Health Score dette : 44/100**, tendance ↘ — trois foyers : VTL/Trevas diffusé partout, couche API dupliquée, tests arrimés au disque local.
- **VTL/Trevas** : type `VtlBindings` référencé **99× dans 31 fichiers** — le plus fort couplage du dépôt, verrou de toute évolution.
- Le **contexte a changé** : **Genesis** structure désormais la donnée en amont (variables déjà calculées) → une grande part du code de Kraftwerk fait **doublon**.

**Recommandation** : refonte **incrémentale** (sans big bang, rollback possible), démarrable **immédiatement** par les fondations (EPIC-0), MVP à valeur démontrable en **1 série** (V0+V1).

---

## 2. Le problème aujourd'hui (constat chiffré)

| Domaine | Constat mesuré | Source |
|---|---|---|
| **Performance** | ~**5 Go de RAM** pour 1800 interrogations ; traitement **séquentiel**, tout chargé en mémoire | [objectifs](./2-cible/01-objectifs-refonte.md) |
| **Dette globale** | **Health Score 44/100** (tendance ↘, dette structurelle non résorbée par les versions) | [audit dette §1-2](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Couplage VTL** | `VtlBindings` **99 réf. / 31 fichiers**, 11 imports `fr.insee.vtl.*`, 2 dépendances Trevas | [audit §5.1](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Architecture** | Monolithe **API + Batch** dans le même artefact ; `MainProcessing` fortement couplé ; ~**27 000 lignes** Java | [audit §5.2](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Suivi des traitements** | Suivi de jobs **partiellement non fonctionnel** (route jamais exposée, 404 possible), 2 stores non persistants | [audit §5.2](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Tests** | Aucun **seuil de couverture** ; tests fonctionnels **couplés au disque** (~82 Mo de fixtures, 15 features à reconstruire) | [audit §5.3](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Build / CI** | Deux POM racine divergents (**SB 4.1.0 vs 4.0.6**) ; **chiffrement jamais buildé/testé en CI** ; pas de scan de vulnérabilités | [audit §5.4-5.5](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Config** | Valeurs **en dur** dans les *resources* de prod (chemins Windows, URL Genesis prod en clair) | [audit §5.5](./1-existant/06-dette-technique-et-garde-fous.md) |
| **Coût équipe** | Le **run** de Kraftwerk consomme une part très importante du temps de l'équipe Traiter ; évolutions **de plus en plus complexes** | [objectifs](./2-cible/01-objectifs-refonte.md) |

> 💡 La dette n'est **pas** dans la stack (Java 25 + Spring Boot 4, récente et saine, choix client assumé) mais dans l'**architecture** et le **couplage VTL** — ce que seule une réécriture ciblée corrige.

---

## 3. Ce que la refonte change (écarts macro)

Vue macro ; détail dans [`2-cible/ecarts-existant-cible.md`](./2-cible/ecarts-existant-cible.md).

| ⛔ Disparaît | ♻️ Externalisé | ✨ Nouveau / amélioré | ✅ Ne change pas |
|---|---|---|---|
| Moteur **VTL/Trevas** | Calcul des **variables** → Genesis | **partitionId** (unité d'appel/export) | Mission : **export CSV/Parquet/JSON** |
| **Parsing local** (XML/Coltrane, `XformsDataParser`) | **Liens 2à2** → BPM | **Post-script SQL/R** sur tables DuckDB | Gestion des **boucles** |
| **Paradonnées** | **Modules TCM** → BPM | **Anonymisation** formalisée | **Reporting** de collecte |
| Scripts d'import **SAS** | | **Chiffrement bout-en-bout** + interchangeable | Stack **Java 25 / Spring Boot 4** |
| **Archivage** des entrées | | **Suivi de job « au fil de l'eau »** unifié | |
| Endpoints **legacy** (`/main`, `/steps`, `/split`) | | **Gestion d'erreurs centralisée** ; **séparation API/Batch** | |

---

## 4. Bénéfices attendus (quantifiés)

| Axe | Existant (base v4.1.3) | Cible | Levier |
|---|---|---|---|
| **Mémoire** | ~5 Go / 1800 interrogations | **< 1 Go** | streaming + virtual threads + DuckDB ensembliste |
| **Temps de traitement** | séquentiel | **réduction visée ~50 %** (NF-002) | parallélisation, moins de calculs (zéro VTL) |
| **Couplage VTL** | 11 imports / **99 réf.** `VtlBindings` | **0 / 0** | réécriture SQL DuckDB + porte qualité ArchUnit « zéro VTL » |
| **Baselines Spring Boot** | 2 (4.1.0 / 4.0.6) | **1** | POM parent unique |
| **Modules non testés en CI** | 2 (chiffrement) | **0** | un seul chemin de build, chiffrement testé |
| **Scan de vulnérabilités** | absent | **présent (bloquant)** + SBOM CycloneDX | supply-chain en CI |
| **Config env. en dur** | ≥ 2 (chemin, URL) | **0** | externalisation (properties + Vault) |
| **Maintenabilité / évolutivité** | monolithe couplé | **hexagonal**, API/Batch séparés, modèle métadonnées unique | ports & adapters |

> Au-delà des chiffres : **évolutions plus faciles** (points d'extension : post-script, transformation multimode), **traitements pilotables** par la MOA (suivi live + erreurs centralisées), **meilleure protection des données** (anonymisation en amont, chiffrement tout au long).

---

## 5. Estimation macro (coût)

Estimations **indicatives** (tailles S/M/L), équipe **3 devs + 1 lead-tech**. Détail : [`3-chemin-vers-la-cible/estimation-versions.md`](./3-chemin-vers-la-cible/estimation-versions.md).

| Version | Périmètre | US | Effort | Durée (3 devs) | Livrable clé |
|---|---|---|---|---|---|
| **V0** | Fondations (socle + garde-fous) | 9 | ~8M | ~1 semaine | Squelette hexagonal + CI + portes qualité |
| **V1** | **MVP** : export JSON incrémental + replay | 5 | ~3L | 3 sprints | Démo client — **jalon M1** |
| **V0+V1** | **MVP complet** | **14** | **~11M–1L** | **1 série** | Endpoint Genesis requis |
| **V2** | Export tabulaire (Parquet/CSV, boucles, états, liens, jobs) | ~15 | ~10M–12M | 1 sprint | Parité tabulaire |
| **V3** | Service robuste (multimode SQL, API/sécurité, reporting) | ~10 | ~8M | 1–2 sprints | Robustesse |
| **V4** | Parité cible (anonymisation, chiffrement/MinIO, observabilité) | ~8 | ~6M | 1–2 sprints | Parité fonctionnelle |
| **V5** | Évolutions (lot, multimode sans réconciliation, codification, date T) | 5 | ~5L | — | Nouvelles capacités |

**Points de repère (en points)** : V0 ≈ **13 points** (~1 dev sur une grosse semaine) ; V0+V1 = **1 série** → intègre les enquêtes de conjoncture **sans régression** ; **V0→V4 en 2 séries** → reprend **toutes les fonctions utilisées** avec un **coût de maintenance bien inférieur**.

**Travaux SI complémentaires** (même équipe, mais **hors Kraftwerk** — prérequis à cadrer) :

| Composant | Tâche | Effort | Priorité |
|---|---|---|---|
| **Genesis** | Endpoint `/partitions/{partitionId}` (NDJSON, streaming, pagination, OIDC) | L | ⭐⭐⭐⭐ **bloquant** |
| **Node (Perret)** | Module de calcul des variables `calculated` (brique VTL-Lunatic) | M–L / 5 pts | ⭐⭐⭐ |
| **Genesis / Perret** | Intégration du module + envoi des `calculated` à la sauvegarde | M / 8 pts | ⭐⭐⭐ |
| **BPM** | Valider présence/structure des liens `LIEN_1…LIEN_20` | S | ⭐⭐⭐ |

> 🔎 **Alternative « ne rien refondre »** : maintenir l'actuel = ~1 sprint (runs, duplication/factorisation) **+ endpoint (13 pts)** — mais **on ne calcule pas les sommes** et **on maintient un système tordu** dont la dette continue de croître.

---

## 6. Risques et conditions de succès

| Risque / condition | Traitement |
|---|---|
| **Prérequis bloquant** : endpoint Genesis `/partitions/{partitionId}` (US-0.9) | À lancer **en parallèle** de V0 ; sans lui, client Genesis non implémentable |
| **Dépendance accrue à Genesis / Perret** (variables calculées externalisées) | Valider tôt que Genesis stocke les `calculated` et que Perret les renvoie |
| **Reconstitution de la dette** dans la cible | **Portes qualité dès le 1ᵉʳ commit** (couverture bloquante, ArchUnit « zéro VTL », scan sécu) — *les poser après coup coûte 10×* |
| **Risque de bascule** | **Migration progressive** par fonctionnalité, coexistence ancien/nouveau, **rollback possible** (NF-003) |
| **Stack récente** (Java 25 / SB 4) | Choix client assumé ; tester rigoureusement, surveiller les versions |

---

## 7. Décision demandée

1. **Feu vert pour EPIC-0 (fondations)** — démarrable immédiatement, pose le socle et les garde-fous.
2. **Cadrage en parallèle des 2 points durs** : endpoint Genesis (US-0.9) et spike du pivot data (US-1.0).
3. **Arbitrage des points ouverts** listés dans [`PRESENTATION.md`](./PRESENTATION.md) (utilité de l'API, pivot, anonymisation, mode sans-DDI, post-script…).

---

## 📚 Références

- [Écarts existant → cible (détail)](./2-cible/ecarts-existant-cible.md)
- [Audit de dette technique & garde-fous](./1-existant/06-dette-technique-et-garde-fous.md)
- [Objectifs de la refonte (constat client)](./2-cible/01-objectifs-refonte.md)
- [Estimation macro V0→V5](./3-chemin-vers-la-cible/estimation-versions.md) · [Backlog MVP → cible](./3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md)
- [Présentation équipe (1 page)](./PRESENTATION.md)

---

*Note de synthèse — 2026-07-22. Chiffres d'effort indicatifs, à adapter au contexte équipe.*
