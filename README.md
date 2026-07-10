# Kraftwerk — Documentation de la refonte

Documentation de l'analyse et de la conception de la refonte de **Kraftwerk** (service INSEE de mise en forme et d'export des données d'enquête post-collecte). Objectif : une version plus performante, sans VTL, alignée Genesis-only, et maintenable.

La documentation est organisée en **trois temps** : d'où l'on part (**existant**), où l'on va (**cible**), et comment on y va (**chemin vers la cible**).

> 👉 **Nouveau sur le projet / présentation à l'équipe ?** Commencez par **[PRESENTATION.md](./PRESENTATION.md)** — résumé 1 page (choix actés + décisions à prendre).

> **Méthodologie** : les documents marqués ✅ *vérifié* sont ancrés dans le code (`fichier:ligne`). Certains documents initiaux (« Mistral Vibe ») non vérifiés ont été soit corrigés en place, soit archivés en annexe lorsqu'ils sont dépassés — c'est signalé au cas par cas.

---

## Arborescence

```
documentation-kraftwerk/
├── README.md                         # Ce fichier — index
│
├── 1-existant/                       # D'OÙ L'ON PART — analyse de l'app actuelle (v4.1.3)
│   ├── 01-architecture-globale.md
│   ├── 02-fonctionnalites-principales.md
│   ├── 03-choix-architecturaux-et-problemes.md
│   ├── 04-comparaison-code-tests-guide.md
│   ├── 05-anomalies-code.md
│   └── 06-dette-technique-et-garde-fous.md
│
├── 2-cible/                          # OÙ L'ON VA — définition de la cible
│   ├── 01-objectifs-refonte.md
│   ├── 02-decisions-client-arbitrages.md
│   ├── 03-specification-fonctionnelle.md
│   ├── 04-specification-technique.md
│   ├── 05-couverture-tests-cucumber.md
│   └── regles-metiers/               # Règles métier cible par domaine (RG/CN/CL)
│       ├── README.md
│       └── 01..09  (ACQ, STR, LIEN, MULTI, CSV, JSON, REP, ANO, EXE)
│
├── 3-chemin-vers-la-cible/           # COMMENT ON Y VA — trajectoire MVP → cible
│   ├── 01-backlog-mvp-vers-cible.md
│   ├── annexe-01-choix-architecturaux-initiaux.md   (archivé)
│   ├── annexe-02-synthese-executive-initiale.md     (archivé)
│   └── annexe-03-suivi-initial.md                   (archivé)
│
└── schemas/                          # Visuels
    ├── existant-vs-cible.drawio      (éditable draw.io)
    └── existant-vs-cible.png         (pour présentation)
```

---

## 1 — Existant (`1-existant/`)

Analyse de l'application actuelle, vérifiée ligne à ligne.

| Doc | Contenu | Statut |
|---|---|---|
| [01-architecture-globale](./1-existant/01-architecture-globale.md) | Architecture modulaire, stack, points d'entrée, services REST, intégrations | ✅ vérifié (2026-07-08) |
| [02-fonctionnalites-principales](./1-existant/02-fonctionnalites-principales.md) | Fonctionnalités par module, endpoints, paramètres, formats | ✅ vérifié (2026-07-08) |
| [03-choix-architecturaux-et-problemes](./1-existant/03-choix-architecturaux-et-problemes.md) | Choix majeurs et problèmes ⚠️ *sa reco « downgrade LTS » est erronée — voir 06* | à lire avec réserve |
| [04-comparaison-code-tests-guide](./1-existant/04-comparaison-code-tests-guide.md) | Écarts code ↔ tests ↔ guide utilisateur ; décisions arbitrées | ✅ |
| [05-anomalies-code](./1-existant/05-anomalies-code.md) | 13 anomalies internes au code (bugs probables, code mort, couplages) | ✅ vérifié |
| [06-dette-technique-et-garde-fous](./1-existant/06-dette-technique-et-garde-fous.md) | Audit de dette scoré (**Health Score 44/100**), registre priorisé, garde-fous anti-dette | ✅ vérifié (2026-07-09) |

## 2 — Cible (`2-cible/`)

Ce que la refonte doit produire.

| Doc | Contenu | Statut |
|---|---|---|
| [01-objectifs-refonte](./2-cible/01-objectifs-refonte.md) | Objectifs client, besoins (must have / évolutions), NFR, contraintes | ✅ source client |
| [02-decisions-client-arbitrages](./2-cible/02-decisions-client-arbitrages.md) | Réponses client et arbitrages de périmètre | ✅ |
| [03-specification-fonctionnelle](./2-cible/03-specification-fonctionnelle.md) | Ce que fait la cible : flux, fonctionnalités par domaine, statuts, points ouverts | ✅ (2026-07-09) |
| [04-specification-technique](./2-cible/04-specification-technique.md) | Archi cible **à l'état de l'art** : virtual threads + DuckDB ensembliste, **séparation API/batch**, MinIO, ports/adapters, client Genesis déclaratif + Resilience4j, `ProblemDetail` (RFC 9457), observabilité OpenTelemetry, SBOM/supply-chain, CDS, **suivi de jobs en mémoire** | ✅ (2026-07-09, v1.1 + révisions client) |
| [05-couverture-tests-cucumber](./2-cible/05-couverture-tests-cucumber.md) | Écart de couverture Cucumber existant → cible (~69 scénarios) | ✅ |
| [regles-metiers/](./2-cible/regles-metiers/README.md) | **Règles métier cible** par domaine (RG/CN/CL), source faisant foi | ✅ |

