# Kraftwerk - Choix Architecturaux et Problèmes Identifiés

## Date d'analyse : 2026-07-08

---

## Choix Architecturaux Majeurs

## 1. Architecture Modulaire Maven

### Organisation
```
parent-pom (4.1.3)
├── kraftwerk-core/      # Logique métier
├── kraftwerk-api/       # API REST
├── kraftwerk-encryption/# Module de chiffrement
└── tests/               # Tests fonctionnels
```

### Avantages
✅ **Séparation des responsabilités** : Core contient la logique métier, API expose les services
✅ **Réutilisabilité** : Le module core peut être utilisé indépendamment
✅ **Maintenabilité** : Each module a son propre scope
✅ **Tests isolés** : Chaque module peut être testé séparément

### Inconvénients
⚠️ **Dépendances croisées** : kraftwerk-api dépend de kraftwerk-core, mais aussi de kraftwerk-encryption
⚠️ **Duplication de configuration** : Certaines configurations sont répétées
⚠️ **Modules tests séparés** : Les tests fonctionnels sont dans un module séparé, ce qui complique l'intégration

---

## 2. Spring Boot 4.1.0

### Avantages
✅ **Framework moderne** : Spring Boot 4.x offre les dernières fonctionnalités
✅ **Auto-configuration** : Réduction du code boilerplate
✅ **Spring Ecosystem** : Intégration facile avec Spring Security, Spring Data, etc.

### ⚠️ Problèmes et Risques

**Version Non-LTS** : Spring Boot 4.1.0 n'est pas une version LTS
- Les versions LTS (Long Term Support) sont plus stables
- Spring Boot 3.x est la dernière LTS (2026)
- Migration recommandée vers 3.x pour la production

**Compatibilité JDK** : Configure pour JDK 25
- JDK 25 n'est pas encore largement adopté en production
- JDK 21 LTS serait plus approprié
- Risque de problèmes de compatibilité avec les dépendances

**Recommandation** : 
- Passer sur **Spring Boot 3.2.x ou 3.3.x** (LTS)
- Utiliser **JDK 21 LTS** pour une meilleure stabilité

---

## 3. Mode Double : API + Batch

### Architecture Actuelle

```java
// Dans KraftwerkApi.java
public static void main(String[] args) {
    boolean batchMode = Arrays.stream(args)
            .anyMatch(arg -> arg.startsWith("--service="));
    
    if(batchMode) {
        app.run(args).close(); // Batch mode - exécute et termine
    } else {
        app.run(args); // API mode - reste en écoute
    }
}
```

### Avantages
✅ **Flexibilité** : Une seule application pour deux modes d'utilisation
✅ **Réutilisation du code** : Pas de duplication de la logique métier
✅ **Simplicité de déploiement** : Un seul artefact à déployer

### Problèmes Identifiés

#### a) Complexité de la Configuration
- Le mode batch nécessite des paramètres en ligne de commande
- Le mode API utilise des propriétés Spring
- **Problème** : Mix de deux paradigmes de configuration

#### b) Gestion des Dépendances
- En mode batch : pas besoin de Spring Security, OAuth2, etc.
- En mode API : toutes ces dépendances sont nécessaires
- **Problème** : L'application batch charge des dépendances inutiles

#### c) Cycle de Vie Différent
- API : Long-running process
- Batch : Exécution courte avec startup/shutdown
- **Problème** : Optimisation difficile pour les deux cas

#### d) Tests Complexes
- Difficile de tester le mode batch dans un contexte Spring
- **Problème** : Les tests d'intégration sont complexes

### Recommandations

**Option 1 : Séparer les Modules**
```
kraftwerk-core/          # Logique métier commune
├── kraftwerk-api/       # Application API Spring Boot
└── kraftwerk-batch/     # Application Batch (CLI)
```
- Avantages : Optimisation par cas d'usage, dépendances minimales
- Inconvénients : Duplication de la configuration commune

**Option 2 : Utiliser Spring Shell**
- Intégrer Spring Shell pour les commandes batch
- Avantages : Une seule application, interface CLI riche
- Inconvénients : Complexité accrue, dépendance supplémentaire

**Option 3 : Conserver l'approche actuelle mais améliorer**
- Créer un `CommandLineRunner` dédié pour le batch
- Isoler la configuration batch de l'API
- Avantages : Moins de changements, évolution progressive

---

## 4. Gestion des Jobs Asynchrones

### Implémentation Actuelle

