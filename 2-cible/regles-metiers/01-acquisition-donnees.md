# Domaine ACQ — Acquisition des données Genesis

## Périmètre

Comment la cible récupère les données à traiter. Dans la cible, **l'unique source est Genesis**, interrogée par **`partitionId`**. Le mode « hors Genesis » (lecture de fichiers de réponses locaux) est **supprimé** (RG-ACQ-90).

**Modèle de regroupement Genesis** (précisions client 2026-07-09) : un **`partitionId`** (UUID) est un **regroupement d'`interrogationId`**. Une partition peut contenir **plusieurs `collectionInstrument`** (ex. un questionnaire web + un questionnaire téléphone). Réciproquement, un même `collectionInstrument` peut appartenir à **plusieurs partitions** (enquêtes mensuelles réutilisant le même questionnaire sur l'année). **L'export se fait par partition, en cumulant les modes.** Chaque partition porte un **`shortLabel`** utilisable pour le **nommage des fichiers** (RG-CSV-02). « Export par lot » = export **par partition**.

Rappel du modèle de données fourni par Genesis (existant, `SurveyUnitUpdateLatest.java:10-27`) : une interrogation = `interrogationId`, `usualSurveyUnitId`, `collectionInstrumentId`, `mode`, `isCapturedIndirectly`, `validationDate`, `questionnaireState`, une liste `collectedVariables` (champ JSON `variablesUpdate`) et une liste `externalVariables`. Chaque variable (`VariableModel`) porte `varId`, `value`, `scope` (nom de groupe ou `RACINE`), `iteration`.

---

## Règles de gestion

### RG-ACQ-01 — Source unique : Genesis par partitionId
La cible récupère les données d'une partition d'enquête via un appel Genesis paramétré par **`partitionId`**.
- **Source** : décision client (2026-07-08) ; `partitionId` déjà présent comme champ (vide) dans l'export JSON existant (`JsonWriterSequence.java`) et dans le schéma JSON du guide (`jsonSIexterne.md`).
- **Statut cible** : 🔵 Adapté — l'existant appelle Genesis par `campaignId` (déprécié) ou `questionnaireModelId` ; la cible bascule sur `partitionId`. Le champ `campaignId` du modèle est d'ailleurs déjà `@Deprecated(forRemoval=true)` (`SurveyUnitUpdateLatest.java:15-18`).

