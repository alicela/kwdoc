# Domaine REP — Données de reporting (suivi de collecte)

## Périmètre

Mise en forme des données de suivi de collecte (états de contact, tentatives, résultat d'identification/repérage). Produit un **fichier de sortie séparé** du fichier de données principal.

> **Périmètre vs paradonnées** : la décision d'exclure les *paradonnées* (domaine ACQ / point A) ne concerne **pas** les *données de reporting*, qui sont un sujet distinct. Le reporting reste dans le périmètre.
>
> **Note guide** : le guide (`sortie.md`) précise que seul le reporting **Sabiane** (enquêteurs) est exporté par Kraftwerk ; le reporting **Platine** (web) et les paradonnées ne le sont pas. À confirmer pour la cible.

---

## Règles de gestion

### RG-REP-01 — Fichier de reporting séparé
Les données de reporting produisent un fichier de sortie **distinct** du fichier principal : `<campagne>_REPORTINGDATA.csv` (et `.parquet`), et non des colonnes fusionnées dans `_RACINE.csv`.
- **Source** : test (`do_we_export_reporting_data.feature` vérifie explicitement que `_RACINE.csv` **n'existe pas** en mode reporting-only) ; code (`ReportingDataParser.integrateReportingDataIntoUE` crée un groupe dédié `REPORTINGDATA`).
- **Statut cible** : 🟢 Conservé.

### RG-REP-02 — Deux points d'entrée selon la source
Un traitement selon que la donnée de reporting provient d'un chemin « Genesis » ou d'un chemin « fichier » historique (endpoints existants `PUT /reportingdata/genesis` et `PUT /reportingdata/main`).
- **Source** : code (`ReportingDataService`) ; guide (`reporting.md`).
- **Statut cible** : 🔵 Adapté — avec la suppression du mode « hors Genesis », le chemin « fichier » (`/reportingdata/main`) est à réévaluer ; la cible devrait n'avoir qu'un chemin aligné sur Genesis/`partitionId`. **Question ouverte**.

### RG-REP-03 — Formats d'entrée inchangés (XML et CSV)
La donnée de reporting en entrée reste au **même format qu'actuellement** : XML ou CSV (en-tête CSV strict à 8 colonnes : `statut,dateInfo,idUe,idContact,nom,prenom,adresse,numeroDeLot`).
- **Source** : décision client (2026-07-08 : format des reportings identique à l'actuel) ; code (`XMLReportingDataParser`, `CSVReportingDataParser`).
- **Statut cible** : 🟢 Conservé.

### RG-REP-04 — Variables produites
Le fichier de reporting contient notamment : identifiants (`interrogationId`, identifiants d'échantillon `RGES/NUMFA/SSECH/LE/EC/BS/NOI` + `IDSTATINSEE` calculé), `IDENQ` (enquêteur), `ORGANIZATION_UNIT_ID`, `TYPE_SPOTTING`, les états `STATE_i`/`STATE_i_DATE` + `LAST_STATE`, le résultat `OUTCOME`/`OUTCOME_DATE`, les tentatives `ATTEMPT_i`/`ATTEMPT_i_DATE` + `NUMBER_CONTACT_ATTEMPTS` + `LAST_ATTEMPT_DATE`, le bloc identification (`IDENTIFICATION`, `ACCESS`, `SITUATION`, `CATEGORY`, `OCCUPANT`, `INDIVIDUALSTATUS`, `INTERVIEWERCANPROCESS`), `OUTCOME_SPOTTING`, les commentaires, `CLOSINGCAUSE`/`CLOSINGCAUSE_DATE`, `REPORT_DATE_COLLECTE`.
- **Source** : code (`ReportingDataParser`, `Constants`) ; tests (`ReportingDataDefinitions` : liste de champs vérifiée) ; guide (`reporting.md`, liste détaillée).
- **Statut cible** : 🟢 Conservé.

### RG-REP-05 — `IDSTATINSEE` calculé
`IDSTATINSEE` = concaténation zéro-paddée de `Rges(2) + Numfa(6) + Ssech(2) + Le + Ec + Bs + Noi(2)`.
- **Source** : code (`InseeSampleIdentifier.getIdStatInsee`).
- **Statut cible** : 🟢 Conservé.

### RG-REP-06 — Sélection de la date de validation d'enquête
Parmi les états, la date de validation retenue suit un ordre de priorité : `PARTIELINT` (3) > `VALPAP` (2) > `VALINT` (1).
- **Source** : code (`State.isValidationState`/`isPriorTo`).
- **Statut cible** : 🟢 Conservé.

### RG-REP-07 — Calcul de `OUTCOME_SPOTTING` (repérage)
`OUTCOME_SPOTTING` est calculé à partir du bloc d'identification, selon un algorithme qui **dépend de `TYPE_SPOTTING`** (la configuration d'identification). Trois familles :

**a) `TYPE_SPOTTING` ∈ {`IASCO`, `HOUSEF2F`}** (ou config nulle) — cascade sur `identification → access → situation → category → occupant` :
- `identification=DESTROY` → `DESTROY`
- `identification=UNIDENTIFIED` → `UNIDENTIF`
- `access=NACC` → `NACCNO`
- `situation=ABSORBED` → `ACCABS`
- puis selon `category` : `VACANT`→`ACCVAC`, `SECONDARY`→`ACCSEC`, `PRIMARY` + `occupant=IDENTIFIED`→`ACCPRIDENT`, `PRIMARY` + `occupant=UNIDENTIFIED`→`ACCPRUNIDENT`, sinon `ACCNO`.
- (Le guide note une simplification récente : retrait des modalités `OCCASIONAL` et `DK` de `category`.)

**b) `TYPE_SPOTTING` ∈ {`INDTEL`, `INTELNOR`}** — sur `individualStatus` (+ `situation`) : `DCD`→`INDDCD`, `NOIDENT`→`INDNOIDENT`, `NOFIELD`→`INDNOFIELD`, `SAMEADRESS`→`INDORDSADR`/`INDNORDSADR` selon `situation`, `OTHERADRESS`→`INDORDOADR`/`INDNORDOADR` selon `situation`.

**c) `TYPE_SPOTTING` ∈ {`INDF2F`, `INF2FNOR`}** — comme (b) mais tenant compte de `interviewerCanProcess` (`-`/`NO`/`YES`), avec en plus la valeur `NOTREAT`.