```java
// InMemoryJobStore.java
public class InMemoryJobStore {
    private final Map<String, JobExecution> jobs = new ConcurrentHashMap<>();
    
    public void start(String jobId) { ... }
    public Optional<JobExecution> get(String jobId) { ... }
}
```

### Avantages
✅ **Simplicité** : Implémentation en mémoire, facile à comprendre
✅ **Performance** : Pas de latence de base de données
✅ **Pas de dépendance externe** : Pas besoin de Redis ou autre

### ⚠️ Problèmes Identifiés

#### a) Pas de Persistance
- Les jobs sont perdus au redémarrage de l'application
- **Impact** : Impossible de suivre les jobs après un restart

#### b) Scalabilité Limitée
- Stockage en mémoire : problème avec beaucoup de jobs simultanés
- **Impact** : Limite le nombre de jobs concurrents

#### c) Pas de Notifications
- Pas de mécanisme de callback ou webhook
- **Impact** : Le client doit poller pour connaître le statut

#### d) Deux Implémentations Similaires
- `InMemoryJobStore` et `InMemoryExportJobStore` : duplication de code
- **Problème** : Violation du principe DRY

### Recommandations

1. **Unifier les JobStores** : Créer une seule implémentation générique
2. **Ajouter la Persistance** : Utiliser une base de données (JPA + Hibernate) ou Redis
3. **Ajouter des Webhooks** : Notifier le client quand le job est terminé
4. **Limiter le Nombre de Jobs** : Configuration du maximum de jobs simultanés

---

## 5. Traitement des Données : Approche Séquentielle

### Implémentation Actuelle

```java
// MainProcessing.java
public void runMain() throws KraftwerkException {
    init();
    for (UserInputsFile userFile : userInputsFileList) {
        unimodalProcess();
        multimodalProcess();
        insertDatabase();
    }
    outputFileWriter();
    writeErrors();
    writeLog();
}
```

### Avantages
✅ **Simplicité** : Facile à comprendre et déboguer
✅ **Contrôle du flux** : Ordre explicite des étapes

### ⚠️ Problèmes Identifiés

#### a) Pas de Traitement Parallèle
- Les fichiers sont traités séquentiellement
- **Impact** : Performance limitée sur les machines multi-cores

#### b) Chargement en Mémoire
- Toutes les données sont chargées avant traitement
- **Impact** : Problèmes de mémoire avec les gros fichiers (limite à 400Mo)

#### c) Couplage Fort
- MainProcessing dépend directement de :
  - ControlInputSequence
  - BuildBindingsSequence
  - UnimodalSequence
  - MultimodalSequence
  - InsertDatabaseSequence
  - WriterSequence
- **Problème** : Difficile de modifier un composant sans affecter les autres

#### d) Gestion des Erreurs Partielle
- Certaines erreurs sont capturées, d'autres non
- **Impact** : Comportement incohérent en cas d'erreur

### Recommandations

1. **Traitement Parallèle** : Utiliser `CompletableFuture` ou `ExecutorService`
2. **Streaming** : Utiliser Java Streams pour le traitement des gros fichiers
3. **Dé-couplage** : Utiliser des événements (Spring ApplicationEvent) ou un bus d'événements
4. **Gestion des Erreurs Centralisée** : Utiliser `@ControllerAdvice` + exceptions personnalisées

---

## 6. Base de Données : DuckDB

### Utilisation Actuelle

```java
// SqlUtils.java
public static Connection openConnection() throws SQLException {
    return DriverManager.getConnection("jdbc:duckdb:");
}
```

### Avantages
✅ **In-memory** : Pas de dépendance externe, rapide
✅ **SQL compatible** : Permet d'utiliser des requêtes SQL
✅ **Léger** : Pas de serveur de base de données à déployer

### ⚠️ Problèmes Identifiés

#### a) Pas de Persistance
- Les données sont perdues à la fin du traitement
- **Impact** : Impossible de reprendre un traitement interrompu

#### b) Gestion des Connexions
- Les connexions sont créées et fermées à chaque traitement
- **Impact** : Overhead de création de connexion

#### c) Limitée aux Traitements Batch
- DuckDB est optimisé pour l'analytics, pas pour les transactions
- **Impact** : Pas adapté pour un usage en temps réel

### Recommandations

1. **Pool de Connexions** : Utiliser HikariCP pour gérer les connexions
2. **Alternative** : Considérer H2 Database pour plus de persistance
3. **Externalisation** : Pour les gros volumes, utiliser PostgreSQL ou autre

---

## 7. Gestion des Fichiers : FileSystem vs MinIO

