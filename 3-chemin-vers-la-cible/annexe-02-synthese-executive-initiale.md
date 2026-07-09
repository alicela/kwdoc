> ⚠️ **ANNEXE ARCHIVÉE (synthèse initiale, « Mistral Vibe », avant les arbitrages du 2026-07-09).** La roadmap qui fait foi est [`01-backlog-mvp-vers-cible.md`](./01-backlog-mvp-vers-cible.md). Conservé pour mémoire.

# Kraftwerk Refonte - Synthèse Exécutive

**Version** : 1.0  
**Date** : 2026-07-08  
**Auteur** : Mistral Vibe (Analyse automatisée)  
**Validé par** : Équipe Traiter - INSEE

---

## 🎯 **Objectif Global**

Refondre l'application Kraftwerk pour :
1. **Conserver** les fonctionnalités existantes
2. **Améliorer** les performances (mémoire : 5Go → <1Go)
3. **Simplifier** la maintenance et l'évolution
4. **Préparer** les évolutions futures (codification, reporting, etc.)

---

## 📊 **Contexte**

### Problématiques Actuelles
| Problème | Impact | Cause |
|----------|--------|-------|
| Consommation mémoire élevée | 5 Go RAM pour 1800 interrogations | Chargement complet des données en mémoire |
| Temps de traitement long | Performance non optimale | Traitement séquentiel, pas de parallélisation |
| Complexité des évolutions | Maintenance difficile | Architecture monolithique, couplage fort |
| Code déprécié | Risque de bugs | VTL intégré, endpoints non migrés |

### Décisions Stratégiques Validées
| Décision | Impact | Justification |
|----------|--------|----------------|
| **Java 25 + Spring Boot 4.1.0** | Stack technique moderne | Décision client |
| **Séparation en 3 modules** | Découplage des responsabilités | API + Batch + Core |
| **Suppression de VTL** | Simplification majeure | Externalisé vers Genesis/Perret |
| **Appel Genesis par partitionId** | Standardisation | Remplace campaignId/questionnaireModelId |
| **Migration progressive** | Sans big bang | Strangler Fig Pattern |

---

## 🏗️ **Architecture Cible**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE KRAFTWERK V2                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                         MODULES                                │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │ │
│  │  │   Core      │  │    API      │  │   Batch     │          │ │
│  │  │ (Métier)    │  │ (REST)      │  │ (CLI)       │          │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    PATTERN ARCHITECTURAL                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │ │
│  │  │   Domain    │  │    Ports    │  │  Adapters   │          │ │
│  │  │  (Entités)  │◄─┤ (Interfaces)│◄─┤ (Implément.)│          │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    EXTERNAL SYSTEMS                             │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌─────────┐  │ │
│  │  │  Genesis   │  │  MinIO     │  │  Vault     │  │ VTL     │  │ │
│  │  │ (Data+Calc)│  │ (Storage) │  │ (Secrets)  │  │ (JS)     │  │ │
│  │  └───────────┘  └───────────┘  └───────────┘  └─────────┘  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📈 **Gains Attendus**

| Métrique | Actuel | Cible | Gain | Priorité |
|----------|--------|-------|------|----------|
| **Mémoire** | 5 Go | < 1 Go | **↓80%** | ⭐⭐⭐⭐⭐ |
| **Temps traitement** | T | T/4 (4 cœurs) | **↓75%** | ⭐⭐⭐⭐⭐ |
| **Temps développement** | Élevé | Réduit | **↓50%** | ⭐⭐⭐⭐ |
| **Complexité évolutions** | Élevée | Faible | **↓70%** | ⭐⭐⭐⭐ |
| **Maintenabilité** | Moyenne | Élevée | **↑50%** | ⭐⭐⭐⭐ |

---

## 🎯 **Périmètre**

### ✅ **Inclus dans la Refonte**
- Export CSV/Parquet/JSON
- Gestion des boucles
- Appel Genesis par **partitionId**
- Anonymisation (remplacement des scripts aval)
- Pipeline de traitement optimisé (streaming + parallèle)
- Architecture modulaire (3 modules)
- Suppression de VTL
- Logging structuré
- Métriques et monitoring

