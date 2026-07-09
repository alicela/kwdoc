# Kraftwerk cible — Règles métier, cas nominaux et edge cases

## Objet

Ce dossier formalise les **règles métier** de la cible Kraftwerk, organisées **par fonctionnalité**, chacune accompagnée de ses **cas nominaux** et **edge cases**. C'est le livrable qui alimente directement la conception et l'implémentation technique.

Il intègre les décisions déjà arbitrées :
- Acquisition des données **uniquement via Genesis, par `partitionId`** (plus de mode « hors Genesis »).
- **Liens 2à2 conservés**.
- **Fusion multimode automatique conservée**.
- **Paradonnées exclues** du périmètre Kraftwerk.
- **Aucun vocabulaire ni dépendance VTL/Trevas** : les transformations conservées doivent être réexprimées autrement (SQL DuckDB notamment).
- **Suivi de jobs unifié, au fil de l'eau, NON persistant** (traitements temporaires et rejouables).
- **Déploiement Kubernetes, stockage MinIO** ; **séparation API / Batch** (utilité de l'API à confirmer).
- **Chiffrement optionnel tout au long du traitement**, implémentation **interchangeable**.

Voir `../../1-existant/04-comparaison-code-tests-guide.md` (section « Décisions arbitrées ») et `../02-decisions-client-arbitrages.md` pour le détail des arbitrages.

---

## Conventions de notation

| Préfixe | Signification |
|---------|---------------|
| **RG-`DOM`-`NN`** | Règle de gestion (business rule) du domaine `DOM` |
| **CN-`DOM`-`NN`** | Cas nominal (scénario passant attendu) |
| **CL-`DOM`-`NN`** | Cas limite / edge case (à gérer explicitement) |

Chaque règle porte deux marqueurs :

- **Source** : d'où vient la règle — `code` (avec `fichier:ligne`), `guide`, `test` (scénario Cucumber), ou `décision` (arbitrage client).
- **Statut cible** :
  - 🟢 **Conservé** — comportement existant repris tel quel.
  - 🔵 **Adapté** — comportement conservé fonctionnellement mais dont l'implémentation change (ex. réécriture sans VTL, passage à `partitionId`).
  - 🟠 **À concevoir** — attendu de la cible non couvert par le code actuel (ou couvert uniquement dans une partie supprimée).
  - 🔴 **Supprimé** — comportement de l'existant explicitement retiré de la cible (documenté pour mémoire / non-régression inverse).

---

## Index des domaines

| Fichier | Domaine | Code |
|---------|---------|------|
| [01-acquisition-donnees.md](./01-acquisition-donnees.md) | Acquisition des données Genesis (partitionId, modes, variables, états, données partielles) | `ACQ` |
| [02-structure-sorties-boucles.md](./02-structure-sorties-boucles.md) | Structure des sorties : niveaux d'information, boucles, identifiants | `STR` |
| [03-liens-2a2.md](./03-liens-2a2.md) | Liens 2à2 (matrice de liens familiaux) | `LIEN` |
| [04-reconciliation-multimode.md](./04-reconciliation-multimode.md) | Réconciliation / fusion multimode automatique | `MULTI` |
| [05-export-csv-parquet.md](./05-export-csv-parquet.md) | Export tabulaire CSV / Parquet (formatage, nommage, scripts d'import) | `CSV` |
| [06-export-json-si-externe.md](./06-export-json-si-externe.md) | Export JSON pour SI externes (incrémental, replay) | `JSON` |
| [07-reporting-data.md](./07-reporting-data.md) | Données de reporting (états, tentatives, `OUTCOME_SPOTTING`) | `REP` |
| [08-anonymisation-chiffrement-stockage.md](./08-anonymisation-chiffrement-stockage.md) | Anonymisation, chiffrement, archivage, stockage | `ANO` |
| [09-execution-jobs-api-securite.md](./09-execution-jobs-api-securite.md) | Exécution (jobs async), API REST, sécurité, erreurs | `EXE` |

---

## Points de conception ouverts signalés dans ces documents

Certaines règles sont marquées 🟠 **À concevoir** parce que l'existant les implémente d'une façon incompatible avec la cible Genesis-only / sans-VTL. Les plus structurants :

1. **Réécriture sans VTL de la répartition des variables en tables (`STR`) et de la réconciliation multimode (`MULTI`)** : logique conservée fonctionnellement mais à réexprimer (SQL DuckDB privilégié). ✅ Validé client.
2. **Suivi de jobs persistant** (`EXE`) : à concevoir (l'existant est en mémoire, non persistant, et partiellement non exposé).
3. **Anonymisation** (`ANO`) : le principe est arrêté (suppression de colonnes, paramétrée par enquête) ; reste à concevoir le **format de configuration** par enquête (stockage, transmission à Kraftwerk).

**Points clarifiés (levés) :**
- **Liens 2à2** (`LIEN`) : plus un chantier Kraftwerk — traitement **déporté dans BPM**. Conserver la dépendance `fr.insee.bpm` et consommer les variables produites.
- **Mode sans DDI** (`ACQ`) : **conservé**.
- **Scripts d'import** (`CSV`) : **R conservé, SAS supprimé**.
- **Format des données de reporting en entrée** (`REP`) : **inchangé** (XML + CSV 8 colonnes).

Les points restants (1-3) devront être tranchés avec l'équipe technique lors de la conception détaillée.
