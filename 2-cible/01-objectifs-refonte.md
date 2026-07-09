# Kraftwerk - Objectifs de la Refonte

## Date : 2026-07-08
**Source** : Spécifications client (équipe Traiter - INSEE)

---

## Contexte et Constat

### Problématiques Actuelles

| Problème | Impact | Mesure |
|----------|--------|---------|
| **Temps de développement** | Run Kraftwerk consomme une part très importante du temps de l'équipe Traiter | Métier + Dev |
| **Complexité des évolutions** | Les évolutions deviennent de plus en plus complexes | Risque de blocage |
| **Codification** | Évolution lourde pour un résultat incertain | Compromis |
| **Ressources infrastructure** | **Plus de 5 Go de RAM** pour les exports CSV BDFWEB | Performance |
| **Échantillon** | 1800 interrogations dans Genesis → 5Go RAM | Non optimisé |
| **Architecture obsolète** | Développé quand la filière était peu structurée | Maintenance difficile |

### Analyse des Causes

1. **Architecture monolithique** : API + Batch dans la même application
2. **Traitement séquentiel** : Pas de parallélisation des fichiers
3. **Chargement en mémoire** : Toutes les données chargées avant traitement
4. **VTL intégré** : Calculs lourde dans Kraftwerk
5. **DuckDB** : Base de données in-memory non optimisée pour les gros volumes
6. **Manque de modularité** : Difficile d'ajouter de nouvelles fonctionnalités

---

## Objectifs de la Refonte

### 1. Fonctionnalités (Stabilité)
✅ **Offrir les mêmes fonctionnalités que celles actuelles** en les améliorant

**Fonctionnalités Actuelles à Conserver** :
- [x] Exporter des données au format **CSV/Parquet/JSON** de manière ponctuelle
- [x] Gestion des **boucles** et des **liens 2 à 2**
- [x] Avec ou non **état de la dernière valeur** de la variable
- [x] **Indicatrice de filtre** (FILTER_RESULT)
- [x] **Indicatrice de non-réponse** (MISSING)
- [x] Exporter des données de manière **automatisée**
- [x] **Calculer les variables calculées** (mais avec bug sur les sommes)
- [x] **Scripts "aval"** pour les exports (à re-questionner)
- [x] **Logs efficaces** et **alerting**
- [x] **Exporter les reportings Sabiane** (à re-questionner)

### 2. Évolutivité (Amélioration)
✅ **Permettre les évolutions souhaitées de manière plus facile**

**Évolutions Fonctionnelles Envisagées** :
- [ ] **Export par lot** : Le lot permettra de rechercher le ou les `collectionInstrumentId` correspondant
- [ ] **Export multimode sans réconciliation** : Les `interrogationId` ayant plusieurs réponses (une par mode) apparaissent tous, avec leurs différents modes
- [ ] **Codification** : Envoyer en codification les données, intégrer les retours de codification
- [ ] **Types de variable** : Revoir les types de variable sorties par les exports (implique probablement des modifications Genesis)
  - Reportings datas web utiles pour qualifier les variables collectées
  - Confirmation de contrôles de reprise
- [ ] **Exports à date T** : Exporter l'état contrôles de reprise à une date T en se basant sur les valeurs et les états des variables collectées à cette date

### 3. Performance (Optimisation)
✅ **Alléger l'architecture pour utiliser moins de ressources**

**Objectifs Concrets** :
- Réduire la consommation mémoire de **5 Go → < 1 Go** pour 1800 interrogations
- Optimiser le temps de traitement
- Permettre le traitement de plus gros volumes

### 4. Migration (Progressivité)
✅ **Permettre une bascule sans big bang, au fur et à mesure des avancées de la refonte**

**Stratégie** :
- Migration incrémentale par fonctionnalité
- Coexistence des anciennes et nouvelles versions pendant la transition
- Rollback possible à tout moment

---

## Décisions Techniques (Client)

### ✅ Stack Technique Validée

| Composant | Version | Justification |
|-----------|---------|---------------|
| **Java** | **JDK 25** | Décision client - On garde JDK 25 |
| **Spring Boot** | **4.1.0** | Décision client - On garde Spring Boot 4 |
| **Build** | Maven | Standard INSEE |

**Note** : Le client a validé explicitement Java 25 + Spring Boot 4, malgré les recommandations initiales de passer sur des versions LTS.

---

## Décisions Architecturales (Client)

### ✅ Suppression de VTL dans Kraftwerk

**Décision** : **Supprimer complètement le module VTL de Kraftwerk**

