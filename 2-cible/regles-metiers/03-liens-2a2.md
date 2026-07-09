# Domaine LIEN — Liens 2à2 (matrice de liens familiaux)

## Périmètre

Les « liens 2à2 » décrivent, pour chaque individu d'un ménage, sa relation à chacun des autres individus (matrice de liens familiaux). C'est une fonctionnalité **conservée** dans la cible (décision client).

> ✅ **Point tranché (client, 2026-07-08).** Le traitement des liens 2à2 a été **déporté dans le module BPM** (`fr.insee.bpm`, déjà dépendance de `kraftwerk-core`). Côté Kraftwerk, il n'y a donc **rien de spécifique à réimplémenter** : la matrice de liens arrive dans les données/métadonnées produites par BPM, comme n'importe quelle autre variable. La seule exigence pour la cible est de **conserver la dépendance BPM**.
>
> L'ancienne implémentation Java `LunaticXmlDataParser.manageTcmLiens` (`kraftwerk-core/.../parsers/LunaticXmlDataParser.java:241-264`), codée en dur pour `BOUCLE_PRENOMS` et couplée au parseur XML du mode « hors Genesis » supprimé, est **obsolète et remplacée par BPM**. Elle est documentée ci-dessous uniquement comme **spécification de référence** de ce que BPM est censé produire, utile pour valider le comportement, pas pour être reprise.

---

## Comportement de référence (implémenté aujourd'hui dans BPM ; ancien code Kraftwerk à titre documentaire)

L'algorithme (désormais porté par BPM), déclenché quand la variable rencontrée s'appelle `LIENS` (`Constants.LIENS = "LIENS"`), sur le groupe `BOUCLE_PRENOMS` :

Pour chaque individu `j` de la boucle, on remplit `MAX_LINKS_ALLOWED - 1 = 20` colonnes `LIEN_1` … `LIEN_20` :
- Pour chaque position de lien `k` (de 1 à 20) :
  - valeur par défaut = `NO_PAIRWISE_VALUE = "99"` (pas de lien / hors matrice) ;
  - si `k` correspond à un individu existant de la matrice (`k <= nombre d'individus`), valeur = `SAME_AXIS_VALUE = "0"` (cas diagonale / même individu, pas de lien à lui-même) ;
  - si le nœud de lien existe réellement et est renseigné, valeur = la valeur du lien lue dans les données.
- Les colonnes sont nommées `LIEN_<k>` (`Constants.LIEN = "LIEN_"`) et rattachées à l'occurrence `j` de la boucle.

Constantes (`Constants.java:154-160`) :

| Constante | Valeur | Sens |
|-----------|--------|------|
| `MAX_LINKS_ALLOWED` | `21` | borne de boucle → 20 colonnes de lien utilisables |
| `BOUCLE_PRENOMS` | `"BOUCLE_PRENOMS"` | nom de boucle codé en dur |
| `LIEN` | `"LIEN_"` | préfixe des colonnes de lien |
| `LIENS` / `PAIRWISE_GROUP_NAME` | `"LIENS"` | nom de la variable/groupe déclencheur |
| `SAME_AXIS_VALUE` | `"0"` | même axe (diagonale, pas de lien) |
| `NO_PAIRWISE_VALUE` | `"99"` | hors matrice / pas de lien |

---

## Règles de gestion

### RG-LIEN-01 — Génération d'une matrice de liens par ménage
Pour une boucle d'individus, une matrice de colonnes décrivant, pour chaque individu, son lien avec chacun des autres individus du même ménage, est produite. **Ce calcul est réalisé par BPM**, en amont ; Kraftwerk consomme le résultat.
- **Source** : décision client (liens 2à2 conservés, déportés dans BPM) ; comportement de référence : ancien `manageTcmLiens`.
- **Statut cible** : 🟢 Conservé (via dépendance BPM — rien à implémenter côté Kraftwerk).

