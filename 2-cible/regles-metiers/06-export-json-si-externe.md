# Domaine JSON — Export JSON pour SI externes

## Périmètre

Export des données au format JSON, destiné à alimenter régulièrement un SI externe. Se distingue de l'export tabulaire par son caractère **incrémental** (différentiel depuis la dernière extraction) et par un mécanisme de **rejeu** (`replay`) sans perturber la production automatisée.

Endpoints existants : `GET /json`, `GET /json/replay`, `POST /debug/json`.

---

## Règles de gestion

### RG-JSON-01 — Contenu équivalent à l'export tabulaire
Le JSON contient les mêmes informations que l'export CSV/Parquet, seul le format diffère : bloc d'identification par interrogation + données structurées par niveau (racine + boucles).
- **Source** : guide (`sortie.md`, `jsonSIexterne.md`).
- **Statut cible** : 🟢 Conservé.

### RG-JSON-02 — Structure du bloc d'identification
Par interrogation : `interrogationId`, `usualSurveyUnitId`, `partitionId`, `collectionInstrumentId`, `mode`, `isCapturedIndirectly`, `questionnaireState`, `validationDate`, puis un objet `data` contenant `RACINE` (tableau) et un tableau par boucle, chaque occurrence portant son identifiant d'occurrence (ex. `PI_TAB-13`).
- **Source** : guide (`jsonSIexterne.md`, exemple JSON complet) ; code (`JsonWriterSequence.java` : `questionnaireState`, `partitionId`…).
- **Statut cible** : 🔵 Adapté — `partitionId` est aujourd'hui présent mais **toujours vide** (« vide à ce jour » dans le guide, chaîne vide dans `JsonWriterSequence`). Dans la cible, `partitionId` devient un identifiant renseigné (puisque c'est le paramètre d'appel). Cohérence à assurer.

### RG-JSON-03 — Export incrémental par défaut (`sinceDate`)
Par défaut, l'export ne contient que les interrogations dont au moins une variable a été mise à jour depuis la **dernière extraction**. Une « date de dernière extraction » est mémorisée et mise à jour à chaque export réussi.
- **Source** : guide (`jsonSIexterne.md`) ; code (`GenesisClient` : `PUT/GET /collection-instruments/{id}/extractions/json/last`).
- **Statut cible** : 🟢 Conservé.

### RG-JSON-04 — Paramètre `sinceDate` explicite
Si `sinceDate` est fourni, l'extraction part de cette date ; sinon, elle part de la dernière date d'extraction mémorisée.
- **Source** : guide (`jsonSIexterne.md`) ; code (`MainService.jsonExtraction`).
- **Statut cible** : 🟢 Conservé.

### RG-JSON-05 — Gestion UTC / heure locale
Les dates peuvent être fournies en UTC (`sinceDate`, `Instant`) ou en heure locale France (`localSinceDate`, `LocalDateTime`), les deux étant **mutuellement exclusives**. La réponse renvoie la date d'extraction en UTC et en local.
- **Source** : code (`MainService.jsonExtraction` : 400 si les deux fournies de façon incohérente, via `DateTimeUtils.resolveInstant`).
- **Statut cible** : 🟢 Conservé.

### RG-JSON-06 — Rejeu (`replay`) sans impact sur la production
`/json/replay` permet de ré-extraire les données mises à jour dans un intervalle `[sinceDate, untilDate]` **sans** mettre à jour la date de dernière extraction mémorisée — pour ne pas perturber les exports automatisés en cours.
- **Source** : guide (`jsonSIexterne.md` : « doit être utilisé en prod pour les exports ponctuels/répétés, car l'endpoint principal enregistrerait une nouvelle date d'extraction ») ; code (`MainService.jsonExtractionReplay`).
- **Statut cible** : 🟢 Conservé — **fonctionnalité importante** à préserver telle quelle.

### RG-JSON-07 — Bornes de dates du rejeu
Pour un rejeu : une date de début est **requise** ; `untilDate` est optionnelle (défaut = maintenant). `untilDate` doit être postérieure à `sinceDate`.
- **Source** : code (`MainService.jsonExtractionReplay` ; `MainProcessingGenesisNew` : 400 « endDate must be after startDate »).
- **Statut cible** : 🟢 Conservé.

### RG-JSON-08 — Debug par interrogations ciblées
`POST /debug/json` permet d'extraire un ensemble d'interrogations précises (liste d'`interrogationIds`) pour le diagnostic ; les échecs par interrogation sont isolés (n'interrompent pas les autres) et remontés dans une liste d'erreurs.
- **Source** : code (`MainService.jsonDebugExtraction`, `DebugJsonExportDto`, `DebugJsonExportResultDto`).
- **Statut cible** : 🟠 À trancher — endpoint utile mais non documenté (Swagger vide). Conserver dans la cible comme outil de diagnostic ?

### RG-JSON-09 — Sortie JSON incrémentale vide si rien de neuf
Si aucune donnée n'a changé depuis la date de référence, l'export produit un résultat vide (l'existant renvoie « 204 No Content » sur `/json`).
- **Source** : code (`MainService.jsonExtraction` renvoie 204 si `runMainJson` = false) ; guide (« si pas de nouvelle donnée depuis la date, export produit vide »).
- **Statut cible** : 🟢 Conservé (forme exacte de la réponse vide à figer).

---

## Cas nominaux

- **CN-JSON-01** — Export incrémental automatique : appel sans date → extraction depuis la dernière date mémorisée → JSON des interrogations modifiées → date de dernière extraction mise à jour.
- **CN-JSON-02** — Rejeu sur intervalle : `/json/replay` avec `sinceDate` et `untilDate` → JSON des interrogations modifiées dans l'intervalle, date de dernière extraction **inchangée**.
- **CN-JSON-03** — Export mono-interrogation de debug : `POST /debug/json` avec une liste d'`interrogationIds` → JSON de ces interrogations + rapport d'erreurs par id.

## Edge cases

- **CL-JSON-01** — **Aucune donnée nouvelle** : réponse vide/204 (cf. RG-JSON-09) — ne doit pas être traité comme une erreur.
- **CL-JSON-02** — **Dates incohérentes** : `untilDate < sinceDate` (400), ou `sinceDate` + `localSinceDate` fournies ensemble (400).
- **CL-JSON-03** — **Interrogation inexistante en debug** : remontée dans la liste d'erreurs sans faire échouer l'ensemble.
- **CL-JSON-04** — **Partition multimode sans mode précisé** : le guide insiste que pour le multimode il faut préciser le mode, sinon plusieurs exports sont nécessaires. Comportement attendu à clarifier avec l'approche `partitionId` (une partition regroupe-t-elle déjà un seul mode ?).
- **CL-JSON-05** — **Échec d'export après mise à jour partielle de la date** : garantir que la date de dernière extraction n'est mise à jour **qu'en cas de succès complet**, pour ne pas « sauter » des données lors du prochain export.