## 3 — Chemin vers la cible (`3-chemin-vers-la-cible/`)

Comment on passe de l'existant à la cible.

| Doc | Contenu | Statut |
|---|---|---|
| [01-backlog-mvp-vers-cible](./3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md) | **Backlog** 14 epics / ~60 US, vagues V0→V5, du MVP (Parquet depuis Genesis, sans liens 2à2 ni chiffrement) à la cible, jalons M0→M5 | ✅ **fait foi** (2026-07-09) |
| [annexe-01-choix-architecturaux-initiaux](./3-chemin-vers-la-cible/annexe-01-choix-architecturaux-initiaux.md) | Choix archi initiaux | 🗄️ archivé (superseded par 2-cible/04) |
| [annexe-02-synthese-executive-initiale](./3-chemin-vers-la-cible/annexe-02-synthese-executive-initiale.md) | Synthèse exécutive initiale | 🗄️ archivé (superseded par le backlog) |
| [annexe-03-suivi-initial](./3-chemin-vers-la-cible/annexe-03-suivi-initial.md) | État d'avancement initial | 🗄️ archivé |

## Schémas (`schemas/`)

**Comparaison existant / cible** (parties prenantes) :
- [existant-vs-cible.png](./schemas/existant-vs-cible.png) — prêt à présenter · [source .drawio](./schemas/existant-vs-cible.drawio)

**Architecture cible en modèle C4** :
- [cible-c4-contexte.png](./schemas/cible-c4-contexte.png) — **niveau 1 (Contexte)** : acteurs + systèmes externes.
- [cible-c4-conteneurs.png](./schemas/cible-c4-conteneurs.png) — **niveau 2 (Conteneurs)** : batch / api / core, DuckDB, suivi de jobs, dans la frontière Kubernetes · [source .drawio](./schemas/cible-c4-conteneurs.drawio).
- [cible-c4-composants.png](./schemas/cible-c4-composants.png) — **niveau 3 (Composants)** : cœur hexagonal (ports & adapters).

**Pipeline & parallélisation** :
- [cible-pipeline-parallelisme.png](./schemas/cible-pipeline-parallelisme.png) — étapes d'un export (fetch → mise en forme par document → accumulation → réconciliation → export) et **où chaque étape est parallélisable** (cf. spec technique §4.1).

---

## Par où commencer

- **Décideur / partie prenante** : le schéma [`schemas/existant-vs-cible.png`](./schemas/existant-vs-cible.png), puis [2-cible/01-objectifs-refonte](./2-cible/01-objectifs-refonte.md) et le [backlog](./3-chemin-vers-la-cible/01-backlog-mvp-vers-cible.md).
- **Architecte / dev** : [2-cible/04-specification-technique](./2-cible/04-specification-technique.md) + [1-existant/06-dette-technique-et-garde-fous](./1-existant/06-dette-technique-et-garde-fous.md).
- **Analyste métier / test** : [2-cible/regles-metiers](./2-cible/regles-metiers/README.md) + [2-cible/05-couverture-tests-cucumber](./2-cible/05-couverture-tests-cucumber.md).

---

## Décisions cibles clés (actées)

- **Stack** : Java 25 + Spring Boot 4.1 + Maven (décision client).
- **Acquisition** : Genesis uniquement, par `partitionId` (fin du mode « hors Genesis »).
- **Zéro VTL/Trevas** : variables déjà calculées par Genesis ; réconciliation et découpage réécrits en **SQL DuckDB**.
- **Performance** : virtual threads + DuckDB ensembliste, objectif **< 1 Go RAM** pour 1800 interrogations.
- **Déploiement** : **Kubernetes**, stockage **MinIO** ; **séparation API / Batch** (le batch est le mode principal, l'**utilité de l'API reste à confirmer**) ; POM parent unique.
- **Jobs** : suivi **unifié en mémoire, au fil de l'eau, NON persistant** (traitements temporaires et rejouables — un plantage perd les jobs en cours, c'est accepté).
- **Chiffrement** : optionnel, sécurise les données **tout au long du traitement** ; implémentation **interchangeable** (lib INSEE interne derrière un port).
- **Liens 2à2** : délégués à **BPM** (contraintes métier : max 20, nommage `LIEN_i`, sentinelles `0`/`99`).
- **Reportés / différés** : mode sans-DDI (attente du nouveau modèle Lunatic enrichi), anonymisation (hors 1ʳᵉ enquête), lecture reporting depuis fichiers (version avancée).
- **État de l'art** : client Genesis déclaratif (`@HttpExchange`) + Resilience4j, erreurs `ProblemDetail` (RFC 9457), observabilité **Micrometer + OpenTelemetry**, contexte par `ScopedValue`, **SBOM + scan supply-chain**, démarrage/mémoire via **CDS/AOT**.

---

*Documentation Kraftwerk — INSEE · dernière réorganisation 2026-07-09*