### Implémentation Actuelle

```java
// FileUtilsInterface
public interface FileUtilsInterface {
    boolean isDirectory(String path);
    List<String> listFilePaths(String path);
    long getSizeOf(String path);
    // ...
}

// Implémentations
public class FileSystemImpl implements FileUtilsInterface { ... }
public class MinioImpl implements FileUtilsInterface { ... }
```

### Avantages
✅ **Abstraction** : Le code métier ne dépend pas de l'implémentation
✅ **Flexibilité** : Possibilité de changer de stockage facilement
✅ **Tests** : Facile de mock pour les tests

### ⚠️ Problèmes Identifiés

#### a) Configuration Complexe
- MinIO nécessite une configuration spécifique (endpoint, credentials, bucket)
- **Problème** : Difficile à configurer et maintenir

#### b) Gestion des Erreurs
- Pas de gestion centralisée des erreurs de fichier
- **Impact** : Messages d'erreur incohérents

#### c) Performances MinIO
- Accès distant vs local : latence accrue
- **Impact** : Ralentissement des traitements

### Recommandations

1. **Configuration Simplifiée** : Utiliser Spring Cloud Config ou Vault pour MinIO
2. **Cache Local** : Télécharger les fichiers MinIO localement avant traitement
3. **Fallback** : Si MinIO indisponible, basculer sur FileSystem
4. **Metrics** : Ajouter des métriques sur les opérations de fichier

---

## 8. Intégration VTL : Trevas

### Utilisation Actuelle

```java
// VtlBindings.java
public class VtlBindings {
    private Map<String, DataFrame> dataFrames = new HashMap<>();
    // Gestion des bindings pour les expressions VTL
}
```

### Avantages
✅ **Standard** : VTL est un standard SDMX
✅ **Flexibilité** : Permet des transformations complexes
✅ **Validation** : Validation des données selon les règles métier

### ⚠️ Problèmes Identifiés

#### a) Couplage avec Trevas
- Dépendance externe forte
- **Problème** : Versioning, compatibilité, maintenance

#### b) Performances
- Évaluation des expressions VTL peut être lente
- **Impact** : Ralentissement des traitements complexes

#### c) Gestion des Erreurs
- Les erreurs VTL ne sont pas toujours bien propagées
- **Impact** : Difficile de déboguer les transformations

### Recommandations

1. **Version Pinning** : Fixer la version de Trevas dans le pom.xml
2. **Cache des Résultats** : Cacher les résultats des expressions VTL fréquemment utilisées
3. **Logging Amélioré** : Ajouter du logging détaillé pour les transformations VTL
4. **Tests Unitaires** : Ajouter des tests unitaires pour les règles VTL

---

## 9. Sécurité

### Implémentation Actuelle

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Avantages
✅ **Standard** : Utilisation des standards Spring Security
✅ **OAuth2** : Support des tokens JWT
✅ **Configurable** : Flexible et extensible

### ⚠️ Problèmes Identifiés

#### a) Configuration Minimale
- Peu de détails sur la configuration de sécurité dans le code
- **Problème** : Difficile de savoir quels endpoints sont protégés

#### b) Pas de Tests de Sécurité
- Aucun test visible pour la sécurité
- **Impact** : Risque de vulnérabilités non détectées

#### c) OAuth2 Resource Server
- Utilisation de OAuth2 Resource Server (pour les APIs)
- **Question** : Est-ce que c'est un Client ou un Resource Server ?
- **Impact** : Configuration différente selon le rôle

### Recommandations

1. **Configuration Explicite** : Documenter la configuration de sécurité
2. **Tests de Sécurité** : Ajouter des tests pour la sécurité (Spring Security Test)
3. **Audit** : Vérifier les permissions sur chaque endpoint
4. **Rate Limiting** : Ajouter du rate limiting pour éviter les abus

---

## 10. Logging

### Implémentation Actuelle

- Utilisation de **Log4j2** (via Spring Boot Starter Logging)
- Configuration dans `application.properties`

### Avantages
✅ **Standard** : Log4j2 est performant et largement utilisé
✅ **Flexible** : Configuration facile via properties

### ⚠️ Problèmes Identifiés

#### a) Niveau de Logging Incohérent
- Certains logs en INFO, d'autres en DEBUG ou WARN
- **Impact** : Difficile de configurer le niveau de logging

#### b) Messages de Log Non Structurés
- Pas de format JSON pour les logs
- **Impact** : Difficile à analyser avec des outils modernes (ELK, etc.)

