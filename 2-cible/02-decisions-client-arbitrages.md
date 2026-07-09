# Kraftwerk - Réponses Client et Mise à Jour des Spécifications

## Date : 2026-07-08
**Statut** : Réponses reçues - Mise à jour des spécifications

---

## 📝 **Réponses du Client aux Questions Ouvertes**

### ❓ Question 1 : Module VTL Externe
**Réponse Client** : 
> "Oui il existe, on fera un POC pour voir comment on l'intègre à la chaîne mais **ça ne concerne plus le module Kraftwerk**"

**Interprétation** :
- ✅ Module VTL externe existe et sera utilisé
- ✅ **Kraftwerk n'aura AUCUNE responsabilité** concernant VTL
- ✅ Un POC séparé sera fait pour l'intégration
- ✅ **Impact** : Kraftwerk peut être simplifié (pas de code VTL, pas de dépendance Trevas)

**Décision Validée** :
- Suppression complète du module VTL dans Kraftwerk
- Kraftwerk **consomme uniquement** les variables calculées déjà présentes dans les données Genesis
- Pas besoin de gérer le calcul, la validation, ou la transformation VTL

---

### ❓ Question 2 : Genesis et Variables Calculées
**Réponse Client** : 
> "Oui, elles auront **vraisemblablement le même format** que les variables collected. Les interactions de Genesis avec ses sources et ses conditions de stockage **n'ont pas à intervenir ici**. Tu peux partir du principe que **Genesis les fournira avec les collected et external**, au même format."

**Interprétation** :
- ✅ Genesis **fournit** les variables calculées
- ✅ Format **identique** aux variables collected et external
- ✅ **Kraftwerk n'a pas à connaître** les détails d'implémentation Genesis
- ✅ **Hypothèse validée** : Les données reçues de Genesis contiennent déjà TOUTES les variables (collected, external, calculées)

**Décision Validée** :
- Kraftwerk **ne calcule rien** - il consomme uniquement les données
- **Pas de distinction** entre variables collected, external, ou calculées dans le traitement
- **Format unifié** : Toutes les variables sont au même format JSON
- **Simplification majeure** : Suppression de toute la logique de calcul

---

### ❓ Question 3 : Scripts "Aval"
**Réponse Client** : 
> "Les scripts sont **essentiellement utilisés aujourd'hui pour anonymiser** (enlever des colonnes). J'imaginais qu'une **requête SQL dans la base DuckDB serait plus pertinente**. Mais peut-être d'autre solution (type **paramètre d'exécution qui liste les variables à supprimer**) ou autre peut aussi convenir"

**Interprétation** :
- ✅ Usage principal : **Anonymisation** (suppression de colonnes)
- ✅ Solution préférée : **Requête SQL dans DuckDB**
- ✅ Alternative acceptée : **Paramètre d'exécution** (liste des variables à supprimer)
- ✅ Flexibilité : Ouvert à d'autres solutions

**Décision Validée** :
- **Option A** : Implémenter un **filtre SQL** dans le pipeline de traitement
  ```sql
  -- Exemple : SELECT * EXCLUDE (col1, col2, col3) FROM data
  -- Ou : SELECT col1, col2, col4, col5 FROM data
  ```
- **Option B** : Ajouter un **paramètre `excludedVariables`** dans la requête d'export
  ```json
  {
    "partitionId": "PART-123",
    "excludedVariables": ["NOM", "PRENOM", "ADRESSE"],
    "outputFormat": "CSV"
  }
  ```
- **Option C** : Combiner les deux (SQL par défaut + paramètre pour les cas simples)

**Recommandation** : **Option C** (Combinaison)
- Paramètre simple pour les cas standards
- SQL pour les cas complexes
- Conservation de la flexibilité

---

### ❓ Question 4 : Reportings Sabiane
**Réponse Client** : 
> "Il faudra du reporting. Pour le moment le **quoi/comment n'est pas encore clair**. Il faut en tout cas **laisser la possibilité de faire évoluer Kraftwerk** pour lui fournir d'autre données (dont les reporting ou la codification) et d'**appliquer des traitements standards** sur ces données (calcul de temps par exemple)"

