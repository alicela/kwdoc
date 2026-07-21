# Kraftwerk v2 (KW2) — Impacts sur le SI

**Date** : 2026-07-20  |  **Version** : 1.0

Ce document recense les **évolutions nécessaires sur les systèmes amont** pour permettre le déploiement de Kraftwerk v2.

---

## 📋 **Impacts par système**

| Système | Impact |
|---------|--------|
| **BPM** | Gérer les **liens 2 à 2** : produire les variables `LIEN_1` à `LIEN_20` avec sentinelles `0`/`99`. BPM garantit la cohérence des données (ex: ménage 1/N individus). Kraftwerk v2 consomme uniquement ces variables sans logique métier. |
| **Genesis** | **Endpoint `/partitions/{partitionId}`** : retourner les interrogations d'une partition en cumulant les modes, paginé (documents = interro×mode complètes). Format : NDJSON (streaming + compression). Champs : `interrogationId`, `usualSurveyUnitId`, `collectionInstrument`, `mode`, `questionnaireState`, `validationDate`, `isCapturedIndirectly`, variables (`varId`/`value`/`scope`/`iteration`), `shortLabel` de partition. Sécurité : OIDC (JWT). **Prérequis bloquant pour le MVP.** |
| **Genesis** | **Module VTL JS** : intégrer dans le process pour permettre à Genesis de précalculer les variables actuellement traitées par VTL/Trevas dans KW1. Objectif : alignement avec la cible KW2 (zéro VTL). |
| **Perret** | **Enregistrement des `calculated` dans Genesis** : Nécessaire dès la **V2** pour que KW2 puisse consommer les variables pré-calculées. |

---

## 📖 **Détail des impacts**

### BPM — Liens 2 à 2
- Kraftwerk v2 **consomme uniquement** les variables de liens produites par BPM.
- **Format attendu** : `LIEN_1` à `LIEN_20` avec sentinelles `0` (début) et `99` (fin).
- **Validation** : BPM garantit la cohérence (ex: ménage 1/N individus).
- **Contrainte** : Toute évolution (borne > 20, nouvelles enquêtes) sera portée **uniquement dans BPM**.
- **Intégration KW2** : Simple consommation via `MetadataPort` (aucune dépendance complexe).

### Genesis — Endpoint `partitionId`
- **Contrat API** :
  - **URL** : `GET /api/partitions/{partitionId}`
  - **Format** : NDJSON (streaming + compression + pagination)
  - **Structure** : Chaque document = une interro×mode **complète** (toutes les variables d'une interro×mode dans le même document, jamais scindée).
  - **Champs obligatoires** : `interrogationId`, `usualSurveyUnitId`, `collectionInstrument`, `mode`, `questionnaireState`, `validationDate`, `isCapturedIndirectly`, variables (`varId`, `value`, `scope`, `iteration`), `shortLabel` de partition.
  - **Sécurité** : OIDC (JWT).
  - **OpenAPI** : Spécification à publier.
- **Blocage** : **Prérequis dur** pour le MVP (EPIC-7). Sans cet endpoint, KW2 ne peut pas démarrer.
- **Workaround dev** : Développement contre un **stub** (`GenesisClientStub`) en attendant la livraison.

### Genesis — Module VTL JS
- **Objectif** : Permettre à Genesis de **précalculer les variables** (actuellement calculées par VTL/Trevas dans KW1) pour alignement avec la cible KW2 (zéro VTL).
- **Portée** : Les variables déjà calculées par Genesis doivent couvrir 100% des besoins KW1 pour éviter les régressions.
- **Synchro** : À coordonner avec l'équipe Genesis pour éviter les doublons avec le travail KW2.

### Perret — Enregistrement des `calculated`
- **Objectif** : Enregistrer les variables de type **`calculated`** dans Genesis pour qu'elles soient disponibles via l'endpoint `partitionId`.
- **Nécessité** : **Dès la V2** (KW2 ne recalcule pas les variables, il les consomme telles quelles).
- **Impact** : Sans cet enregistrement, KW2 ne pourra pas fonctionner correctement (variables manquantes).

---

## ⚠️ **Risques**

| Risque | Impact | Mitigation |
|--------|--------|------------|
| Retard sur l'endpoint `partitionId` (Genesis) | **Blocage MVP** (EPIC-7 impossible) | Suivi régulier avec Genesis, développement contre stub |
| Liens 2à2 non disponibles (BPM) | EPIC-11 reporté, **M2 incomplet** | Valider planning BPM dès maintenant |
| Module VTL JS non prêt | KW2 ne peut remplacer KW1 | Maintenir KW1 en parallèle temporairement |

---

## ✅ **Prochaines étapes**
- [ ] Valider avec BPM : planning pour les variables `LIEN_*`
- [ ] Confirmer avec Genesis : priorité endpoint `partitionId` (US-0.9)
- [ ] Clarifier besoin exact du module VTL JS (périmètre variables, timing)
- [ ] Créer les tickets correspondants dans les backlogs BPM/Genesis
