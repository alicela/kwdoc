# Estimation Macro — Kraftwerk v2 (V0→V5)

**Date** : 2026-07-21  |  **Objectif** : Estimation des versions, avec focus sur V0 et V1  |  **Statut** : Fiche de travail

---

## 📌 Contexte & Hypothèses

- **Stack technique** : Java 25 + Spring Boot 4.1 + Maven
- **Équipe** : A priori : 3 devs backend + 1 lead-tech
- **Objectif V1 (MVP)** : Démontrer la valeur avec **export JSON incrémental + replay** depuis Genesis (SI externe)
- **Contrainte performance** : **< 1 Go RAM** pour 1800 interrogations
- **Prérequis bloquant** : Endpoint Genesis `/partitions/{partitionId}` (US-0.9, à faire avec ou sans refonte)

---

---

## 🎯 Sommaire par Version

| Version | Épics | US | Objectif | Effort total (indicatif) | Livrable clé |
|---------|-------|----|----------|--------------------------|--------------|
| **V0** | 1 (EPIC-0) | 9 | **Fondations** (socle technique + garde-fous) | **~8M** | Squelette hexagonal + CI + portes qualité |
| **V1** | 1 (EPIC-7) | 5 | **MVP : Export JSON SI externe** | **~3L** | Export JSON incrémental + replay, async, démo client |
| **V2** | 5 (EPIC-1,2,4,5,11) | ~15 | **Export tabulaire complet** | **~10M-12M** | Parquet/CSV + boucles + états + liens 2à2 + suivi jobs |
| **V3** | 3 (EPIC-3,6,8) | ~10 | **Service robuste** | **~8M** | Multimode SQL + sécurité/API + reporting |
| **V4** | 3 (EPIC-9,10,12) | ~8 | **Parité cible** | **~6M** | Anonymisation + chiffrement/MinIO + observabilité |
| **V5** | 1 (EPIC-13) | 5 | **Évolutions** | **~5L** | Lot, multimode sans réconciliation, codification, date T |

---


---

## 🔗 Travaux Complémentaires sur le SI

*Tâches externes à Kraftwerk mais nécessaires pour la solution complète. **Toutes gérées par la même équipe.***

| Composant | US/Tâche | Effort estimé | Dépendances | Priorité |
|-----------|----------|----------------|-------------|----------|
| **BPM** | Valider la présence et la structure des **liens 2à2** (`LIEN_1` à `LIEN_20`) | **S** | — | ⭐⭐⭐ |
| **Node (Perret)** | Développer **module complémentaire Node** pour calculer les variables `calculated` | **M-L** | Spécifications variables | ⭐⭐⭐ |
| **Genesis** | **Intégrer le module Node** dans le process de calcul des variables | **M** | Module Node prêt | ⭐⭐⭐ |
| **Perret** | **Envoyer les `calculated` à la sauvegarde** (front + back) | **M** | Module Node intégré | ⭐⭐⭐ |
| **Genesis** | **Ajouter endpoint `/partitions/{partitionId}`** (NDJSON, streaming, pagination, OIDC) | **L** | — | ⭐⭐⭐⭐ |

> ⚠️ **Impact sur Kraftwerk** : Sans ces travaux, **US-0.4** (Client Genesis) et **US-0.9** (endpoint) ne peuvent pas être implémentés correctement.

---

## 🔍 Détail par Version

---

### ➡️ V0 — Fondations (EPIC-0)

**Objectif** : Poser le socle technique **avant toute fonctionnalité**.

*« Les portes qualité posées après coup coûtent 10× » (doc 09).*

| US | Titre | Effort | Complexité | Dépendances | Risque | Notes |
|----|-------|--------|------------|-------------|--------|-------|
| US-0.1 | Squelette Maven **modules séparés** (`core`/`batch`/`api`/`encryption`) + **POM parent unique** | **M** | Architectural | — | ⭐⭐⭐ | Découpage modulaire critique |
| US-0.2 | Structure **Ports & Adapters** + config Spring Boot 4.1 / Java 25 | **M** | Technique | US-0.1 | ⭐⭐ | Pattern hexagonal |
| US-0.3 | Pipeline **CI + portes qualité bloquantes** : couverture, ArchUnit « zéro VTL », **SBOM CycloneDX**, NullAway, PIT, build unique | **M** | DevOps | US-0.1 | ⭐⭐⭐ | Nouveaux outils à configurer |
| US-0.4 | **Client Genesis déclaratif** (`@HttpExchange`/`RestClient`) + **Resilience4j** + auth service-à-service | **L** | Intégration | — | ⭐⭐⭐ | Coordination avec équipe Genesis |
| US-0.5 | **Adapter métadonnées BPM** (`MetadataPort`, DDI + Lunatic) | **M** | Technique | US-0.1 | ⭐⭐ | Intégration lib BPM |
| US-0.6 | **Moteur DuckDB** (ingestion via appender, base éphémère par job, écriture) | **M** | Technique | US-0.1 | ⭐⭐⭐ | Nouveau moteur, POC nécessaire |
| US-0.7 | **Docker multi-stage** rootless + `HEALTHCHECK` + **CDS/AOT cache** + config typée externalisée | **S** | DevOps | US-0.1 | ⭐⭐ | Optimisation démarrage |
| US-0.8 | **Socle observabilité & erreurs** : Micrometer Observation + OTLP + ScopedValue + `@RestControllerAdvice`→`ProblemDetail` | **M** | Technique | US-0.1 | ⭐⭐⭐ | Nouveaux standards Java 25 |
| **US-0.9** | **Endpoint Genesis par `partitionId`** (côté Genesis) : NDJSON streaming + pagination + OIDC | **L** | **Externe (Genesis)** | — | ⭐⭐⭐⭐ | **Blocage si retard** |