#### Remplacement pour les Variables Calculées
```
Actuellement :
Enquête → Collecte → Genesis → Kraftwerk (VTL) → Export

Cible :
Enquête → Collecte → Genesis → Module VTL Externe (JS) → Genesis (stockage) → Kraftwerk → Export
```

**Mécanisme** :
1. Quand une donnée arrive dans Genesis, **passer par un module VTL externe** (JavaScript, identique aux plateformes)
2. **Calculer les sommes** et autres variables calculées
3. **Stocker le résultat des calculs** dans Genesis
4. Si Perret modifie une donnée, **Perret renvoie les variables calculées** à Genesis
5. **Kraftwerk lit directement les résultats** depuis Genesis

**Avantages** :
- ✅ **Cohérence** : Même module VTL que les plateformes de collecte
- ✅ **Pas de duplication** : Plus de recalcul dans Kraftwerk
- ✅ **Performance** : Calculs faits une fois, stockés
- ✅ **Simplification** : Kraftwerk ne fait plus de calculs VTL

**Inconvénients** :
- ⚠️ **Temps de process** : Légèrement augmenté (calculs faits à la collecte)
- ⚠️ **Volume Genesis** : Légèrement augmenté (stockage des résultats)

**Impact** : 
- Suppression de `VtlBindings.java`, `MultimodalSequence.java` (partie VTL)
- Suppression des dépendances Trevas
- Simplification du flux de traitement

#### Remplacement pour les Scripts "Aval"
**Action** : Étudier les usages et proposer des services adaptés

**Options** :
- **Option A** : **SQL** - Proposer des requêtes SQL pour les traitements aval
- **Option B** : **Services dédiés** - Créer des endpoints spécifiques pour les besoins aval
- **Option C** : **Scripts externes** - Externaliser complètement les scripts aval

**Recommandation** : **Option A + B** - Combiner SQL pour les cas simples et services dédiés pour les cas complexes

---

## Décisions Fonctionnelles (Client)

### ✅ Fonctionnalités à Re-questionner

1. **Scripts "aval"** : À re-questionner - usage à étudier
2. **Reportings Sabiane** : À re-questionner - pertinence à valider

**Action** : 
- Analyser les usages actuels
- Consulter les utilisateurs (CPIE)
- Décider : conserver, adapter ou supprimer

---

## Synthèse des Besoins

### Besoins Fonctionnels (Must Have)

| ID | Besoin | Priorité | Complexité |
|----|--------|----------|------------|
| FH-001 | Export CSV/Parquet/JSON ponctuel | ⭐⭐⭐⭐⭐ | Moyenne |
| FH-002 | Gestion des boucles | ⭐⭐⭐⭐⭐ | Moyenne |
| FH-003 | Liens 2 à 2 | ⭐⭐⭐⭐⭐ | Moyenne |
| FH-004 | Indicatrice FILTER_RESULT | ⭐⭐⭐⭐ | Faible |
| FH-005 | Indicatrice MISSING | ⭐⭐⭐⭐ | Faible |
| FH-006 | Export automatisé | ⭐⭐⭐⭐⭐ | Moyenne |
| FH-007 | Variables calculées (via Genesis) | ⭐⭐⭐⭐ | Faible |
| FH-008 | Logs efficaces | ⭐⭐⭐⭐ | Moyenne |
| FH-009 | Alerting | ⭐⭐⭐ | Moyenne |

### Besoins Évolutifs (Future)

| ID | Besoin | Priorité | Complexité |
|----|--------|----------|------------|
| EV-001 | Export par lot (collectionInstrumentId) | ⭐⭐⭐ | Élevée |
| EV-002 | Export multimode sans réconciliation | ⭐⭐⭐ | Élevée |
| EV-003 | Intégration codification | ⭐⭐ | Élevée |
| EV-004 | Revoir types de variable | ⭐⭐ | Très Élevée |
| EV-005 | Export état à date T | ⭐⭐ | Élevée |

### Besoins Non Fonctionnels

| ID | Besoin | Priorité | Mesurable |
|----|--------|----------|-----------|
| NF-001 | Réduire consommation mémoire | ⭐⭐⭐⭐⭐ | < 1 Go pour 1800 interrogations |
| NF-002 | Améliorer temps de traitement | ⭐⭐⭐⭐ | Réduction de 50% |
| NF-003 | Migration progressive | ⭐⭐⭐⭐⭐ | Sans big bang |
| NF-004 | Maintenabilité | ⭐⭐⭐⭐ | Architecture modulaire |
| NF-005 | Documenté | ⭐⭐⭐⭐ | 100% des fonctionnalités documentées |
| NF-006 | Testé | ⭐⭐⭐⭐ | Couverture > 80% |

