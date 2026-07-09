# Domaine ANO — Anonymisation, chiffrement, archivage, stockage

## Périmètre

Fonctionnalités transverses appliquées aux sorties : suppression de variables (anonymisation), chiffrement, archivage ZIP, et stockage (disque / MinIO).

---

## Anonymisation (suppression de variables)

> ⏸️ **Priorité (client 2026-07-09) :** l'anonymisation **n'est pas nécessaire pour la première enquête** qui utilisera KW2. Elle est donc **hors MVP** et planifiée dans une vague ultérieure.

### RG-ANO-01 — Anonymisation = suppression de colonnes
L'anonymisation consiste à **supprimer (drop) des colonnes** des tables produites. Elle remplace l'usage principal des anciens « scripts aval ».
- **Source** : décision client (2026-07-08 : « pour anonymiser il faut drop des colonnes de tables ») ; Q3 de `02-decisions-client-arbitrages.md`.
- **Statut cible** : 🟠 À concevoir (fonctionnalité formalisée, **non requise pour la 1ʳᵉ enquête**).

### RG-ANO-02 — Liste de colonnes paramétrée par enquête
La liste des colonnes à supprimer est **paramétrable par enquête** (configuration propre à chaque enquête, et non requête ad hoc à chaque appel). L'implémentation peut s'appuyer sur DuckDB (ex. `SELECT * EXCLUDE (...)`) puisque la base est déjà au cœur de l'écriture des sorties, mais le point d'entrée fonctionnel est une **liste de variables à exclure définie au niveau de l'enquête**.
- **Source** : décision client (2026-07-08 : « à paramétrer selon l'enquête »).
- **Statut cible** : 🟠 À concevoir — le format exact de cette configuration par enquête (où elle est stockée, comment elle est fournie à Kraftwerk) est à définir en conception.

### RG-ANO-03 — Anonymisation avant écriture (et avant archivage/chiffrement)
Les colonnes exclues doivent être absentes des fichiers de sortie (CSV, Parquet, JSON) — pas seulement masquées — et l'exclusion doit intervenir **avant** tout archivage/chiffrement (cf. CL-ANO-05).
- **Source** : décision client.
- **Statut cible** : 🟠 À concevoir.

### RG-ANO-04 — Portée par table
La suppression s'applique par table (RACINE, boucles, reporting), une même variable pouvant devoir être supprimée dans plusieurs tables. La configuration doit permettre de cibler les colonnes indépendamment du niveau d'information.
- **Source** : décision client (« drop des colonnes de tables »).
- **Statut cible** : 🟠 À concevoir.

---

## Chiffrement

### RG-ANO-10 — Chiffrement optionnel sur demande
Le chiffrement des sorties est activable par un paramètre (`withEncryption`, défaut : désactivé), utile pour les enquêtes sensibles.
- **Source** : code (`withEncryption` sur les endpoints) ; guide (`enqgenesis.md` : option « crypter le résultat », défaut false).
- **Statut cible** : 🟢 Conservé.

### RG-ANO-11 — Chiffrement sécurisant les données tout au long du traitement
Le chiffrement optionnel vise à **sécuriser les données pendant toute la durée du traitement**, et non plus seulement l'archive finale. L'existant ne chiffrait que l'**archive ZIP** de sortie (`<date>.zip.enc`, dossier clair supprimé) ; la cible doit **étendre la protection** aux données manipulées en cours de traitement (fichiers de travail / sorties intermédiaires) selon la sensibilité de l'enquête.
- **Source** : décision client (2026-07-09) ; existant : `OutputZipService.encryptAndArchiveOutputs`, tests `out_data_encryption.feature` (4 scénarios).
- **Statut cible** : 🔵 Adapté — étendre la couverture du chiffrement au-delà du seul ZIP final ; périmètre exact (quelles données, à quel moment) à concevoir.

### RG-ANO-12 — Chiffrement symétrique via clé Vault, **implémentation interchangeable**
Le chiffrement est symétrique (AES), la clé provenant de Vault (chemin `chiffrement/trust_key/aes_key`, mount `filiere_enquetes`), via la librairie **interne** INSEE `lib_java_chiffrement_core`. Cette lib restant interne, le code doit **isoler le chiffrement derrière une interface** (port `EncryptionPort`) afin qu'un tiers qui **clone** le projet puisse **brancher une autre implémentation** sans toucher au cœur. Le profil « sans chiffrement » fournit une implémentation no-op.
- **Source** : décision client (2026-07-09) ; existant : `EncryptionUtilsImpl`, `Constants.java:163-167`, `kraftwerk-encryption-tests`.
- **Statut cible** : 🔵 Adapté — même mécanisme, mais **pluggable** (interface + implémentations substituables).

### RG-ANO-13 — Chiffrement asymétrique / signature : hors périmètre Kraftwerk (à confirmer)
Le guide (`expauto.md`) décrit un chiffrement asymétrique + signature pour les envois externes, mais ces mécanismes sont **absents du code Kraftwerk** et relèvent vraisemblablement du service d'export automatisé (Genesis/Bangles).
- **Source** : guide (`expauto.md`) ; absence de code (point H de `04-comparaison-code-tests-guide.md`).
- **Statut cible** : 🔴 Hors périmètre (hypothèse par défaut, à confirmer).

---

## Archivage et stockage

### RG-ANO-20 — Archivage optionnel des sorties
Les sorties peuvent être archivées (ZIP) en fin de traitement (paramètre `archiveAtEnd`, défaut : désactivé). Le chiffrement (RG-ANO-11) implique une archive.
- **Source** : code (`archiveAtEnd`) ; guide (`enqhorsgenesis.md`).
- **Statut cible** : 🟢 Conservé.

### RG-ANO-21 — Stockage cible : MinIO (déploiement Kubernetes)
Le déploiement cible étant sur **Kubernetes**, le backend de stockage de production est **MinIO** (`MinioImpl`) ; **pas de filesystem a priori**. L'abstraction (`StoragePort`, ex-`FileUtilsInterface`) est **conservée** — utile pour les tests et un éventuel usage local en dev — mais MinIO est le backend visé en production.
- **Source** : décision client (2026-07-09 : « déployé sur Kube, nécessitera une connexion MinIO, pas de filesystem a priori ») ; existant : `FileUtilsInterface` + implémentations.
- **Statut cible** : 🔵 Adapté — MinIO en production (abstraction préservée pour dev/test).

### RG-ANO-22 — Archivage des fichiers d'entrée
L'existant sait archiver/renommer les fichiers d'entrée traités (`MinioImpl.archiveInputFiles`, `renameInputFile`).
- **Source** : code ; **confirmé inutilisé** (client 2026-07-09).
- **Statut cible** : 🔴 Supprimé — fonctionnalité **plus utilisée** ; sans objet en Genesis-only.

---

## Cas nominaux

- **CN-ANO-01** — Export chiffré : `withEncryption=true` → `<date>.zip.enc` produit, dossier clair supprimé, déchiffrable via clé Vault (CN vérifié par les 4 scénarios de `out_data_encryption.feature`).
- **CN-ANO-02** — Export anonymisé : liste `["NOM","PRENOM","ADRESSE"]` fournie → ces colonnes absentes de tous les fichiers produits.
- **CN-ANO-03** — Sortie vers MinIO : `minio.enable=true` → fichiers écrits dans le bucket configuré au lieu du disque.

## Edge cases

- **CL-ANO-01** — **Variable à exclure inexistante** : anonymisation demandée sur une variable absente des données → ignorer silencieusement ou signaler ? À définir.
- **CL-ANO-02** — **Exclusion d'une variable identifiante** (`interrogationId`) : faut-il l'interdire (risque de rendre la sortie inexploitable) ou l'autoriser ?
- **CL-ANO-03** — **Vault indisponible** au moment du chiffrement : comportement attendu (échec explicite du job, pas de sortie en clair laissée).
- **CL-ANO-04** — **MinIO indisponible** alors que `minio.enable=true` : échec ou bascule sur disque ? (l'ancienne cible évoquait un fallback ; à décider).
- **CL-ANO-05** — **Chiffrement + anonymisation combinés** : l'anonymisation doit s'appliquer **avant** l'archivage/chiffrement (sinon des données sensibles se retrouvent dans l'archive chiffrée mais restituées au déchiffrement).