#### c) Pas de Correlation ID
- Pas d'ID de corrélation pour suivre une requête à travers les services
- **Impact** : Difficile de déboguer les problèmes en production

### Recommandations

1. **Structured Logging** : Utiliser Logstash Logback Encoder ou autre
2. **Correlation ID** : Ajouter un filtre pour générer un correlation ID
3. **Niveaux de Logging Standardisés** : Définir une politique de logging
4. **Centralisation** : Configurer l'envoi des logs vers un serveur central

---

## 11. Tests

### Implémentation Actuelle

#### Tests Unitaires
- **JUnit 5** : Framework de test
- **AssertJ** : Assertions fluides
- **Mockito** : (pas visible, mais probablement utilisé)

#### Tests Fonctionnels
- **Cucumber 7.34.4** : Tests BDD
- Step definitions en Java
- Scénarios dans des fichiers .feature

#### Tests de Mutation
- **PITest** : Tests de mutation pour vérifier la qualité des tests

### Avantages
✅ **Couverture** : Bonne couverture des fonctionnalités principales
✅ **BDD** : Scénarios fonctionnels bien décrits
✅ **Qualité** : Utilisation de PITest pour la qualité des tests

### ⚠️ Problèmes Identifiés

#### a) Tests Lents
- Les tests fonctionnels Cucumber nécessitent un setup complexe
- **Impact** : Temps d'exécution long

#### b) Duplication de Code
- Beaucoup de code dupliqué dans les step definitions
- **Impact** : Maintenance difficile

#### c) Peu de Tests d'Intégration
- Peu de tests qui testent l'intégration entre les composants
- **Impact** : Risque d'erreurs d'intégration

#### d) Tests de Sécurité Manquants
- Aucun test visible pour la sécurité
- **Impact** : Risque de vulnérabilités

#### e) Tests de Performance Manquants
- Aucun test de performance visible
- **Impact** : Difficile d'identifier les goulots d'étranglement

### Recommandations

1. **Tests d'Intégration** : Ajouter des tests d'intégration avec `@SpringBootTest`
2. **Tests de Sécurité** : Ajouter des tests pour la sécurité
3. **Tests de Performance** : Ajouter des tests de performance (JMH, Gatling)
4. **Refactoring des Step Definitions** : Extraire la logique commune
5. **Tests Parallèles** : Configurer l'exécution parallèle des tests
6. **Rapport de Couverture** : Générer des rapports de couverture (JaCoCo)

---

## 12. Build et Déploiement

### Implémentation Actuelle

- **Maven** : Outil de build
- **Spring Boot Maven Plugin** : Pour créer l'executable JAR
- **Dockerfile** : Pour la conteneurisation
- **GitHub Actions** : (probablement) pour le CI/CD

### Avantages
✅ **Standard** : Utilisation de Maven, standard dans l'écosystème Java
✅ **Containerisé** : Dockerfile présent pour la conteneurisation

### ⚠️ Problèmes Identifiés

#### a) Configuration CI/CD Non Visible
- Le fichier .github/ n'est pas analysé
- **Impact** : Difficile de connaître le processus de CI/CD

#### b) Multi-module Complexe
- Build Maven multi-module peut être lent
- **Impact** : Temps de build long

#### c) Pas de Multi-stage Build
- Dockerfile semble simple, pas de multi-stage build
- **Impact** : Image Docker plus grosse que nécessaire

### Recommandations

1. **Optimisation du Build Maven** : Utiliser des plugins pour accélérer le build
2. **Multi-stage Docker Build** : Réduire la taille de l'image
3. **CI/CD Pipeline** : Documenter et optimiser le pipeline
4. **Artifact Repository** : Utiliser Nexus ou Artifactory pour les dépendances

---

## Synthèse des Problèmes Majeurs

