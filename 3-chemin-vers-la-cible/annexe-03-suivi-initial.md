> ⚠️ **ANNEXE ARCHIVÉE (état d'avancement initial, « Mistral Vibe »).** Pour l'état à jour, voir le [`README.md`](../README.md) et le backlog [`01-backlog-mvp-vers-cible.md`](./01-backlog-mvp-vers-cible.md). Conservé pour mémoire.

# Analyses Terminées - Kraftwerk Refonte

**Date** : 2026-07-08  
**Version Kraftwerk** : 4.1.3  
**Stack Cible** : Java 25 + Spring Boot 4.1.0

---

## 📊 **Bilan des Analyses Réalisées**

### ✅ **Analyse Complète de l'Existant**

| Document | Contenu | Statut |
|----------|---------|--------|
| [01-architecture-globale.md](../1-existant/01-architecture-globale.md) | Architecture modulaire, stack technique, services REST, processus métier | ✅ **Terminé** |
| [02-fonctionnalites-principales.md](../1-existant/02-fonctionnalites-principales.md) | Toutes les fonctionnalités par module, endpoints, paramètres, cas d'usage | ✅ **Terminé** |
| [03-choix-architecturaux-et-problemes.md](../1-existant/03-choix-architecturaux-et-problemes.md) | 37 recommandations, problèmes identifiés, axes d'amélioration | ✅ **Terminé** |
| [04-comparaison-code-tests-guide.md](../1-existant/04-comparaison-code-tests-guide.md) | 10 incohérences identifiées, 9 questions pour décision | ✅ **Terminé** |
| [01-objectifs-refonte.md](../2-cible/01-objectifs-refonte.md) | Objectifs client, décisions validées, besoin fonctionnels/techniques | ✅ **Terminé** |
| [annexe-01-choix-architecturaux-initiaux.md](./annexe-01-choix-architecturaux-initiaux.md) | **20 décisions architecturales détaillées** avec code exemples | ✅ **Terminé** |

---

## 🎯 **Décisions Validées par le Client**

### **Stack Technique**
- ✅ **Java 25** (malgré recommandation JDK 21 LTS)
- ✅ **Spring Boot 4.1.0** (malgré recommandation 3.x LTS)
- ✅ **Maven** (standard INSEE)

### **Architecture**
- ✅ **Séparation en 3 modules** : core, api, batch
- ✅ **Pattern Ports & Adapters** (Clean Architecture)
- ✅ **Migration progressive** (Strangler Fig Pattern)

### **Fonctionnalités**
- ✅ **Conserver toutes les fonctionnalités actuelles** (CSV/Parquet/JSON, boucles, liens 2à2, FILTER_RESULT, MISSING)
- ✅ **Supprimer VTL de Kraftwerk** (externaliser vers Genesis/Perret)

### **Évolutions**
- ⏳ **Export par lot** (à prioriser)
- ⏳ **Export multimode sans réconciliation** (à prioriser)
- ⏳ **Intégration codification** (à prioriser)
- ⏳ **Revoir types de variable** (complexe, dépend de Genesis)
- ⏳ **Export à date T** (à prioriser)

---

## 🏗️ **Architecture Cible Proposée**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE KRAFTWERK 4.2.0                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────┐ │
│  │   API Module     │     │  Batch Module    │     │  Core Module │ │
│  │   (Spring Boot)  │     │   (Spring Boot)  │     │   (Métier)   │ │
│  └────────┬────────┘     └────────┬────────┘     └──────┬──────┘ │
│           │                      │                     │         │
│           └──────────────────────┼─────────────────────┘         │
│                                │                                   │
│                                ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    CORE MODULE (Métier)                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │ │
│  │  │   Domain    │  │    Ports    │  │  Processing       │   │ │
│  │  │   (Entités) │  │ (Interfaces)│  │   (Pipeline)      │   │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   EXTERNAL SYSTEMS                             │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌─────────┐  │ │
│  │  │  Genesis   │  │  MinIO     │  │  Vault     │  │  VTL     │  │ │
│  │  │  (Data+    │  │  (Storage) │  │ (Secrets)  │  │ (JS)     │  │ │
│  │  │  Calculée) │  │            │  │            │  │         │  │ │
│  │  └───────────┘  └───────────┘  └───────────┘  └─────────┘  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📈 **Gains Attendus**

| Métrique | Actuel | Cible | Gain | Priorité |
|----------|--------|-------|------|----------|
| **Consommation mémoire** | 5 Go | < 1 Go | **80%** | ⭐⭐⭐⭐⭐ |
| **Temps de traitement** | T | T/4 (4 cœurs) | **75%** | ⭐⭐⭐⭐⭐ |
| **Temps de développement** | Élevé | Réduit de 50% | **50%** | ⭐⭐⭐⭐ |
| **Complexité évolutions** | Élevée | Réduite de 70% | **70%** | ⭐⭐⭐⭐ |
| **Maintenabilité** | Moyenne | Élevée | **50%** | ⭐⭐⭐⭐ |
| **Taille image Docker** | > 500 Mo | < 200 Mo | **60%** | ⭐⭐⭐ |

---

## 🔧 **Choix Architecturaux Clés**

### **1. Architecture Modulaire (3 Modules)**
- **kraftwerk-core** : Logique métier commune
- **kraftwerk-api** : Application REST Spring Boot
- **kraftwerk-batch** : Application CLI Spring Boot
- **Avantages** : Découplage, optimisation dépendances, déploiement flexible

### **2. Pattern Ports & Adapters (Clean Architecture)**
- **Domain Layer** : Entités, Use Cases, Services métier
- **Port Layer** : Interfaces (InputPort, ProcessingPort, OutputPort, etc.)
- **Adapter Layer** : Implémentations concrètes (GenesisAdapter, MinioAdapter, etc.)
- **Avantages** : Maintenabilité, testabilité, découplage des technologies

### **3. Traitement Streaming + Parallèle**
- **Java Streams + Project Reactor** : Traitement sans chargement complet en mémoire
- **Parallel Processing** : Utilisation optimale des CPU multi-cores
- **Gain** : -80% mémoire, -75% temps

### **4. Suppression de VTL**
- **Actuellement** : Kraftwerk calcule les variables avec Trevas (Java)
- **Cible** : Genesis stocke les variables calculées (calculées par module VTL JS externe)
- **Avantages** : Simplification, cohérence avec la filière, suppression des bugs de somme

### **5. Pipeline de Traitement Optimisé**
```
Input (Streaming) → Parsing → Enrichissement (Parallèle) → Transformation → 
Agrégation → Output (Streaming)
```

### **6. Gestion des Jobs Persistante**
- **Actuellement** : InMemoryJobStore (jobs perdus au restart)
- **Cible** : Spring Batch + Base de données (PostgreSQL/H2)
- **Avantages** : Persistance, historique, reprise après crash

### **7. Stockage Optimisé**
- **Cache Local** : Téléchargement Genesis → Traitement local → Upload MinIO
- **Fallback** : Basculer sur FileSystem si MinIO indisponible
- **Avantages** : Réduction des accès réseau, performance

### **8. Client Genesis Optimisé**
- **Pagination** : Récupération par pages (1000 items/page)
- **Streaming** : Flux continu vers le pipeline
- **Avantages** : Mémoire constante, début de traitement immédiat

### **9. Logging Structuré + Metrics**
- **Structured Logging** : JSON format avec Logstash
- **Correlation ID** : Suivi des requêtes à travers les services
- **Metrics** : Micrometer + Prometheus pour le monitoring
- **Avantages** : Observabilité, analyse ELK, alertes

### **10. Configuration Externalisée**
- **Hiérarchie** : Default (JAR) → Environnement → Vault → Config Server
- **Vault** : Secrets et configuration sensible
- **Avantages** : Sécurité, flexibilité, maintenance

---

## 🗺️ **Roadmap de la Refonte**

### **Phase 0 : Préparation (1-2 semaines)**
- [ ] Valider les points ouverts (VTL externe, Genesis, scripts aval)
- [ ] Prioriser les fonctionnalités à migrer
- [ ] Définir la stratégie de migration détaillée
- [ ] Configurer l'environnement de développement (JDK 25, Spring Boot 4)
- [ ] Mettre en place les outils (CI/CD, Docker, monitoring)

### **Phase 1 : Fondations (Sprint 1-2)**
- [ ] Créer la structure modulaire (parent/core/api/batch)
- [ ] Configurer Spring Boot 4.1.0 + JDK 25
- [ ] Implémenter le pattern Ports & Adapters
- [ ] Configurer le pipeline CI/CD
- [ ] Supprimer VTL et ses dépendances (Trevas)
- [ ] Implémenter le client Genesis optimisé (pagination + streaming)

### **Phase 2 : Fonctionnalités de Base (Sprint 3-6)**
- [ ] Implémenter l'export CSV/Parquet simple (sans boucles)
- [ ] Implémenter la gestion des boucles
- [ ] Implémenter l'export Genesis (by questionnaireModelId)
- [ ] Implémenter le traitement streaming/parallèle
- [ ] Ajouter les tests unitaires (JUnit 5 + Mockito)
- [ ] Ajouter les tests d'intégration (@SpringBootTest)

### **Phase 3 : Fonctionnalités Avancées (Sprint 7-8)**
- [ ] Implémenter le reporting data
- [ ] Implémenter les scripts aval (SQL + Services REST)
- [ ] Ajouter la persistance des jobs (Spring Batch + DB)
- [ ] Optimiser les performances (benchmark + tuning)
- [ ] Ajouter le monitoring (Metrics + Structured Logging)
- [ ] Ajouter les tests de sécurité

### **Phase 4 : Évolutions Fonctionnelles (Sprint 9-10)**
- [ ] Implémenter l'export par lot (collectionInstrumentId)
- [ ] Implémenter l'export multimode sans réconciliation
- [ ] Préparer l'intégration codification
- [ ] Implémenter l'export à date T

### **Phase 5 : Migration et Cleanup (Sprint 11-12)**
- [ ] Migrer les utilisateurs par fonctionnalité (feature flags)
- [ ] Supprimer l'ancienne version
- [ ] Archiver le code legacy
- [ ] Documenter complètement (guide utilisateur + technique)
- [ ] Finaliser les tests de performance (Gatling)

---

## 🎯 **Prochaines Étapes Immédiates**

### **Pour Vous (Client)**

1. **✅ Valider les choix architecturaux** proposés dans [annexe-01-choix-architecturaux-initiaux.md](./annexe-01-choix-architecturaux-initiaux.md)
2. **⏳ Répondre aux questions ouvertes** :
   - Module VTL externe existe-t-il ? Comment l'intégrer ?
   - Genesis peut-il stocker les variables calculées ?
   - Perret peut-il renvoyer les variables calculées à Genesis ?
   - Quels sont les usages des scripts "aval" ?
   - Les reportings Sabiane sont-ils toujours nécessaires ?
3. **⏳ Prioriser les fonctionnalités** pour la migration
4. **⏳ Nommer un responsable** pour la validation des livrables

### **Pour Moi (Assistant)**

Une fois vos réponses reçues, je pourrai :

1. **Finaliser l'architecture cible** avec vos décisions
2. **Créer le squelette du projet** :
   - Structure Maven (parent/core/api/batch)
   - Configuration Spring Boot 4 + JDK 25
   - Implémentation du pattern Ports & Adapters
   - Configuration de base (logging, metrics, sécurité)
3. **Produire les spécifications détaillées** :
   - Spécifications techniques (diagrammes, séquences)
   - Spécifications fonctionnelles (user stories, critères d'acceptation)
   - Règles métiers extraites et formalisées
4. **Extraire les cas de test** :
   - Cas nominaux par fonctionnalité
   - Edge cases à tester
   - Scénarios Cucumber mis à jour

---

## 📦 **Livrables à Produire**

### **Déjà Produits** ✅
- [x] Analyse complète de l'existant
- [x] Identification des incohérences
- [x] Documentation des objectifs client
- [x] Propositions architecturales détaillées

### **À Produire** (En attente de vos réponses)
- [ ] **Squelette du projet** (structure Maven + configuration)
- [ ] **Spécifications techniques détaillées**
- [ ] **Spécifications fonctionnelles**
- [ ] **Règles métiers formalisées**
- [ ] **Cas de test complets**
- [ ] **Plan de migration détaillé**
- [ ] **Documentation technique** (README, JavaDoc, ADR)
- [ ] **Documentation utilisateur mise à jour**

---

## 💡 **Recommandations Finale**

### **Priorités Absolues**
1. **Commencer par la structure modulaire** : C'est la fondation de tout le reste
2. **Supprimer VTL en premier** : Gain immédiat en simplicité et performance
3. **Implémenter le streaming** : Réduction drastique de la mémoire
4. **Ajouter les tests** : Garantir la qualité dès le début

### **À Éviter**
1. ❌ **Trop optimiser trop tôt** : Commencer simple, optimiser après
2. ❌ **Tout migrer en une fois** : Migration progressive par fonctionnalité
3. ❌ **Négocier la stack technique** : Java 25 + Spring Boot 4 est validé
4. ❌ **Oublier la documentation** : Documenter chaque décision (ADR)

### **Bonnes Pratiques**
1. ✅ **Feature Flags** : Pour la migration progressive
2. ✅ **Tests Automatiques** : 100% des nouvelles fonctionnalités
3. ✅ **Revue de Code** : Pour chaque PR
4. ✅ **Monitoring** : Dès le début du projet
5. ✅ **Feedback Utilisateur** : Validation à chaque sprint

---

## 📞 **Contact et Support**

Pour toute question ou clarification sur cette analyse, je suis disponible pour :
- Expliquer un choix architectural
- Détailer une recommandation
- Proposer des exemples de code supplémentaires
- Aider à la prise de décision

**Format de demande recommandé** :
```
Question sur : [Nom du document/section]
Contexte : [Brève description]
Options envisagées : [A, B, C]
Recommandation attendue : [Oui/Non]
```

---

## 🎉 **Résumé Exécutif**

| Aspect | État Actuel | État Cible | Gain |
|--------|-------------|------------|------|
| **Architecture** | Monolithique | 3 modules découplés | ⭐⭐⭐⭐ |
| **Performance** | 5 Go RAM, lent | < 1 Go RAM, rapide | ⭐⭐⭐⭐⭐ |
| **Maintenabilité** | Complexe | Simple et modulaire | ⭐⭐⭐⭐ |
| **Évolutivité** | Difficile | Facile | ⭐⭐⭐⭐ |
| **Documentation** | Partielle | Complète | ⭐⭐⭐⭐ |
| **Tests** | Incomplets | Pyramide complète | ⭐⭐⭐⭐ |

**Temps estimé** : 10-12 semaines (5 sprints de 2 semaines)  
**Équipe recommandée** : 2-3 développeurs + 1 architecte  
**Coût estimé** : À calculer selon les ressources disponibles

---

*Document de synthèse - Kraftwerk Refonte - 2026-07-08*