---

## Contraintes

### Contraintes Techniques
- **JDK 25** : Obligatoire (décision client)
- **Spring Boot 4.1.0** : Obligatoire (décision client)
- **Maven** : Obligatoire (standard INSEE)
- **Genesis** : Source de données principale
- **MinIO** : Stockage objet (optionnel mais recommandé)

### Contraintes Fonctionnelles
- **Compatibilité descendante** : Pas de breaking changes pendant la migration
- **Performance** : < 1 Go RAM pour 1800 interrogations
- **Stabilité** : Pas de régression fonctionnelle

### Contraintes Organisationnelles
- **Migration progressive** : Par fonctionnalité
- **Rollback possible** : À tout moment
- **Validation utilisateur** : Implication des CPIE

---

## Impact des Décisions Client

### Sur l'Architecture

1. **Java 25 + Spring Boot 4** :
   - ✅ **Avantages** : Accès aux dernières fonctionnalités
   - ⚠️ **Risques** : Moins de stabilité, dépendances potentiellement non matures
   - 🎯 **Recommandation** : Surveiller les versions, tester rigoureusement

2. **Suppression de VTL** :
   - ✅ **Simplification** : Moins de code, moins de complexité
   - ✅ **Cohérence** : Alignement avec la filière
   - ⚠️ **Dépendance** : Dépendance accrue sur Genesis et Perret
   - 🎯 **Action** : Valider que Genesis et Perret supporteront ce flux

### Sur les Performances

**Objectif** : Réduire de **5 Go → < 1 Go** pour 1800 interrogations

**Leviers identifiés** :
1. **Traitement streaming** : Ne plus charger toutes les données en mémoire
2. **Parallélisation** : Traitement parallèle des fichiers
3. **Suppression VTL** : Moins de calculs lourds
4. **Optimisation DuckDB** : Meilleure gestion des connexions
5. **Externalisation** : Décharger certains traitements sur Genesis

---

## Prochaines Étapes

### 1. Clarifier les Points Ouverts
- [ ] **Scripts "aval"** : Étudier les usages, proposer des alternatives
- [ ] **Reportings Sabiane** : Valider la pertinence, proposer une solution
- [ ] **Module VTL externe** : Valider l'existence et la disponibilité
- [ ] **Genesis** : Valider que Genesis peut stocker les variables calculées

### 2. Prioriser les Évolutions
- [ ] **Export par lot** : Priorité ?
- [ ] **Multimode sans réconciliation** : Priorité ?
- [ ] **Codification** : Priorité ?
- [ ] **Types de variable** : Priorité ?
- [ ] **Export à date T** : Priorité ?

### 3. Définir la Stratégie de Migration
- [ ] **Ordre des fonctionnalités** à migrer
- [ ] **Stratégie de coexistence** ancienne/nouvelle version
- [ ] **Mécanisme de rollback**
- [ ] **Critères de validation** par fonctionnalité

---

## Questions pour Validation

1. **Module VTL externe** : 
   - [ ] Existe-t-il déjà un module VTL en JavaScript identique aux plateformes ?
   - [ ] Qui le maintient ?
   - [ ] Comment l'intégrer ? (API REST, librairie, autre)

2. **Genesis** :
   - [ ] Genesis peut-il stocker les variables calculées ?
   - [ ] Perret peut-il renvoyer les variables calculées à Genesis ?
   - [ ] Quel est le format de stockage des variables calculées ?

3. **Performances** :
   - [ ] Quel est le volume typique des données à traiter ?
   - [ ] Quelle est la fréquence des exports ?
   - [ ] Quel est le SLA attendu ?

4. **Migration** :
   - [ ] Quelles fonctionnalités migrer en premier ?
   - [ ] Quel est le calendrier souhaité ?
   - [ ] Qui valide les livrables ?

---

## Conclusion

La refonte de Kraftwerk a pour objectif de **résoudre les problèmes de maintenance, performance et évolutivité** tout en **conservant les fonctionnalités existantes** et en **permettant les évolutions futures**.

**Décisions clés déjà prises** :
- Java 25 + Spring Boot 4
- Suppression de VTL de Kraftwerk
- Migration progressive sans big bang

**Prochaines étapes** :
1. Valider les points ouverts (VTL externe, Genesis, scripts aval)
2. Prioriser les évolutions fonctionnelles
3. Définir la stratégie de migration
4. Lancer l'analyse architecturale détaillée

---

*Document basé sur les spécifications client - 2026-07-08*