| Catégorie | Problème | Impact | Priorité | Solution Proposée |
|-----------|----------|--------|----------|-------------------|
| **Stack Technique** | Spring Boot 4.1.0 non-LTS | Instabilité potentielle | ⭐⭐⭐⭐ | Passer sur Spring Boot 3.x LTS |
| **Stack Technique** | JDK 25 non standard | Compatibilité limitée | ⭐⭐⭐ | Passer sur JDK 21 LTS |
| **Architecture** | Mode API + Batch dans une seule app | Complexité, dépendances inutiles | ⭐⭐⭐ | Séparer en deux applications ou modules |
| **Performance** | Traitement séquentiel | Lent sur gros volumes | ⭐⭐⭐⭐ | Traitement parallèle + streaming |
| **Performance** | Limite de 400Mo par fichier | Contrainte forte | ⭐⭐⭐ | Augmenter la limite ou améliorer le streaming |
| **Architecture** | Couplage fort dans MainProcessing | Maintenance difficile | ⭐⭐⭐ | Découpler avec des événements |
| **Données** | DuckDB in-memory uniquement | Pas de persistance | ⭐⭐ | Ajouter persistance ou utiliser autre DB |
| **Jobs** | InMemoryJobStore sans persistance | Jobs perdus au restart | ⭐⭐⭐ | Ajouter persistance (DB, Redis) |
| **Tests** | Peu de tests d'intégration | Risque d'erreurs | ⭐⭐⭐ | Ajouter des tests d'intégration |
| **Sécurité** | Pas de tests de sécurité | Risques de vulnérabilités | ⭐⭐⭐⭐ | Ajouter des tests de sécurité |
| **Logging** | Messages non structurés | Analyse difficile | ⭐⭐ | Structured logging + correlation ID |
| **Code** | Endpoints dépréciés encore présents | Confusion, maintenance | ⭐⭐⭐ | Supprimer ou documenter la migration |

---

## Recommandations Prioritaires

### 🔴 Urgent (À faire avant la refonte)

1. **Passer sur Spring Boot 3.x LTS** avec JDK 21
   - Stabiliser la stack technique
   - Éviter les problèmes de compatibilité

2. **Supprimer ou Documenter les Endpoints Dépréciés**
   - Décider si on garde campaignId ou on migre complètement vers questionnaireModelId
   - Si on supprime : documenter la migration pour les clients
   - Si on garde : documenter le plan de dépréciation

3. **Ajouter des Tests de Sécurité**
   - Vérifier que les endpoints sensibles sont protégés
   - Tester les permissions et rôles

### 🟡 Important (À inclure dans la refonte)

4. **Séparer API et Batch**
   - Soit deux applications séparées
   - Soit un module dédié pour le batch

5. **Implémenter le Traitement Parallèle**
   - Utiliser CompletableFuture ou ExecutorService
   - Permettre la configuration du nombre de threads

6. **Ajouter la Persistance pour les Jobs**
   - Utiliser une base de données ou Redis
   - Permettre le suivi après restart

7. **Améliorer la Gestion des Erreurs**
   - Centraliser la gestion des erreurs
   - Standardiser les codes HTTP
   - Améliorer les messages d'erreur

### 🟢 Améliorations (Optionnel mais recommandé)

8. **Structured Logging**
   - JSON format pour les logs
   - Correlation ID

9. **Optimisation des Tests**
   - Ajouter des tests d'intégration
   - Ajouter des tests de performance
   - Refactoring des step definitions Cucumber

10. **Amélioration de la Documentation**
    - Documentation technique complète
    - Documentation des règles métier
    - Documentation des endpoints (Swagger/OpenAPI)

---

## Questions pour la Suite

1. **Quelle version de Spring Boot souhaitez-vous utiliser pour la cible ?**
   - Recommendation : Spring Boot 3.2.x ou 3.3.x avec JDK 21

2. **Comment souhaitez-vous gérer le mode batch ?**
   - Option A : Application séparée
   - Option B : Module séparé dans la même application
   - Option C : Intégrer Spring Shell
   - Option D : Garder l'approche actuelle mais améliorer

3. **Quelle stratégie pour les endpoints dépréciés (campaignId) ?**
   - Option A : Supprimer complètement
   - Option B : Garder mais marquer comme déprécié avec date de fin
   - Option C : Implémenter une redirection automatique vers questionnaireModelId

4. **Quel niveau de persistance pour les jobs ?**
   - Option A : Base de données relationnelle (PostgreSQL, etc.)
   - Option B : Redis (pour la performance)
   - Option C : Base de données embarquée (H2)
   - Option D : Pas de persistance (conserver InMemory)

5. **Quelle approche pour le traitement parallèle ?**
   - Option A : CompletableFuture (simple, intégré à Java)
   - Option B : ExecutorService avec thread pool
   - Option C : Spring @Async
   - Option D : Reactive Programming (WebFlux, R2DBC)

6. **Comment souhaitez-vous gérer le stockage des fichiers ?**
   - Option A : FileSystem uniquement
   - Option B : MinIO obligatoire
   - Option C : MinIO optionnel avec fallback FileSystem
   - Option D : Support multi-backend (FileSystem, MinIO, S3, etc.)
