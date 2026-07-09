# Domaine MULTI — Réconciliation / fusion multimode automatique

## Périmètre

Quand une partition contient des interrogations issues de plusieurs modes de collecte (WEB, TEL, F2F, PAPER…), la cible **fusionne automatiquement** les données des différents modes en un jeu de données unique. Fonctionnalité **conservée** (décision client, contre l'affirmation du guide selon laquelle ce ne serait « pas encore permis à ce jour »).

> ⚠️ **Point de conception.** La logique existe dans le code (`ReconciliationProcessing.java`, ~300 lignes) mais est **écrite en VTL** (génération de scripts `union`/`left_join` exécutés par Trevas). VTL étant retiré de la cible, cette logique doit être **réexprimée sans VTL** (SQL DuckDB probable). Voir RG-MULTI-90.

---

## Règles de gestion

### RG-MULTI-01 — Fusion automatique des modes
Les jeux de données unimodaux (un par mode) sont fusionnés automatiquement en un jeu de données multimode unique (nommé `MULTIMODE` dans l'existant, `Constants.MULTIMODE_DATASET_NAME`).
- **Source** : code (`ReconciliationProcessing`, `MultimodalSequence`) ; décision client (point B de `04-comparaison-code-tests-guide.md`).
- **Statut cible** : 🔵 Adapté (réécriture sans VTL).

### RG-MULTI-02 — Variable d'identification du mode
La fusion ajoute une variable identifiant le mode d'origine de chaque ligne, nommée `MODE_KRAFTWERK` (`DataProcessing.MODE_VARIABLE_NAME = "MODE_KRAFTWERK"`), dont les valeurs sont les codes de mode (`data_mode`).
- **Source** : code (`ReconciliationProcessing.createModeIdentifier`, `DataProcessing.java:28`) ; guide (`vtl.md` : « une variable `MODE_KRAFTWERK` est créée pour identifier le mode d'origine »).
- **Statut cible** : 🟢 Conservé.

### RG-MULTI-03 — Union des variables communes et absentes
La fusion réalise l'union des colonnes de tous les modes : une variable présente dans un seul mode se retrouve dans le jeu multimode (valeur absente pour les lignes des autres modes). Les variables communes sont alignées.
- **Source** : code (`ReconciliationProcessing` calcule les mesures communes/absentes entre modes).
- **Statut cible** : 🔵 Adapté.

### RG-MULTI-04 — Divergence de type entre modes = cas d'exception, pas d'harmonisation silencieuse
En principe une même variable porte le **même type dans tous les modes** (les DDI sont normalement identiques). Il **ne devrait donc pas** y avoir de divergence de type. Si une divergence survient malgré tout, la cible **ne doit pas harmoniser silencieusement** : elle doit produire une **erreur lisible et exploitable par l'utilisateur** ; en **solution d'attente**, on peut **créer deux colonnes distinctes** (une par type) le temps de corriger la source.
- **Source** : décision client (2026-07-09) — révise le comportement de l'existant (`ReconciliationProcessing` convertissait entier→`number`).
- **Statut cible** : 🔵 Adapté — remplacer l'harmonisation implicite par une **gestion d'erreur explicite** (+ double colonne en repli).

### RG-MULTI-05 — Point d'extension « transformation multimode »
Après la fusion, un point de personnalisation permettait d'appliquer des traitements supplémentaires sur le jeu multimode (gestion de doublons inter-modes, etc.).
- **Source** : guide (`vtl.md` : script `gestion_doublons.vtl`, étape « traitements multimode » qui « ne contient aucun traitement automatique aujourd'hui ») ; code (`MultimodeTransformations` est un simple hook sans instruction automatique).
- **Statut cible** : 🟠 À concevoir — VTL étant retiré, décider comment (et si) offrir un point d'extension utilisateur pour la gestion des doublons inter-modes.

### RG-MULTI-90 — Réécriture sans VTL (validée client)
Toute la mécanique de réconciliation (union, jointure, ajout de `MODE_KRAFTWERK`, harmonisation de types) doit être réimplémentée sans moteur VTL/Trevas. Il en va de même pour la répartition des variables en tables (RG-STR-01) : ces deux traitements, aujourd'hui portés par VTL, sont à réécrire.
- **Source** : décision client (2026-07-08 : réécriture de la réconciliation **et** de la répartition en tables validée ; plus de VTL).
- **Statut cible** : 🟠 À concevoir. Piste privilégiée et acceptée : opérations SQL dans DuckDB (déjà utilisé pour l'écriture des sorties).

---

## Cas nominaux

- **CN-MULTI-01** — Deux modes (WEB + TEL) avec le même questionnaire : jeu multimode contenant toutes les interrogations des deux modes, chaque ligne portant `MODE_KRAFTWERK` = `WEB` ou `TEL`, colonnes alignées.
- **CN-MULTI-02** — Mode unique : pas de réconciliation à faire, le jeu « multimode » est identique au jeu unimodal (à confirmer : produit-on quand même la variable `MODE_KRAFTWERK` ?).

## Edge cases

- **CL-MULTI-01** — **Variable présente dans un seul mode** : doit apparaître dans le multimode, avec valeur vide/nulle pour les lignes des autres modes.
- **CL-MULTI-02** — **Conflit de type sur une même variable entre modes** : anomalie non attendue (DDI censés identiques). Comportement cible (RG-MULTI-04) : **erreur lisible** signalant la variable et les types en conflit ; **repli** possible en **deux colonnes** distinctes. Pas de conversion silencieuse.
- **CL-MULTI-03** — **Même interrogation présente dans deux modes (doublon inter-mode)** : le guide évoque une gestion manuelle par script ; la cible doit définir une règle par défaut (garder les deux lignes distinguées par `MODE_KRAFTWERK` ? dédupliquer ? prioriser un mode ?).
- **CL-MULTI-04** — **Modes avec structures de boucles différentes** : la fusion doit se faire niveau d'information par niveau d'information ; le comportement quand une boucle existe dans un mode et pas dans l'autre est à préciser.
