# Domaine CSV — Export tabulaire CSV / Parquet

## Périmètre

Génération des fichiers de sortie tabulaires (CSV et Parquet), un fichier par niveau d'information (racine + boucles), à partir des données mises en forme. Dans l'existant, l'écriture passe par des instructions SQL `COPY ... TO` de DuckDB.

---

## Règles de gestion

### RG-CSV-01 — Deux formats de sortie : CSV et Parquet
Chaque table (racine, chaque boucle) est produite en CSV **et** en Parquet.
- **Source** : code (`outputs/csv/`, `outputs/parquet/`) ; guide (`sortie.md`) ; tests (chaque scénario d'export vérifie les deux formats).
- **Statut cible** : 🟢 Conservé. (Le guide mentionne le JSON comme format « bientôt » ; l'export JSON est traité séparément, domaine `JSON`.)

### RG-CSV-02 — Un fichier par table
Il y a un fichier par dataset : `<campagne>_RACINE.csv/.parquet`, et `<campagne>_<NOM_BOUCLE>.csv/.parquet` par boucle. Plus, le cas échéant, `<campagne>_REPORTINGDATA.csv/.parquet` (domaine `REP`).
- **Source** : code (`OutputFiles.setOutputDatasetNames`, `CsvOutputFiles`/`ParquetOutputFiles` : `<parent>_<datasetName>.<ext>`) ; tests (noms de fichiers vérifiés).
- **Statut cible** : 🔵 Adapté — le préfixe de nommage cible est le **`shortLabel` de la partition** (client 2026-07-09) : `<shortLabel>_RACINE.*`, `<shortLabel>_<BOUCLE>.*`, `<shortLabel>_REPORTINGDATA.*`.

### RG-CSV-03 — Séparateur CSV
Le séparateur de colonnes CSV est `;` (`Constants.CSV_OUTPUTS_SEPARATOR = ';'`).
- **Source** : code (`Constants.java:54`).
- **Statut cible** : 🟢 Conservé (configurable ? aujourd'hui constant).

### RG-CSV-04 — Guillemets systématiques
Tous les champs (en-tête et données) sont entourés de guillemets doubles.
- **Source** : test (`MainDefinitions.checkCSV` : vérifie que chaque champ est encadré de `"`) ; propriété `fr.insee.postcollecte.csv.output.quote`.
- **Statut cible** : 🟢 Conservé.

### RG-CSV-05 — Booléens exportés en 0/1
Les valeurs booléennes ne sont **pas** écrites `true`/`false` mais `0`/`1`.
- **Source** : test (`MainDefinitions.checkCSV` : échoue si un champ vaut littéralement `"true"`/`"false"`).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-06 — Pas de notation scientifique pour les grands nombres
Les grands nombres entiers sont écrits en toutes lettres, sans notation exponentielle (`E`). Ex. `19800340`, pas `1.98E7`.
- **Source** : test (`do_we_export_numeric.feature` : `T_NPIECES = 19800340` pour l'interrogation `0000005`).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-07 — En-tête unique (pas de ré-append)
Un fichier de sortie contient exactement une ligne d'en-tête ; le retraitement/incrémentation ne doit pas ré-ajouter de ligne d'en-tête.
- **Source** : test (`do_we_increment.feature` + `MainDefinitions.uniqueHeaderCheck`).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-08 — Parquet sans ligne d'en-tête
Le Parquet porte le schéma dans ses métadonnées : le nombre de lignes de données = nombre de lignes CSV − 1 (l'en-tête CSV n'existe pas en Parquet).
- **Source** : test (`MainDefinitions.check_parquet_output_root_table` : assert `rows == csvLines - 1`).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-09 — Script d'import R (SAS supprimé)
Chaque export s'accompagne d'un **script d'import R**. Le script d'import **SAS n'est plus produit**.
- **Source** : décision client (2026-07-08 : R conservé, SAS non) ; existant : code (`WriterSequence`, `ImportScript`) produisait les deux.
- **Statut cible** : R → 🟢 Conservé ; SAS → 🔴 Supprimé.

### RG-CSV-10 — Fichier de log d'erreurs de traitement
Chaque export produit un fichier d'erreurs (existant : `errors.txt`, `Constants.ERRORS_FILE_NAME`) recensant les problèmes non bloquants rencontrés.
- **Source** : code (`TextFileWriter.writeErrorsFile`) ; tests (`do_we_generate_errors.feature` : vérifie l'existence du fichier) ; guide (« un fichier listant les erreurs VTL »).
- **Statut cible** : 🔵 Adapté — le concept de fichier d'erreurs est conservé, mais son contenu ne parlera plus d'« erreurs VTL ». Voir domaine `EXE` pour la gestion des erreurs.

### RG-CSV-11 — Fichier de log d'exécution
Chaque exécution produit un fichier de log (existant : nom préfixé `<campagne>_LOG_...`).
- **Source** : code (`TextFileWriter.writeLogFile`) ; tests (`do_we_generate_logs.feature`).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-12 — Sorties horodatées par exécution
Les sorties d'une exécution sont rangées dans un sous-dossier daté (une exécution = un sous-dossier), permettant de conserver plusieurs exécutions.
- **Source** : code (structure `out/<campagne>/<date>`) ; tests (les vérifications ciblent « le dernier sous-dossier d'exécution ») ; guide (`enqgenesis.md` : « sous-dossiers par date d'extraction »).
- **Statut cible** : 🟢 Conservé.

### RG-CSV-13 — Post-traitement par script utilisateur sur les tables finales (nouveau)
La cible offre la possibilité de faire passer un **script utilisateur sur les tables finales dans DuckDB**, avant écriture des sorties. Langage privilégié : **SQL** (exécuté dans DuckDB) ; **R** à l'étude. Ce point remplace l'usage des anciens « scripts aval » et fournit le point d'extension de la transformation multimode (RG-MULTI-05).
- **Source** : décision client (2026-07-09).
- **Statut cible** : 🟠 À concevoir — périmètre des langages autorisés, sécurité d'exécution (script SQL maîtrisé vs arbitraire), interface de fourniture du script.

---

## Cas nominaux

- **CN-CSV-01** — Export racine simple : `<campagne>_RACINE.csv` avec en-tête unique, champs entre guillemets, séparateur `;`, + `<campagne>_RACINE.parquet` avec autant de colonnes et N−1 lignes.
- **CN-CSV-02** — Export avec boucle : fichiers racine + fichiers boucle (`do_we_export_data_file_by_file.feature` : `BOUCLE_PRENOMS` 11 lignes/223 variables en v1).
- **CN-CSV-03** — Export multimode : les tables reflètent le jeu multimode réconcilié (colonne `MODE_KRAFTWERK` présente).

## Edge cases

- **CL-CSV-01** — **Valeur contenant le séparateur `;` ou un guillemet `"`** : doit être correctement échappée (encadrement + échappement des guillemets internes). À couvrir par test (non couvert aujourd'hui).
- **CL-CSV-02** — **Valeur contenant un saut de ligne** : idem, doit rester dans un seul champ CSV.
- **CL-CSV-03** — **Grand décimal / nombre à virgule** : RG-CSV-06 ne couvre explicitement que les grands entiers ; le format des décimaux (séparateur décimal `.` ou `,`, précision, absence d'exposant) est à spécifier.
- **CL-CSV-04** — **Table vide** (aucune ligne de données, ex. boucle sans occurrence sur toute la partition) : produit-on un fichier avec en-tête seul, ou pas de fichier ? À définir.
- **CL-CSV-05** — **Valeur nulle / manquante** : représentation en CSV (champ vide `""`) et en Parquet (`NULL`) à figer et à distinguer d'une chaîne vide légitime.
- **CL-CSV-06** — **Caractères non-ASCII / encodage** : les sorties doivent être en UTF-8 (l'existant fixe `project.build.sourceEncoding=UTF-8` ; à garantir aussi sur les fichiers produits).