- **Source** : code (`ReportingIdentification.getOutcomeSpotting` + `outcomeForHouseF2F`/`outcomeForPhone`/`outcomeForIndF2F`) ; guide (`reporting.md` section 2, algorithme détaillé) ; tests (`do_we_export_reporting_data.feature` : 9 lignes de vérification, ex. `IDENTIFICATION=DESTROY`→`OUTCOME_SPOTTING=DESTROY`, config `INDTEL`→`INDIVIDUALSTATUS=OTHER_ADDRESS`…).
- **Statut cible** : 🟢 Conservé — **cohérence code/guide/tests confirmée**, c'est la règle métier de reporting la plus stable. À réimplémenter à l'identique (pas de VTL impliqué ici, c'est du Java).

### RG-REP-08 — Format des dates de reporting
Les dates dans le fichier de reporting suivent le format `yyyy-MM-dd-hh-mm-ss`.
- **Source** : test (`ReportingDataDefinitions.check_contact_attempt_date_format` sur `STATE_1_DATE`, `REPORT_DATE_COLLECTE`).
- **Statut cible** : 🟢 Conservé.

### RG-REP-09 — Nomenclatures d'états, tentatives, résultats
Vocabulaires codés fixes : états Sabiane (`NVM, NNS, ANV, VIN, VIC, PRC, AOC, APS, INS, WFT, WFS, TBR, FIN, CLO, NVA`), états web (`INITLA, PND, DECHET, PARTIELINT, HC, VALPAP, VALINT, REFUSAL, RELANCE`), résultats de contact `OUTCOME` (`INA, REF, IMP, UTR, ALA, UCD, DUK, NOA, NUH`), tentatives `ATTEMPT` (`INA, APT, REF, TUN, NOC, MES, UCD, PUN, NPS, NLH`), causes de clôture `CLOSINGCAUSE` (`NPA, NPI, NPX, ROW`).
- **Source** : code (enums `StateType`, `ContactOutcomeType`, `ContactAttemptType`, `ClosingCauseValue`) ; guide (`reporting.md`).
- **Statut cible** : 🟢 Conservé.

---

## Cas nominaux

- **CN-REP-01** — Reporting d'enquête face-à-face (config `HOUSEF2F`) : fichier `_REPORTINGDATA` avec états, tentatives, bloc identification, `OUTCOME_SPOTTING` calculé selon la cascade (a).
- **CN-REP-02** — Reporting téléphonique (config `INDTEL`) : `OUTCOME_SPOTTING` calculé selon (b).
- **CN-REP-03** — Comptage cohérent : `do_we_export_reporting_data.feature` vérifie p. ex. 2 tentatives `REF` pour l'UE `0000003`, 5 états `WFT` pour l'UE `0000002`.

## Edge cases

- **CL-REP-01** — **`TYPE_SPOTTING` inconnu / absent** : le code retombe sur la cascade `HOUSEF2F` par défaut (config nulle). À confirmer comme comportement voulu.
- **CL-REP-02** — **Bloc identification incomplet** (ex. `access` absent) : l'algorithme `OUTCOME_SPOTTING` doit produire une valeur définie (vide ?) plutôt qu'échouer. Le code produit une valeur vide dans les branches non couvertes.
- **CL-REP-03** — **En-tête CSV non conforme** (≠ les 8 colonnes attendues) : le parseur CSV actuel rejette/loggue ; comportement à figer (erreur explicite).
- **CL-REP-04** — **Nombre variable de tentatives/états** : `STATE_i` / `ATTEMPT_i` ont autant de colonnes que d'occurrences ; le nombre de colonnes varie donc d'une partition à l'autre. À prendre en compte pour la stabilité du schéma de sortie.
- **CL-REP-05** — **Commentaires** : `COMMENT_i` est décrit comme « toujours vide » dans le guide — confirmer si ces colonnes doivent être produites du tout.