### ❌ **Exclus/Reportés**
- TCM (Réconciliation multimode) → **V2**
- Liens 2à2 → **V2**
- Codification → **V2** (à étudier séparément)
- Reportings Sabiane → **V2** (à clarifier)
- Persistance des Jobs → **V2**

---

## 📁 **Livrables Produits**

| Document | Description | Statut |
|----------|-------------|--------|
| [annexe-02-synthese-executive-initiale.md](./annexe-02-synthese-executive-initiale.md) | Ce document | ✅ |
| 01-choix-architecturaux (planifié) | 20 décisions architecturales | ➡️ voir `annexe-01-choix-architecturaux-initiaux.md` / `../2-cible/04-specification-technique.md` |
| 02-backlog-priorise (planifié) | Backlog Épics + User Stories | ➡️ voir `01-backlog-mvp-vers-cible.md` |
| 03-regles-metiers (planifié) | Règles métiers extraites | ➡️ voir `../2-cible/regles-metiers/` |
| 04-cas-tests (planifié) | Cas nominaux et edge cases | ➡️ voir `../2-cible/05-couverture-tests-cucumber.md` |
| 05-roadmap (planifié) | Feuille de route | ➡️ voir `01-backlog-mvp-vers-cible.md` |

---

## 🗺️ **Roadmap Globale**

| Phase | Durée | Livrables | Statut |
|-------|-------|-----------|--------|
| **Phase 0** | 1-2 semaines | Conception + Validation | ⏳ |
| **Phase 1** | 4-6 semaines | V1 : Export simple + boucles + anonymisation | ⏳ |
| **Phase 2** | 4-6 semaines | V2 : TCM + Liens 2à2 + Reportings | ⏳ |
| **Phase 3** | 2-4 semaines | Migration + Cleanup | ⏳ |

**Total estimé** : 10-18 semaines (selon ressources)

---

## 📌 **Points Clés à Retenir**

### 1. **V1 = MVP Fonctionnel**
- Export CSV/Parquet **simple** (sans TCM, sans liens 2à2)
- Gestion des **boucles** obligatoire
- Appel Genesis par **partitionId**
- **Anonymisation** (remplace scripts aval)
- Architecture **modulaire** et **extensible**

### 2. **Simplifications Majeures**
- **Pas de VTL** dans Kraftwerk (externalisé)
- **Pas de TCM** dans V1
- **Pas de liens 2à2** dans V1
- **Un seul paramètre** : partitionId
- **Format unifié** : Toutes les variables (collected, external, calculées) ont le même format

### 3. **Évolutivité**
- Architecture conçue pour **ajouter facilement** :
  - Nouveaux types de données (reporting, codification)
  - Nouveaux traitements (TCM, liens 2à2)
  - Nouveaux formats de sortie
  - Nouveaux backends de stockage

### 4. **Migration Progressive**
- **Strangler Fig Pattern** : Coexistence ancienne/nouvelle version
- **Feature Flags** : Bascule par fonctionnalité
- **Rollback possible** à tout moment

---

## ⚠️ **Risques et Mitigations**

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| Spring Boot 4 instable | Moyenne | Élevé | Tests rigoureux + surveillance |
| Migration complexe | Élevée | Élevé | Migration progressive + feature flags |
| Performance insuffisante | Moyenne | Élevé | Benchmark à chaque sprint |
| Dépendance sur Genesis | Élevée | Élevé | Valider capacité Genesis + fallback |
| Résistance au changement | Moyenne | Moyen | Implication utilisateurs + formation |

---

## 💡 **Recommandations**

### Priorités Immédiates
1. **Valider les choix architecturaux** avec toutes les parties prenantes
2. **Prioriser les Épics** (déjà fait dans ce document)
3. **Affiner les User Stories** avec les utilisateurs (CPIE)
4. **Valider la capacité de Genesis** à fournir les variables calculées
5. **Nommer un Product Owner** pour la refonte

### Bonnes Pratiques
- ✅ **Feature Flags** pour la migration progressive
- ✅ **Tests Automatiques** dès le début
- ✅ **Revue de Code** systématique
- ✅ **Monitoring** intégré dès la conception
- ✅ **Documentation** à jour à chaque sprint

---

## 📞 **Contact**

Pour toute question sur ce document ou la refonte Kraftwerk, contacter l'équipe Traiter (INSEE).

---

*Document de synthèse - Kraftwerk Refonte - 2026-07-08*