### RG-LIEN-02 — Nombre maximal d'individus liés
La matrice supporte jusqu'à **20 liens par individu** (borne `MAX_LINKS_ALLOWED = 21`, exclusive).
- **Source** : code (`Constants.MAX_LINKS_ALLOWED`, boucle `k < 21`).
- **Statut cible** : 🟠 À reconfirmer — cette borne fixe est-elle une contrainte métier réelle (taille max de ménage enquêté) ou un choix technique arbitraire à assouplir ?

### RG-LIEN-03 — Valeurs sentinelles
- `"0"` : même axe (l'individu vis-à-vis de lui-même) → pas de lien.
- `"99"` : position hors matrice (au-delà du nombre d'individus réels) → pas de lien.
- Toute autre valeur : le code du lien familial effectif entre les deux individus.
- **Source** : code (`SAME_AXIS_VALUE`, `NO_PAIRWISE_VALUE`).
- **Statut cible** : 🔵 Adapté — sémantique à conserver ; les valeurs littérales `"0"`/`"99"` peuvent être reconsidérées (risque de collision avec de vrais codes de lien à vérifier).

### RG-LIEN-04 — Nommage des colonnes de lien
Les colonnes sont nommées `LIEN_1` … `LIEN_20` et rattachées à l'occurrence d'individu dans la table de boucle.
- **Source** : code (`Constants.LIEN + k`).
- **Statut cible** : 🔵 Adapté.

### RG-LIEN-90 — Traitement porté par BPM
La dérivation de la matrice de liens 2à2 est de la responsabilité de **BPM**, pas de Kraftwerk. Kraftwerk doit conserver la dépendance `fr.insee.bpm` et consommer les variables de liens produites, sans logique dédiée.
- **Source** : décision client (2026-07-08).
- **Statut cible** : 🟢 Conservé (via BPM). L'ancien code `manageTcmLiens` dans le parseur XML supprimé n'est pas repris.

### RG-LIEN-91 — Spécificité d'enquête gérée par BPM
Les particularités (nom de boucle `BOUCLE_PRENOMS`, variable déclencheur `LIENS`, bornes) relèvent désormais de BPM. Toute évolution (généralisation à d'autres enquêtes, assouplissement de la borne de 20 liens) est à porter **dans BPM**, pas dans Kraftwerk.
- **Source** : décision client.
- **Statut cible** : 🟢 Conservé (via BPM).

---

## Cas nominaux

- **CN-LIEN-01** — Ménage de 3 individus : chaque individu a `LIEN_1`, `LIEN_2`, `LIEN_3` renseignés selon la matrice réelle (dont `"0"` vis-à-vis de lui-même), `LIEN_4`…`LIEN_20` = `"99"`.
- **CN-LIEN-02** — Ménage d'1 individu : `LIEN_1 = "0"`, `LIEN_2`…`LIEN_20` = `"99"`.

## Edge cases

> Ces cas relèvent désormais du comportement de **BPM** (Kraftwerk consomme le résultat). Ils sont conservés ici comme attentes à vérifier au niveau de BPM / des données reçues, pas comme logique à implémenter dans Kraftwerk.

- **CL-LIEN-01** — **Ménage de plus de 20 individus** : dépasse `MAX_LINKS_ALLOWED`. Dans le comportement de référence les liens au-delà de 20 sont silencieusement ignorés. À vérifier côté BPM : erreur explicite, ou augmentation/suppression de la borne.
- **CL-LIEN-02** — **Boucle d'individus absente** ou variable `LIENS` absente : pas de matrice produite. Comportement attendu à confirmer (pas de colonnes `LIEN_*` du tout).
- **CL-LIEN-03** — **Valeur de lien réelle égale à `"0"` ou `"99"`** : collision avec les sentinelles. À vérifier côté nomenclature des liens familiaux (le domaine des vrais codes de lien ne doit pas contenir `0`/`99`, sinon ambiguïté).
- **CL-LIEN-04** — **Enquête autre que l'enquête ménage historique** utilisant une boucle nommée différemment de `BOUCLE_PRENOMS` : non gérée aujourd'hui (nom codé en dur). Cf. RG-LIEN-91.
