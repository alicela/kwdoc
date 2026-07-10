# Kraftwerk 2 (refonte) — présentation à l'équipe

**En une phrase** : reconstruire Kraftwerk pour qu'il soit **plus léger** (< 1 Go RAM vs ~5), **plus simple à faire évoluer** et **aligné sur la filière actuelle (Genesis)**, en conservant ses fonctions d'export.

> L'outil actuel a rempli son rôle ; la refonte capitalise dessus. L'analyse de l'existant a été faite **vérifiée dans le code** (`1-existant/`), pas de table rase à l'aveugle.

## Pourquoi
- Mémoire élevée (~5 Go pour 1800 interrogations), traitement séquentiel.
- Évolutions de plus en plus coûteuses ; **VTL/Trevas** lourd et diffus dans tout le cœur.
- Le contexte a changé : **Genesis** structure désormais la donnée en amont (variables déjà calculées).

## Ce qui ne change pas
Mission : **mettre en forme et exporter** les données post-collecte (CSV / Parquet / JSON, boucles, reporting). Stack **moderne et confirmée** : **Java 25 + Spring Boot 4 + Maven**.

## Choix structurants actés
| Domaine | Décision |
|---|---|
| Source | **Genesis uniquement, par `partitionId`** (fin du mode fichiers locaux) |
| Calcul | **Zéro VTL** → **SQL DuckDB** (découpage en niveaux, réconciliation multimode) |
| Moteur | **DuckDB** (retenu pour ses exports Parquet/JSON/CSV + calcul ensembliste) |
| Performance | **virtual threads + DuckDB**, pivot **par interro×mode en flux**, cible **< 1 Go** |
| Déploiement | **Kubernetes**, stockage **MinIO**, **séparation API / Batch** |
| Architecture | **hexagonale** (ports & adapters), **POM parent unique** |
| Jobs | suivi **en mémoire, non persistant** (traitements temporaires, rejouables) |
| Sécurité données | chiffrement **optionnel, tout au long**, implémentation **interchangeable** |
| Liens 2à2 | délégués à **BPM** |
| État de l'art | client Genesis **déclaratif** + Resilience4j, erreurs **ProblemDetail**, observabilité **OpenTelemetry**, **SBOM**, **portes qualité CI** (dont « zéro-VTL ») |

## Ce qui ne sera **pas** repris
Mode « hors Genesis » et parseurs de fichiers locaux (dont `XformsDataParser`) · découpage de gros XML (`SplitterService`) · **paradonnées** · **modules TCM en VTL** et VTL utilisateur · script d'import **SAS** · archivage des fichiers d'entrée.

## Décisions restant à prendre — **vos avis attendus** 🗳️
| # | Point | Piste actuelle |
|---|---|---|
| 1 | **Endpoint Genesis par `partitionId`** à créer (équipe Genesis) — et son **format** | **NDJSON** en streaming + compression + pagination |
| 2 | **Utilité du mode API** (le batch suffit-il pour vos usages ?) | à trancher |
| 3 | **Pivot** « lignes de variables → tables larges » : staging+`GROUP BY` (A) **vs** par interro×mode en flux (B) | spike (US-1.0), B pressenti |
| 4 | Numérotation d'occurrence : `dense_rank` (contigu) **vs** `iteration` brut | à décider |
| 5 | Format de la **config d'anonymisation** par enquête | *hors 1ʳᵉ enquête* |
| 6 | **Mode sans-DDI** (Lunatic seul) | différé — attente du nouveau modèle Lunatic enrichi |
| 7 | Langages du **post-script** sur tables finales | SQL sûr ; R à discuter |
| 8 | Règle par défaut **doublon inter-mode** · forme de l'**inspection DuckDB** (diagnostic) | à cadrer |

## Par où on démarre
- **EPIC-0 (fondations)** — squelette hexagonal (core / batch / api), ports & adapters, CI + portes qualité : **peut démarrer tout de suite**.
- **En parallèle, les 2 points durs** : l'**endpoint Genesis** (US-0.9, équipe Genesis) et le **spike du pivot** (US-1.0). Le reste (JSON, reporting, chiffrement, multimode, anonymisation) arrive **par incréments**.

## Pour aller plus loin
- **Vue d'ensemble** : `schemas/existant-vs-cible.png`
- **Architecture cible (C4)** : `schemas/cible-c4-contexte.png` · `…-conteneurs.png` · `…-composants.png`
- **Pipeline & parallélisation** : `schemas/cible-pipeline-parallelisme.png`
- **Détail** : specs fonctionnelle & technique et règles métier dans `2-cible/` · backlog `3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md` · audit de dette `1-existant/06-dette-technique-et-garde-fous.md`

*(Index complet : [README](./README.md).)*