**Interprétation** :
- ✅ **Besoin de reporting confirmé** mais spécifications non finalisées
- ✅ **Évolutivité requise** : Kraftwerk doit pouvoir traiter de nouveaux types de données
- ✅ **Traitements standards** : Calcul de temps, agrégations, etc.
- ✅ **Flexibilité** : Architecture doit permettre l'ajout de nouveaux types de données

**Décision Validée** :
- **Architecture extensible** : Conçue pour accepter de nouveaux types de données
- **Pipeline générique** : Pas spécifique à un type de données
- **Traitements standards** : Implémenter une bibliothèque de transformations communes
  - Calcul de temps (durée, dates)
  - Agrégations (count, sum, avg)
  - Filtres génériques
- **Plugin system** : Pour ajouter de nouveaux traitements sans modifier le core

**Exemple d'Extension Future** :
```java
// Ajout d'un nouveau type de données
public class ReportingDataInputAdapter implements DataInputPort {
    @Override
    public DataFrame readData(ProcessingContext context) {
        // Lecture des données de reporting
        // Application des traitements standards (calcul de temps, etc.)
    }
}
```

---

### ❓ Question 5 : Premières Fonctionnalités et Calendrier
**Réponse Client** : 
> "Les **premières fonctionnalités attendues** devront permettre un **export csv/parquet simple** (sans TCM et donc sans lien 2à2). **Les boucles devront être gérées**, et **l'appel à Genesis se fera par partitionId**. Le calendrier sera défini à l'issue de la conception."

**Interprétation** :
- ✅ **Première fonctionnalité** : Export CSV/Parquet **simple**
- ❌ **Exclu** : TCM (Traitement de Cohérence Multimode) et liens 2à2
- ✅ **Inclus** : Gestion des **boucles** (obligatoire dès le début)
- ✅ **Nouvelle information** : Appel Genesis par **partitionId** (et non campaignId ou questionnaireModelId)
- ⏳ **Calendrier** : À définir après la conception

**Décision Validée - Mise à Jour Majeure** :
- **partitionId** devient le **paramètre principal** pour les appels Genesis
- **Pas de TCM** dans la première version (plus simple)
- **Gestion des boucles** obligatoire dès le début
- **Export simple** : CSV/Parquet sans transformation complexe

**Impact sur l'Architecture** :
- Modification du **GenesisClient** : Utilisation de `partitionId` au lieu de `campaignId`/`questionnaireModelId`
- Simplification du **pipeline** : Pas de réconciliation TCM à gérer
- **Focus** : Export simple + boucles

---

---

## 🔄 **Mise à Jour des Spécifications**

### **Changements Majeurs**

| Élément | Ancienne Version | Nouvelle Version (Validée) |
|---------|------------------|----------------------------|
| **Paramètre principal** | campaignId / questionnaireModelId | **partitionId** |
| **TCM (Réconciliation)** | À implémenter | **Exclu de la V1** |
| **Liens 2à2** | À gérer | **Exclu de la V1** |
| **Variables calculées** | À calculer par Kraftwerk | **Fournies par Genesis** (même format) |
| **Scripts aval** | À étudier | **Anonymisation** (SQL ou paramètre) |
| **Reportings** | À re-questionner | **À prévoir** (architecture extensible) |
| **VTL** | À supprimer | **Déjà externalisé** (hors scope Kraftwerk) |

---

### **Fonctionnalités Prioritaires (V1)**

#### **Must Have (Livraison 1)**

| ID | Fonctionnalité | Description | Complexité |
|----|---------------|-------------|------------|
| FH-001 | Export CSV/Parquet simple | Export basique sans TCM/liens 2à2 | ⭐⭐ |
| FH-002 | Gestion des boucles | Detection et export des structures en boucle | ⭐⭐⭐ |
| FH-003 | Appel Genesis par partitionId | Récupération des données via partitionId | ⭐⭐ |
| FH-004 | Anonymisation | Suppression de colonnes (SQL ou paramètre) | ⭐⭐ |
| FH-005 | Formats CSV/Parquet | Génération des deux formats | ⭐⭐ |
| FH-006 | Logging structuré | JSON + Correlation ID | ⭐ |
| FH-007 | Métriques | Monitoring via Micrometer | ⭐ |

#### **Nice to Have (Livraison 2)**

