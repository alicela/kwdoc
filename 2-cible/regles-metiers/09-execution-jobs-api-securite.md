# Domaine EXE — Exécution (jobs async), API, sécurité, erreurs

## Périmètre

Aspects techniques transverses : déclenchement asynchrone et suivi des traitements, exposition API, sécurité, et gestion des erreurs.

---

## Exécution asynchrone et suivi des jobs

### RG-EXE-01 — Traitements longs asynchrones
Un traitement d'export est déclenché de façon asynchrone : l'appel renvoie immédiatement `HTTP 202 Accepted` avec un `jobId`, le traitement se poursuivant en tâche de fond.
- **Source** : code (endpoints `/main*` renvoient 202 + jobId, exécutés via l'exécuteur `kraftwerkExecutor`) ; guide (`enqgenesis.md`).
- **Statut cible** : 🟢 Conservé.

### RG-EXE-02 — Suivi de statut unifié « au fil de l'eau », **non persistant**
La cible expose un **unique** mécanisme de suivi de statut de job, cohérent pour tous les traitements asynchrones, permettant d'**observer la progression au fil de l'eau**. Ce suivi **n'a pas à être persisté** : en cas de plantage serveur, les traitements en cours sont perdus — **ce n'est pas grave, tout est temporaire et rejouable**. Un **stockage en mémoire** suffit.
- **Source** : décision client (2026-07-09 — **révise** le point D de `04-comparaison-code-tests-guide.md`, qui demandait un suivi persistant).
- **Statut cible** : 🔵 Adapté — **unifier** les deux stores en mémoire existants (`InMemoryJobStore`, `InMemoryExportJobStore`) et **exposer correctement** le statut (l'un n'était jamais routé, `05-anomalies-code.md` points 1-2) ; **sans persistance**.

### RG-EXE-03 — Statuts de job
Un job passe par des statuts : en cours, terminé, terminé avec avertissements (partiel), échoué.
- **Source** : code (`JobStatus` : `RUNNING`, `DONE`, `PARTIAL`, `FAILED`).
- **Statut cible** : 🔵 Adapté — jeu de statuts à conserver, appliqué uniformément.

### RG-EXE-04 — Contenu du résultat de job
Le résultat consultable inclut au minimum : statut, horodatages début/fin, un résumé de contrôle (ex. nombre d'interrogations traitées), et la liste des erreurs éventuelles.
- **Source** : code (`ExportJobResultDto` : `status`, `checkResult`, `errors`, `startTime`, `endTime`).
- **Statut cible** : 🔵 Adapté.

### RG-EXE-05 — Consultation du statut
Le statut d'un job est consultable via un endpoint dédié (existant : `GET /jobs/{jobId}`).
- **Source** : code (`MainService.getJobStatus`) ; guide (`enqgenesis.md`).
- **Statut cible** : 🔵 Adapté — endpoint conservé, mais couvrant **tous** les jobs (cf. RG-EXE-02), avec gestion propre des erreurs (l'existant laisse fuir `KraftwerkException` non catchée, `05-anomalies-code.md` point 3).

---

## API REST

### RG-EXE-10 — Endpoints alignés sur Genesis / partitionId
La surface d'API cible se limite aux traitements Genesis (par `partitionId`) : export tabulaire, export JSON (+ replay), reporting, + suivi de jobs + health-check. Les endpoints du mode « hors Genesis » (`/main`, `/main/file-by-file`, `/main/lunatic-only`) et les endpoints dépréciés par `campaignId` sont retirés.
- **Source** : décisions client (Genesis-only, partitionId).
- **Statut cible** : 🔵 Adapté / 🔴 Supprimé (pour les endpoints legacy).

### RG-EXE-11 — « Pas-à-pas » et « split » supprimés ; diagnostic repensé
- Le **pas-à-pas** (`/steps/**`) servait surtout à observer les **datasets VTL à chaque étape** → **sans objet** puisque VTL est retiré : **supprimé**.
- Le **découpage de fichiers XML volumineux** (`/split/lunatic-xml`) traite des fichiers locaux (mode supprimé) : **supprimé**.
- **Besoin de diagnostic conservé** (formats variés, erreurs imprévisibles) : le **debug par interrogation** (RG-JSON-08) est **utile à la MOA** et conservé ; piste complémentaire à l'étude : **inspecter la base DuckDB de travail** (voir les tables intermédiaires d'une exécution).
- **Source** : décision client (2026-07-09) ; code (`StepByStepService`, `SplitterService`).
- **Statut cible** : 🔴 Supprimé (`/steps/**`, `/split/**`) + 🟠 À concevoir (inspection DuckDB).

### RG-EXE-14 — Séparation API / Batch
API et Batch sont **séparés** (modules/artefacts distincts partageant le cœur métier). Le déploiement cible étant sur **Kubernetes**, le **mode batch** (Job) est le mode d'exécution principal ; **l'utilité du mode API reste à confirmer**.
- **Source** : décision client (2026-07-09 — révise l'option « artefact unique » envisagée le 2026-07-09 plus tôt).
- **Statut cible** : 🔵 Adapté / 🟠 Point ouvert (utilité de l'API).

### RG-EXE-12 — Documentation API (OpenAPI/Swagger)
L'API est documentée via OpenAPI/Swagger, avec un schéma de sécurité conditionnel (aucune auth / OIDC).
- **Source** : code (`OpenApiConfiguration`).
- **Statut cible** : 🟢 Conservé (chaque endpoint cible devra être correctement décrit — l'existant a des `@Operation` vides, ex. `/debug/json`).

### RG-EXE-13 — Health-check
Un endpoint de health-check vérifie l'application, l'accès à Genesis et l'accès au stockage.
- **Source** : code (`HealthcheckService`).
- **Statut cible** : 🔵 Adapté — concept conservé, mais corriger les défauts de l'existant (résultat du test de stockage ignoré, exception Genesis non catchée qui fait planter la requête — `05-anomalies-code.md` point 4).

---

## Sécurité

### RG-EXE-20 — Authentification OIDC/JWT
L'accès à l'API est protégé par OIDC (JWT Bearer), avec un mode `NONE` possible pour les environnements ouverts. Pas de Basic Auth ni de session.
- **Source** : code (`OIDCAuthConfiguration`, `NoAuthConfiguration`).
- **Statut cible** : 🟢 Conservé.

### RG-EXE-21 — Rôles et autorisations
Trois rôles applicatifs : `ADMIN`, `USER`, `SCHEDULER`, avec hiérarchie (`ADMIN` inclut `USER` et `SCHEDULER`). Les traitements d'export requièrent `USER` ou `SCHEDULER` ; les endpoints d'administration/étapes requièrent `ADMIN`.
- **Source** : code (`ApplicationRole`, `RoleConfiguration`, règles `/main/**`, `/steps/**`, `/split/**`).
- **Statut cible** : 🔵 Adapté — modèle de rôles conservé ; les règles d'autorisation seront à recartographier sur les nouveaux endpoints.

### RG-EXE-22 — Authentification service-à-service vers Genesis
Les appels sortants vers Genesis utilisent un jeton de service (OIDC client-credentials), rafraîchi automatiquement, avec rejeu une fois sur 401.
- **Source** : code (`OidcService`, `GenesisAuthInterceptor`).
- **Statut cible** : 🟢 Conservé.

---

## Gestion des erreurs

### RG-EXE-30 — Gestion des erreurs centralisée
La cible doit centraliser la conversion des exceptions métier en réponses HTTP (contrairement à l'existant sans `@ControllerAdvice`, où chaque contrôleur gère à la main et laisse parfois fuir des 500 génériques).
- **Source** : `05-anomalies-code.md` point 3.
- **Statut cible** : 🟠 À concevoir (bonne pratique cible).

### RG-EXE-31 — Erreurs non bloquantes collectées
Les erreurs non bloquantes d'un traitement (ex. problème sur une variable) sont collectées et restituées dans un fichier d'erreurs et/ou le résultat de job, sans forcément interrompre tout le traitement.
- **Source** : code (`KraftwerkError`, `errors.txt`, statut `PARTIAL`).
- **Statut cible** : 🔵 Adapté — principe conservé ; **attention** : l'existant continue le traitement même sur des erreurs graves (capture jusqu'aux `Error` Java, `05-anomalies-code.md` point 8). La cible doit distinguer erreurs récupérables (collectées, statut `PARTIAL`) et erreurs fatales (arrêt + `FAILED`).

### RG-EXE-32 — Codes HTTP explicites
Les erreurs remontent des codes HTTP appropriés (400 paramètre invalide, 404 introuvable, 500 erreur interne…).
- **Source** : code (valeurs portées par `KraftwerkException`).
- **Statut cible** : 🔵 Adapté — définir une table de correspondance exception → code HTTP cohérente et centralisée (absente aujourd'hui).

---

## Cas nominaux

- **CN-EXE-01** — Export asynchrone : appel → 202 + jobId → job `RUNNING` → job `DONE` consultable via l'endpoint de statut, avec résumé (nb d'interrogations).
- **CN-EXE-02** — Job terminé avec avertissements : erreurs non bloquantes collectées → statut `PARTIAL` + liste d'erreurs dans le résultat.
- **CN-EXE-03** — Accès autorisé : appelant portant le rôle `USER` → export autorisé ; appelant sans rôle requis → 403.

## Edge cases

- **CL-EXE-01** — **`jobId` invalide/inconnu** : format non-UUID → 400 ; UUID inconnu → 404 — avec une vraie réponse HTTP structurée (pas une exception non gérée comme aujourd'hui).
- **CL-EXE-02** — **Redémarrage pendant un job** : le suivi étant **non persistant** (RG-EXE-02), les jobs en cours sont **perdus** au redémarrage — comportement **accepté** (traitements temporaires et rejouables). Il suffit de relancer l'export.
- **CL-EXE-03** — **Genesis injoignable** : à l'appel d'export → échec propre du job (`FAILED` + message) ; au health-check → statut dégradé (et non plantage de la requête, cf. RG-EXE-13).
- **CL-EXE-04** — **JWT expiré / rôle manquant** : 401 / 403 explicites.
- **CL-EXE-05** — **Vault/MinIO indisponibles** : cf. domaine `ANO` — le job échoue proprement sans laisser de sortie en clair ni d'état incohérent.