### RG-ACQ-02 — Récupération paginée / par lots
Les données sont récupérées par lots (paramètre `batchSize`) pour maîtriser la mémoire.
- **Source** : code (`batchSize` par défaut `1000`, présent sur tous les endpoints Genesis et en batch) ; guide (`enqgenesis.md`, `jsonSIexterne.md`).
- **Statut cible** : 🟢 Conservé (valeur par défaut et bornes à reconfirmer selon l'architecture cible).

### RG-ACQ-03 — Les variables arrivent déjà calculées de Genesis
Kraftwerk **ne calcule aucune variable**. Les variables collectées, externes **et calculées** sont fournies par Genesis, au même format, dans `collectedVariables`/`externalVariables`.
- **Source** : décision client (Q2 de `02-decisions-client-arbitrages.md`) ; code (`BuildBindingsSequenceGenesis.addVariablesToGroupInstance` ne fait qu'insérer les variables reçues, sans dérivation).
- **Statut cible** : 🔵 Adapté — supprime toute la logique de dérivation VTL (`CalculatedProcessing`) qui n'existait que dans le chemin fichiers (`UnimodalSequence`).
- **Conséquence** : voir `05` et le domaine variables ci-dessous — `FILTER_RESULT_*` et `*_MISSING` sont aussi des variables reçues de Genesis, pas calculées par Kraftwerk.

### RG-ACQ-04 — Répartition par mode de collecte
Une partition peut contenir plusieurs modes (`mode` de chaque interrogation, enum `Mode`). Les données sont regroupées par mode avant traitement unimodal, puis fusionnées (voir domaine `MULTI`).
- **Source** : code (`BuildBindingsSequenceGenesis.buildVtlBindings` filtre `surveyUnits` par `dataMode`).
- **Statut cible** : 🟢 Conservé.

### RG-ACQ-05 — Filtrage optionnel par mode
Si un `dataMode` est précisé à l'appel, seul ce mode est traité ; sinon tous les modes de la partition sont traités.
- **Source** : guide (`enqgenesis.md`, `jsonSIexterne.md` : « si le mode n'est pas précisé, Kraftwerk exporte tous les modes »).
- **Statut cible** : 🟢 Conservé.

### RG-ACQ-06 — Identifiants fixes propagés à tous les niveaux
Chaque interrogation porte deux identifiants toujours présents en sortie : `interrogationId` (technique, `Constants.ROOT_IDENTIFIER_NAME = "interrogationId"`) et `usualSurveyUnitId` (métier, `Constants.SURVEY_UNIT_IDENTIFIER_NAME = "usualSurveyUnitId"`). S'y ajoutent `validationDate` et `questionnaireState`.
- **Source** : code (`BuildBindingsSequenceGenesis` lignes ~60-67 ; `Constants.java:60-62`) ; guide (`enqgenesis.md` : table RACINE contient 2 identifiants + `questionnaireState` + `validationDate`).
- **Statut cible** : 🟢 Conservé.

### RG-ACQ-07 — Métadonnées de structure issues du DDI (+ Lunatic)
La structure des variables et des groupes/boucles est fournie par le modèle de métadonnées DDI (enrichi par le Lunatic), via la librairie externe `fr.insee.bpm`. Genesis fournit les *valeurs*, le DDI fournit la *structure* (types, groupes d'appartenance des variables).
- **Source** : code (`BuildBindingsSequenceGenesis` utilise `metadataModels.get(dataMode)` et `metadataModel.getVariables().getVariable(name).getGroupName()`).
- **Statut cible** : 🟢 Conservé (dépendance `fr.insee.bpm` à confirmer pour la cible).

### RG-ACQ-08 — Mode « Lunatic only » (sans DDI) — **différé**
Un traitement basé uniquement sur le modèle Lunatic (sans DDI) n'est **pas développé dans un premier temps**. Le **modèle Lunatic va évoluer** pour être **enrichi des informations aujourd'hui portées par le DDI** ; on **attend ce nouveau modèle**. Le moment venu, la solution cible devra être **plus fluide** que le mode sans-DDI actuel (qui ne garantit pas la correction du résultat).
- **Source** : décision client (2026-07-09, révision de la position du 2026-07-08) ; existant : `/main/genesis/by-questionnaire/lunatic-only`.
- **Statut cible** : ⏸️ **Différé** (attente du modèle Lunatic enrichi) — ni conservé tel quel, ni supprimé.

### RG-ACQ-10 — Données partielles : `questionnaireState`
Chaque interrogation porte un `questionnaireState` valant `FINISHED` (questionnaire validé) ou `STARTED` (réponse partielle / non validée). Cette variable est propagée telle quelle dans les exports (CSV/Parquet et JSON).
- **Source** : guide (abondamment : `entree.md`, `intgenesis.md`, `sortie.md`, `enqgenesis.md`, `jsonSIexterne.md`) ; code (`Constants.QUESTIONNAIRE_STATE_NAME`, `JsonWriterSequence.java:62`, propagé comme identifiant fixe par `InformationLevelsProcessing`).
- **Statut cible** : 🟢 Conservé — **mais non couvert par les tests actuels** (aucune occurrence de `STARTED`/`FINISHED` en Cucumber) : à couvrir explicitement (voir CL-ACQ-02).

### RG-ACQ-11 — États de variable optionnels (`addStates`)
Si le paramètre `addStates` est activé, chaque variable reçoit une variable d'état associée (suffixe `_STATE`, `Constants.VARIABLE_STATE_SUFFIX_NAME`) indiquant son statut (collectée / éditée / forcée). Les variables `FILTER_RESULT_*` et `*_MISSING` sont **exclues** de cet ajout d'état.
- **Source** : code (`BuildBindingsSequenceGenesis.addVariablesToGroupInstance` + `isFilterResultOrMissing`) ; guide (`jsonSIexterne.md` : variables `_STATE` optionnelles).
- **Statut cible** : 🟢 Conservé.

### RG-ACQ-12 — `isCapturedIndirectly`
Indicateur signalant qu'une donnée a été captée via un mode différent de celui prévu.
- **Source** : guide (`jsonSIexterne.md`) ; code (`SurveyUnitUpdateLatest.isCapturedIndirectly`).
- **Statut cible** : 🟢 Conservé (usage précis en sortie à préciser).

### RG-ACQ-90 — Suppression du mode « hors Genesis » et des traitements associés
La lecture de fichiers de réponses locaux est retirée, ainsi que tout ce qui en dépend :
- parseurs de fichiers locaux, dont **`XformsDataParser`** et les parseurs XML/JSON Lunatic locaux ;
- le **découpage de fichiers XML volumineux** (`SplitterService`, endpoint `/split/lunatic-xml`) ;
- la limite de **400 Mio** (`fr.insee.postcollecte.size-limit`), qui ne s'appliquait qu'à ce mode.

Sont également **exclus** de la cible (précisions client 2026-07-09) : les **paradonnées**, les **modules TCM en VTL** et le **VTL utilisateur** (scripts métier `.vtl`), et l'**archivage des fichiers d'entrée** (RG-ANO-22). Seules les **données de reporting** pourront, dans une **version avancée**, être lues depuis des fichiers à part (RG-REP-02).
- **Source** : décisions client (point C de `04-comparaison-code-tests-guide.md` ; précisions 2026-07-09).
- **Statut cible** : 🔴 Supprimé.

---

## Cas nominaux

- **CN-ACQ-01** — Partition mono-mode, questionnaires tous `FINISHED` : Genesis renvoie N interrogations d'un seul mode, chacune avec ses variables collectées/externes/calculées ; Kraftwerk produit les sorties sans réconciliation multimode (un seul mode).
- **CN-ACQ-02** — Partition multimode : Genesis renvoie des interrogations de 2+ modes ; traitement unimodal par mode puis fusion (domaine `MULTI`).
- **CN-ACQ-03** — Extraction avec `dataMode` précisé : seul le mode demandé est extrait et traité.

## Edge cases

- **CL-ACQ-01** — **Partition vide** : Genesis ne renvoie aucune interrogation. Comportement attendu à définir (aujourd'hui l'export JSON existant produit un « 204 No Content » ; cf. `MainService.jsonExtraction`). → à formaliser : sortie vide vs code de retour dédié.
- **CL-ACQ-02** — **Interrogation partielle (`STARTED`)** : une interrogation non validée doit apparaître dans l'export avec `questionnaireState=STARTED`, **si et seulement si** elle a été transmise à Genesis (le guide insiste : si la donnée partielle n'a pas été poussée dans Genesis, l'unité n'apparaît pas). Kraftwerk ne décide pas de sa présence, il reflète ce que Genesis renvoie.
- **CL-ACQ-03** — **Interrogation sans `validationDate`** : le code gère explicitement le cas `null` (n'écrit pas de date). À conserver.
- **CL-ACQ-04** — **Variable d'un groupe absent des métadonnées** : une variable reçue de Genesis dont le `groupName` n'existe pas dans le DDI. Le code actuel (`addGroupVariables`) teste `hasVariable(...)` avant insertion ; le comportement pour une variable inconnue (ignorée silencieusement ?) est à préciser et à tracer comme erreur/avertissement dans la cible.
- **CL-ACQ-05** — **Plusieurs valeurs pour une même interrogation (reprise / soumissions multiples)** : le guide précise qu'on retient la **valeur la plus récente** enregistrée dans Genesis (`sortie.md`). Genesis fournissant déjà « latest » (`SurveyUnitUpdateLatest`), cette règle est portée en amont — à confirmer que Kraftwerk n'a rien à dédupliquer côté valeurs.