**📊 Synthèse V0**
- **Effort total** : **~8M** (9 US)
- **Durée estimée** : **1 semaine** (équipe de 3 devs)
- **Risques majeurs** : US-0.4 (coordination Genesis), US-0.3/US-0.8 (nouveaux outils)
- **Livrable** : Socle technique validé, CI verte, stubs opérationnels

---

### ➡️ V1 — MVP : Export JSON SI externe (EPIC-7)

**Objectif** : **Démontrer la valeur** avec un export JSON fonctionnel (incrémental + replay).

*Chemin critique : EPIC-0 → EPIC-7.*

| US | Titre | Effort | Complexité | Dépendances | Risque | Notes |
|----|-------|--------|------------|-------------|--------|-------|
| US-7.1 | Export JSON **structure complète** (bloc identification + `data` RACINE + boucles), `partitionId` renseigné | **L** | Métier | US-0.9, US-0.4 | ⭐⭐⭐ | Pivot data + intégration Genesis |
| US-7.2 | **Incrémental** par `sinceDate` / dernière extraction ; MAJ date **au succès seulement** | **M** | Technique | US-7.1 | ⭐⭐ | Gestion état |
| US-7.3 | **Replay** `[sinceDate, untilDate]` sans faire avancer la date d'extraction | **M** | Technique | US-7.1 | ⭐⭐ | Requête historique |
| US-7.4 | UTC/local exclusifs ; sortie vide/204 si rien de neuf | **S** | Technique | US-7.1 | ⭐ | Formatage |
| US-7.5 | **Debug par `interrogationIds`** (échecs isolés) | **M** | Technique | US-7.1 | ⭐ | Diagnostic |

**📊 Synthèse V1**
- **Effort total** : **~3L** (5 US)
- **Durée estimée** : **3 sprints** (après V0)
- **Risques majeurs** : US-7.1 (pivot data + intégration Genesis), dépendance forte sur US-0.9
- **Livrable** : **Démo client** avec export JSON incrémental + replay d'une partition réelle
- **Jalon** : **M1 — MVP** atteint

---

---

## 📋 Tableau Synthétique pour Estimation

| Version | Périmètre | US | Effort | Durée (3 devs) | Dépendances critiques | Livrable |
|---------|-----------|----|--------|-----------------|------------------------|-----------|
| **V0** | Fondations | 9 | ~8M | 1 semaine | — | Socle + CI + stubs |
| **V1** | MVP JSON | 5 | ~3L | 3 sprints | US-0.9 (Genesis) | Démo client |
| **V0+V1** | **MVP complet** | **14** | **~11M-1L** | **1 série** | Endpoint Genesis | **M1 atteint** |
| **V2** | Export tabulaire | ~15 | ~10M-12M | 1 sprint | V1 | Parquet/CSV + boucles |
| **V3** | Service robuste | ~10 | ~8M | 1-2 sprints | V2 | Multimode + API |
| **V4** | Parité cible | ~8 | ~6M | 1-2 sprints | V2 | Chiffrement + obs |
| **V5** | Évolutions | 5 | ~5L | ? | V4 | Fonctionnalités avancées |

---

---


---

## 📊 Estimations alternatives (en points)

*Estimations fournies en points (et non en jours-homme) — Objectif : rendre compte de l'estimation pour décider de sa pertinence.*

> **💡 Point clé** : En 1 série, on peut intégrer les enquêtes de conjoncture (V0+V1) sans régression sur l'actuel, et en calculant les agrégats VTL. En **2 séries de sprints** (V0 à V4), on obtient une version qui **reprend toutes les fonctions utilisées** avec un **coût de maintenance bien inférieur** à la version actuelle.

### Estimation détaillée

- **V0** : 13 points — soit environ 1 dev sur une grosse semaine (fondations avec *process* des variables calculées et stockage par `interrogationId`, parallélisable avec *endpoint Genesis* sur une itération)
- **V1** : 3 sprints
- **V0+V1** : 1 série
- **Mise à jour kwactuel** : 1 sprint (beaucoup de runs, duplication/factorisation) + endpoint (13 points) — *mais on ne calcule pas les sommes et on maintient un système tordu*
- **Brique VTL-Lunatic en Node** : 5 points
- **Mise à jour Perret** : 8 points (3 front + 5 back)




---

---

---

## 📚 Références

- [Backlog complet (V0→V5)](./01-backlog-mvp-vers-cible.md)
- [Spécification fonctionnelle](../2-cible/03-specification-fonctionnelle.md)
- [Spécification technique](../2-cible/04-specification-technique.md)
- [Décisions client](../2-cible/02-decisions-client-arbitrages.md)

---

*Document généré le 2026-07-21 — Mis à jour avec les travaux SI complémentaires — À adapter selon ton contexte équipe et technique.*