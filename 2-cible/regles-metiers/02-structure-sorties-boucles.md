# Domaine STR — Structure des sorties : niveaux d'information et boucles

## Périmètre

Comment les données d'une interrogation sont réparties en plusieurs tables (« niveaux d'information ») : une table racine + une table par boucle. C'est le cœur de la mise en forme tabulaire.

---

## Règles de gestion

### RG-STR-01 — Une table par niveau d'information
La cible produit **une table par niveau d'information** = une table par groupe défini dans la structure DDI de l'enquête. Le niveau de base est la table racine `RACINE` (`Constants.ROOT_GROUP_NAME = "RACINE"`), qui contient toutes les variables au niveau de l'unité enquêtée. Chaque boucle (groupe sous la racine) donne une table supplémentaire.
- **Source** : guide (`kraftwerk.md`, `vtl.md`, `sortie.md` : « un tableau par niveau d'information = un tableau par groupe du DDI » ; exemple ménage → table ménage + table individu) ; code (`InformationLevelsProcessing` génère un dataset par groupe : `RACINE` + un par sous-groupe).
- **Statut cible** : 🔵 Adapté — **réécriture sans VTL validée client (2026-07-08)**. La logique de répartition des variables en tables (aujourd'hui en VTL `keep`/`rename`) est conservée fonctionnellement mais doit être réexprimée sans moteur VTL (SQL DuckDB probable, cf. RG-MULTI-90).

### RG-STR-02 — Contenu de la table racine
La table `RACINE` contient : les 2 identifiants fixes (`interrogationId`, `usualSurveyUnitId`), `questionnaireState`, `validationDate`, et toutes les variables de niveau unité (variables dont le `scope` = `RACINE`).
- **Source** : guide (`enqgenesis.md`) ; code (`BuildBindingsSequenceGenesis` : variables `scope == RACINE` insérées au niveau racine).
- **Statut cible** : 🟢 Conservé.

### RG-STR-03 — Contenu d'une table de boucle
Une table de boucle contient une ligne par occurrence de la boucle pour une interrogation, avec : les identifiants hérités (`interrogationId`, `usualSurveyUnitId`), un **identifiant d'occurrence** propre à la boucle, et les variables de la boucle (variables dont le `scope` = nom de la boucle, indexées par `iteration`).
- **Source** : guide (`enqgenesis.md` : « chaque ligne = une occurrence de boucle/tableau dynamique, 3 identifiants dont un d'occurrence ») ; code (`VariableModel.scope`/`iteration` ; `check_loop_field_value` dans les tests localise la ligne par `interrogationId` + suffixe d'itération).
- **Statut cible** : 🟢 Conservé.

### RG-STR-04 — Nommage de l'identifiant d'occurrence
L'identifiant d'occurrence est de la forme `<NOM_BOUCLE>-<NN>` (ex. `BOUCLE_PRENOM-01`), où `NN` est le rang de l'occurrence dans le questionnaire de l'interrogation. Le séparateur est `-`.
- **Source** : guide (`enqgenesis.md` : `BOUCLE_PRENOM_01`, `FORFAIT_01` ; `vtl.md` : `BOUCLE_INDIVIDUS-01`) ; tests (`MainDefinitions.check_loop_field_value` : le suffixe après le dernier `-` = numéro d'itération).
- **Statut cible** : 🟢 Conservé. ⚠️ **Écart de notation** entre le guide (parfois `_01`, parfois `-01`) et le code/tests (`-`) : figer `-` comme séparateur dans la cible.

### RG-STR-05 — Règle « 1 niveau d'information = 1 identifiant »
Chaque niveau d'information possède exactement un identifiant qui lui est propre (l'identifiant d'occurrence pour une boucle, `interrogationId` pour la racine). Toutes les tables contiennent `interrogationId`.
- **Source** : guide (`vtl.md`, section « Variables identifiantes »).
- **Statut cible** : 🟢 Conservé.

### RG-STR-06 — Nommage qualifié des variables de groupe
Une variable appartenant à un groupe est désignée en interne par un **nom qualifié** préfixé du nom de groupe et séparé par un point (ex. `BOUCLE_INDIVIDUS.AGE`). Le séparateur est `Constants.METADATA_SEPARATOR = "."`.
- **Source** : guide (`vtl.md`) ; code (`GroupProcessing`, `Constants.java:50`).
- **Statut cible** : 🔵 Adapté — mécanisme conservé mais aujourd'hui la hiérarchie est reconstruite en **reparsant** ces noms à points, ce qui est fragile (cf. CL-STR-03). À revoir à la conception.

### RG-STR-10 — Pas de boucles imbriquées
Une boucle ne peut pas contenir une autre boucle. La cible ne gère qu'**un seul niveau de groupe sous la racine**.
- **Source** : guide (`vtl.md` : « Il ne peut pas y avoir de boucles imbriquées ») ; code (Javadoc explicite d'`InformationLevelsProcessing` : « for now, only works with at most one level of group under root group »).
- **Statut cible** : 🟢 Conservé (limitation assumée). **Question ouverte** : la cible doit-elle lever cette limitation ou la maintenir ? (le guide la présente comme une contrainte définitive, le code comme un « for now »).

---

## Cas nominaux

- **CN-STR-01** — Interrogation sans boucle : une seule table `RACINE`, une ligne par interrogation.
- **CN-STR-02** — Interrogation avec une boucle (ex. ménage → individus) : table `RACINE` (1 ligne/interrogation) + table boucle (N lignes/interrogation, une par occurrence). Vérifié par `do_we_get_loops.feature` (`B_PRENOMREP`, itérations 1 et 2) et `do_we_export_loop_genesis.feature` (`TABESA`, `TABOFATS`).
- **CN-STR-03** — Plusieurs boucles au même niveau (ex. `BOUCLE_PRENOM`, `FORFAIT`, `NATPROD_1`) : une table par boucle, toutes filles de la racine.

## Edge cases

- **CL-STR-01** — **Boucle sans aucune occurrence pour une interrogation** : l'interrogation apparaît en racine mais n'a aucune ligne dans la table de boucle. Comportement attendu : table de boucle sans ligne pour cette interrogation (pas de ligne « vide »).
- **CL-STR-02** — **Variable présente dans les données mais absente du DDI** (ou l'inverse) : cf. CL-ACQ-04 ; impacte le rattachement au bon niveau. À tracer.
- **CL-STR-03** — **Nom de variable/groupe contenant un `.`** : romprait la reconstruction de hiérarchie par parsing de noms qualifiés (`05-anomalies-code.md` point 9). La cible doit soit interdire ce caractère, soit ne pas reconstruire la structure par parsing de chaîne (conserver la structure métadonnée d'origine).
- **CL-STR-04** — **Occurrences non contiguës / trous d'itération** : itérations 1 et 3 présentes, 2 absente. Numérotation d'occurrence attendue à définir (rang réel dans les données vs index d'itération Genesis).
- **CL-STR-05** — **Demande de boucle imbriquée** : structure DDI avec boucle dans une boucle. Aujourd'hui non géré (silencieusement ?). La cible doit au minimum détecter et signaler ce cas plutôt que produire un résultat faux.