| ID | Fonctionnalité | Description | Complexité |
|----|---------------|-------------|------------|
| FH-008 | Export JSON | Génération de fichiers JSON locaux | ⭐⭐ |
| FH-009 | TCM (Réconciliation) | Réconciliation multimode | ⭐⭐⭐⭐ |
| FH-010 | Liens 2à2 | Gestion des liens entre tables | ⭐⭐⭐⭐ |
| FH-011 | Persistance Jobs | Spring Batch + Base de données | ⭐⭐⭐ |
| FH-012 | Reportings | Traitement des données de reporting | ⭐⭐⭐ |
| FH-013 | Codification | Intégration avec les données codifiées | ⭐⭐⭐⭐ |

---

## 🏗️ **Architecture Mise à Jour**

### **Pipeline de Traitement Simplifié (V1)**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE V1 (Simplifié)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. INPUT : Appel Genesis par partitionId                         │
│     ├─ Récupération des données (collected + external + calculées) │
│     ├─ Format unique : JSON standard                                │
│     └─ Streaming (pagination)                                       │
│                                                                     │
│  2. PARSING : Conversion JSON → DataFrame                           │
│     ├─ Parsing des données principales                              │
│     └─ Détection des boucles                                        │
│                                                                     │
│  3. ANONYMISATION (Optionnel)                                      │
│     ├─ Application du filtre (SQL ou liste d'exclusion)            │
│     └─ Paramétrable par appel                                       │
│                                                                     │
│  4. AGRÉGATION : Gestion des boucles                                │
│     ├─ Création de DataFrames par niveau (racine + boucles)        │
│     └─ Conservation de la hiérarchie                                │
│                                                                     │
│  5. OUTPUT : Génération CSV/Parquet                                 │
│     ├─ Écriture streaming par DataFrame                             │
│     └─ Formatage selon le type de sortie                            │
│                                                                     │
│  6. STORAGE : Sauvegarde des fichiers                              │
│     ├─ Local (cache)                                                │
│     └─ MinIO (optionnel)                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### **Modifications Architecturales**

#### **1. GenesisClient**
```java
// Avant : Utilisait campaignId ou questionnaireModelId
public interface GenesisClient {
    DataFrame getDataByCampaignId(String campaignId);
    DataFrame getDataByQuestionnaireModelId(String questionnaireModelId);
}

// Après : Utilise partitionId uniquement
public interface GenesisClient {
    Flux<Data> getDataByPartitionId(String partitionId);
    Flux<Data> getDataByPartitionId(String partitionId, Instant sinceDate);
    Flux<Data> getDataByPartitionId(String partitionId, Instant sinceDate, int batchSize);
}
```

#### **2. ExportRequest DTO**
```java
// Nouveau DTO pour la V1
public record ExportRequest(
    String partitionId,           // ⭐ NOUVEAU : Paramètre principal
    Instant sinceDate,           // Optionnel : date de début
    Instant untilDate,           // Optionnel : date de fin
    Set<String> excludedVariables, // Optionnel : pour anonymisation
    boolean includeRoot,          // Optionnel : inclure le niveau racine
    boolean includeLoops,         // Optionnel : inclure les boucles
    OutputFormat format,         // CSV, PARQUET, JSON
    boolean archive,             // Optionnel : archiver les résultats
    boolean encrypt               // Optionnel : chiffrer les résultats
) {}
```

#### **3. ProcessingContext**
```java
// Contexte simplifié pour la V1
public record ProcessingContext(
    String partitionId,
    Set<String> excludedVariables,
    OutputFormat outputFormat,
    Path outputDirectory,
    boolean includeRoot,
    boolean includeLoops
) {}
```

---

## 📊 **Impact sur les Modules**

### **kraftwerk-core**

#### **Domain Layer**
```java
// Entités simplifiées
public record Variable(
    String name,
    String value,
    String type,
    Metadata metadata
) {}

public record DataFrame(
    String partitionId,
    String interrogationId,
    Map<String, Variable> variables,
    List<Loop> loops
) {}

public record Loop(
    String name,
    int iteration,
    Map<String, Variable> variables
) {}
```

#### **Port Layer**
```java
// Ports pour la V1
public interface DataInputPort {
    Flux<DataFrame> readData(ProcessingContext context);
}

public interface DataProcessingPort {
    Flux<DataFrame> process(DataFrame input, ProcessingContext context);
}

public interface DataOutputPort {
    Mono<Void> write(DataFrame data, ProcessingContext context);
}

public interface AnonymizationPort {
    DataFrame anonymize(DataFrame data, Set<String> excludedVariables);
}

public interface LoopDetectionPort {
    List<Loop> detectLoops(DataFrame data);
}
```

#### **Processing Pipeline**
```java
@Service
public class ExportProcessingPipeline {
    private final DataInputPort dataInputPort;
    private final AnonymizationPort anonymizationPort;
    private final LoopDetectionPort loopDetectionPort;
    private final DataOutputPort dataOutputPort;
    
    public Mono<ExportResult> process(ProcessingContext context) {
        return dataInputPort.readData(context)
            .doOnNext(data -> {
                // Anonymisation si variables à exclure
                if (context.excludedVariables() != null && !context.excludedVariables().isEmpty()) {
                    anonymize(data, context.excludedVariables());
                }
            })
            .flatMap(data -> {
                // Détection des boucles
                List<Loop> loops = loopDetectionPort.detectLoops(data);
                return processWithLoops(data, loops, context);
            })
            .flatMap(dataOutputPort::write)
            .collectList()
            .map(results -> new ExportResult(
                results.size(),
                context.format(),
                Instant.now()
            ));
    }
}
```

---

### **kraftwerk-api**

#### **Nouveaux Endpoints (V1)**
```java
@RestController
@RequestMapping("/api/v1/export")
public class ExportController {
    private final ExportService exportService;
    
    // Export simple par partitionId
    @PostMapping("/simple")
    @Operation(summary = "Export simple par partitionId")
    public Mono<ExportResult> exportSimple(
        @RequestBody ExportRequest request
    ) {
        return exportService.export(request);
    }
    
    // Export avec anonymisation
    @PostMapping("/anonymized")
    @Operation(summary = "Export avec anonymisation")
    public Mono<ExportResult> exportAnonymized(
        @RequestBody ExportRequest request
    ) {
        if (request.excludedVariables() == null || request.excludedVariables().isEmpty()) {
            throw new KraftwerkException(400, "excludedVariables is required for anonymized export");
        }
        return exportService.export(request);
    }
    
    // Export asynchrone
    @PostMapping("/async")
    @Operation(summary = "Export asynchrone")
    public Mono<JobAcceptedResponse> exportAsync(
        @RequestBody ExportRequest request
    ) {
        String jobId = UUID.randomUUID().toString();
        exportService.exportAsync(jobId, request);
        return Mono.just(new JobAcceptedResponse(jobId));
    }
    
    // Statut d'un job
    @GetMapping("/jobs/{jobId}")
    @Operation(summary = "Statut d'un job d'export")
    public Mono<JobStatus> getJobStatus(@PathVariable String jobId) {
        return exportService.getJobStatus(jobId);
    }
}
```

---

## 🎯 **Mise à Jour des Règles Métiers**

### **Règles Confirmées**

#### **1. Format des Données**
- ✅ Toutes les variables (collected, external, calculées) ont le **même format JSON**
- ✅ Pas de distinction à faire dans Kraftwerk entre les types de variables
- ✅ Structure : `{name: string, value: any, type: string, metadata: {...}}`

#### **2. Gestion des Boucles**
- ✅ **Obligatoire dès la V1**
- ✅ Détection automatique des structures en boucle
- ✅ Génération de fichiers séparés par boucle
- ✅ Conservation de la hiérarchie (racine + boucles)

#### **3. Anonymisation**
- ✅ **Fonctionnalité obligatoire** (remplace les scripts aval pour ce besoin)
- ✅ Deux implémentations possibles :
  - **Filtre SQL** : `SELECT * EXCLUDE (col1, col2) FROM data`
  - **Paramètre** : Liste des variables à exclure
- ✅ **Recommandation** : Implémenter les deux pour flexibilité

#### **4. Appel Genesis**
- ✅ **Par partitionId uniquement** (plus de campaignId/questionnaireModelId)
- ✅ **Pagination obligatoire** pour éviter les gros volumes en mémoire
- ✅ **Streaming** : Flux continu vers le pipeline
- ✅ **Format de retour** : Données complètes (collected + external + calculées)

---

## 🗺️ **Nouvelle Roadmap (V1)**

### **Phase 0 : Conception (1-2 semaines)**
- [x] **Analyse de l'existant** (Terminé)
- [x] **Réponses client reçues** (Terminé)
- [ ] **Mise à jour des spécifications** (En cours)
- [ ] **Validation des choix architecturaux**
- [ ] **Estimation détaillée** (temps, ressources)

### **Phase 1 : Fondations (Sprint 1)**
- [ ] Créer la structure modulaire (parent/core/api/batch)
- [ ] Configurer Spring Boot 4.1.0 + JDK 25
- [ ] Implémenter le pattern Ports & Adapters
- [ ] **Nouveau** : Configurer GenesisClient avec partitionId
- [ ] Configurer le pipeline CI/CD
- [ ] Supprimer toutes les dépendances VTL/Trevas

### **Phase 2 : Pipeline de Base (Sprint 2-3)**
- [ ] Implémenter **DataInputPort** (Genesis par partitionId)
- [ ] Implémenter **DataFrame** et modèles associés
- [ ] Implémenter **LoopDetectionPort**
- [ ] Implémenter **DataOutputPort** (CSV/Parquet)
- [ ] Ajouter **AnonymizationPort** (SQL + paramètre)
- [ ] Ajouter les tests unitaires

### **Phase 3 : Fonctionnalités V1 (Sprint 4-5)**
- [ ] Implémenter **ExportController** (endpoints V1)
- [ ] Implémenter **ExportService**
- [ ] Implémenter **ProcessingPipeline**
- [ ] Ajouter la gestion des erreurs
- [ ] Ajouter le logging structuré
- [ ] Ajouter les métriques
- [ ] Tests d'intégration

### **Phase 4 : Optimisation et Validation (Sprint 6)**
- [ ] Optimiser les performances (benchmark)
- [ ] Valider avec des données réelles
- [ ] Documenter complètement
- [ ] Préparer la migration

---

## 📈 **Nouveaux Objectifs (V1)**

| Métrique | Cible V1 | Mesure |
|----------|----------|--------|
| **Fonctionnalités** | Export simple + boucles + anonymisation | 100% |
| **Performance** | < 1 Go RAM pour 1800 interrogations | Benchmark |
| **Temps de traitement** | < 5 min pour 1800 interrogations | Benchmark |
| **Tests** | Couverture > 80% | SonarQube |
| **Documentation** | Complète | 100% |
| **Maintenabilité** | Architecture modulaire | Évaluation |

---

## 🔄 **Changements par Rapport à l'Ancienne Version**

### **Ce qui est Simplifié** ✅
1. **Pas de VTL** dans Kraftwerk (externalisé)
2. **Pas de TCM** dans la V1 (réconciliation multimode)
3. **Pas de liens 2à2** dans la V1
4. **Un seul paramètre** : partitionId (au lieu de campaignId/questionnaireModelId)
5. **Variables calculées** fournies par Genesis (pas de calcul à faire)

### **Ce qui est Conservé** ✅
1. **Formats de sortie** : CSV, Parquet
2. **Gestion des boucles** (obligatoire)
3. **Structure des données** (niveaux d'information)
4. **Intégration Genesis** (mais simplifiée)
5. **Architecture modulaire**

### **Ce qui est Ajouté** ➕
1. **Anonymisation** (SQL + paramètre)
2. **Streaming** (pagination Genesis)
3. **Parallel processing** (optimisation performance)
4. **Logging structuré**
5. **Metrics**

### **Ce qui est Reporté** ⏳
1. **TCM (Réconciliation)** → V2
2. **Liens 2à2** → V2
3. **Reportings** → V2 (mais architecture préparée)
4. **Codification** → V2
5. **Persistance Jobs** → V2
6. **Export JSON local** → V2

---

## 💡 **Recommandations Spécifiques**

### **1. Implémentation de l'Anonymisation**

**Option Recommandée : Combinaison SQL + Paramètre**

```java
@Component
public class AnonymizationService implements AnonymizationPort {
    
    // Méthode 1 : Par liste d'exclusion
    @Override
    public DataFrame anonymize(DataFrame data, Set<String> excludedVariables) {
        Map<String, Variable> filteredVariables = data.variables().entrySet().stream()
            .filter(entry -> !excludedVariables.contains(entry.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
        
        return new DataFrame(
            data.partitionId(),
            data.interrogationId(),
            filteredVariables,
            data.loops()
        );
    }
    
    // Méthode 2 : Par requête SQL (via DuckDB)
    public DataFrame anonymizeBySql(DataFrame data, String sqlQuery) {
        // Exécuter la requête SQL sur le DataFrame
        // Retourner le DataFrame filtré
    }
}
```

**Endpoint API** :
```java
@PostMapping("/export/anonymized")
public Mono<ExportResult> exportAnonymized(
    @RequestParam String partitionId,
    @RequestBody AnonymizationConfig config
) {
    // config contient : excludedVariables OU sqlQuery
}

public record AnonymizationConfig(
    Set<String> excludedVariables,  // Option 1
    String sqlQuery,                 // Option 2
    OutputFormat format
) {}
```

### **2. Gestion des Boucles**

**Implémentation recommandée** :

```java
@Component
public class LoopDetectionService implements LoopDetectionPort {
    
    @Override
    public List<Loop> detectLoops(DataFrame data) {
        // Détection automatique des structures en boucle
        // dans les variables
        
        Map<String, List<Variable>> loopVariables = data.variables().entrySet().stream()
            .filter(entry -> entry.getKey().contains("_"))  // Convention : loop_iteration
            .collect(Collectors.groupingBy(
                entry -> entry.getKey().split("_")[0],  // Nom de la boucle
                Collectors.toList()
            ));
        
        return loopVariables.entrySet().stream()
            .map(entry -> {
                String loopName = entry.getKey();
                List<Variable> variables = entry.getValue();
                
                // Extraire les itérations
                Map<Integer, Map<String, Variable>> iterations = variables.stream()
                    .collect(Collectors.groupingBy(
                        var -> extractIteration(var.name()),
                        Collectors.toMap(
                            var -> var.name().split("_")[1],  // Nom sans itération
                            Function.identity()
                        )
                    ));
                
                return new Loop(loopName, iterations);
            })
            .collect(Collectors.toList());
    }
    
    private int extractIteration(String variableName) {
        // Extraire le numéro d'itération du nom de variable
        String[] parts = variableName.split("_");
        return Integer.parseInt(parts[parts.length - 1]);
    }
}
```

**Convention de nommage** :
- `BOUCLE_PRENOMS_1_NOM` : Boucle BOUCLE_PRENOMS, itération 1, variable NOM
- `BOUCLE_PRENOMS_2_NOM` : Boucle BOUCLE_PRENOMS, itération 2, variable NOM

### **3. Appel Genesis par partitionId**

**Implémentation recommandée** :

```java
@Service
public class GenesisClient {
    private final WebClient webClient;
    private final int defaultPageSize = 1000;
    
    public Flux<Data> getDataByPartitionId(String partitionId) {
        return getDataByPartitionId(partitionId, null, null, defaultPageSize);
    }
    
    public Flux<Data> getDataByPartitionId(String partitionId, 
                                         Instant sinceDate, 
                                         Instant untilDate,
                                         int pageSize) {
        return Flux.generate(
            () -> new GenesisPageRequest(partitionId, sinceDate, untilDate, 0, pageSize),
            (request, sink) -> {
                try {
                    GenesisPageResponse response = webClient.get()
                        .uri("/api/partitions/{partitionId}/data", partitionId)
                        .param("sinceDate", sinceDate != null ? sinceDate.toString() : null)
                        .param("untilDate", untilDate != null ? untilDate.toString() : null)
                        .param("page", request.page())
                        .param("size", request.pageSize())
                        .retrieve()
                        .bodyToMono(GenesisPageResponse.class)
                        .block();
                    
                    if (response == null || response.data().isEmpty()) {
                        sink.complete();
                        return request;
                    }
                    
                    response.data().forEach(sink::next);
                    
                    if (response.isLast()) {
                        sink.complete();
                    }
                    
                    return new GenesisPageRequest(
                        partitionId, 
                        sinceDate, 
                        untilDate,
                        request.page() + 1,
                        pageSize
                    );
                } catch (Exception e) {
                    sink.error(e);
                    return request; // Ne pas continuer
                }
            }
        );
    }
    
    private record GenesisPageRequest(
        String partitionId,
        Instant sinceDate,
        Instant untilDate,
        int page,
        int pageSize
    ) {}
}
```

---

## 📝 **Spécifications Techniques Mises à Jour**

### **Requirements Non Fonctionnels (V1)**

| ID | Requirement | Critère de Succès |
|----|-------------|-------------------|
| NF-001 | **Consommation mémoire** | < 1 Go RAM pour 1800 interrogations |
| NF-002 | **Temps de traitement** | < 5 min pour 1800 interrogations |
| NF-003 | **Format CSV** | Fichier CSV valide, avec en-têtes |
| NF-004 | **Format Parquet** | Fichier Parquet valide, compatible Spark |
| NF-005 | **Gestion des boucles** | 1 fichier par boucle + 1 fichier racine |
| NF-006 | **Anonymisation** | Variables exclues absentes des fichiers |
| NF-007 | **Logging** | Logs JSON avec correlation ID |
| NF-008 | **Metrics** | Métriques Prometheus disponibles |
| NF-009 | **Gestion des erreurs** | Codes HTTP appropriés + messages clairs |
| NF-010 | **Tests** | Couverture > 80% |

---

## 🎯 **Prochaines Étapes Immédiates**

### **Pour le Client**

1. **✅ Valider cette mise à jour** des spécifications
2. **⏳ Confirmer le calendrier** (après la conception)
3. **⏳ Nommer un responsable** pour la validation des livrables
4. **⏳ Fournir un exemple** de partitionId et de données Genesis (pour les tests)

### **Pour Moi (Assistant)**

Je peux **immédiatement** :

1. **Générer le squelette du projet** avec :
   - Structure Maven (parent/core/api/batch)
   - Configuration Spring Boot 4.1.0 + JDK 25
   - GenesisClient avec partitionId
   - Ports & Adapters pour la V1
   - Modèles DataFrame, Variable, Loop

2. **Produire les spécifications détaillées** :
   - Spécifications techniques (diagrammes, séquences)
   - Spécifications fonctionnelles (user stories V1)
   - Règles métiers formalisées
   - Cas de test pour la V1

3. **Créer des exemples de code** :
   - Exemple d'implémentation du pipeline
   - Exemple de DataInputPort (Genesis)
   - Exemple de LoopDetectionPort
   - Exemple de AnonymizationPort

---

## 🚀 **Prêt à Démarrer !**

**Avec vos réponses, j'ai maintenant toutes les informations nécessaires pour :**

1. ✅ **Créer le squelette du projet Kraftwerk V1**
2. ✅ **Implémenter le pipeline de traitement** (partitionId → CSV/Parquet + boucles + anonymisation)
3. ✅ **Produire une documentation complète**
4. ✅ **Préparer les tests**

**Prochaine étape** : 
- **Soit** : Vous validez cette mise à jour et je commence la génération du code
- **Soit** : Vous avez des ajustements à apporter

**Temps estimé pour le squelette** : 1-2 jours  
**Temps estimé pour la V1 complète** : 4-6 semaines (2-3 sprints)

---

## Addendum de confirmation (2026-07-08, après revue de l'existant vérifiée sur le code)

Après la production d'une description fidèle et vérifiée de l'existant (`01-architecture-globale.md`, `02-fonctionnalites-principales.md`, `05-anomalies-code.md`), les points suivants ont été reconfirmés/précisés par le client :

- **`partitionId` est confirmé comme paramètre d'appel Genesis pour la cible** — à retenir comme base de travail dès maintenant pour la suite (comparaison code/tests/guide, règles métier, spécifications cible).
- **Correction par rapport à la table de la section « Changements Majeurs » ci-dessus** : le traitement des **liens 2à2** (matrice de liens familiaux, aujourd'hui codée en dur pour `BOUCLE_PRENOMS` dans `LunaticXmlDataParser.manageTcmLiens`, cf. `02-fonctionnalites-principales.md` §4 et `05-anomalies-code.md` point 10) est **conservé dans la cible**, contrairement à ce qui était noté plus haut ("Exclu de la V1"). Seul le TCM au sens large (réconciliation multimode via VTL) reste hors périmètre proche.
- **VTL ne doit plus être mentionné/porté par Kraftwerk** : confirmation de l'externalisation complète de VTL (cf. Question 1 plus haut) — la cible ne doit pas réintroduire de terminologie ou de dépendance VTL/Trevas dans son architecture ou sa documentation.

Ces trois points sont à traiter comme acquis pour la suite des travaux (comparaison code/tests/guide, extraction des règles métier, spécifications cible), sauf nouvel élément contraire.
