# Stratégie de génération des numéros d'immatriculation des véhicules — SIGATT

> **Objet** : définir, de bout en bout, l'algorithme de génération des numéros d'immatriculation
> (les « plaques », ex. `2089 G3 03`) pour le microservice `sigatt-immatriculation-service`,
> **sans aucun risque de doublon**, en conformité avec la réglementation burkinabè et en continuité
> du système existant (352 922 numéros réels analysés).
>
> **Sources de cette stratégie** :
> - Décret n°2017-0114/PRES/PM du 17 mars 2017 (modalités d'immatriculation, 52 articles) ;
> - Arrêté conjoint n°2017-0101/MTMUSR/M-SECU du 21 juillet 2017 (normes des plaques, 24 articles) ;
> - Arrêté nomenclature genres/carrosseries/sources d'énergie ; Décret n°2017-0994 (carte grise) ;
> - `prod.txt` : extraction massive du système existant (352 922 lignes) — analysée statistiquement ;
> - Le code existant : `sigatt-backend` (socle CRUD, dossier d'immatriculation SIGATT-233),
>   `sigatt-desktop` (saisie hors-ligne + synchronisation bulk), référentiels et seeds SQL.

---

## 1. Ne pas confondre les deux numéros

Le projet manipule déjà **deux numérotations distinctes** — la confusion serait une source de bugs :

| | Numéro de **dossier** (existant) | Numéro d'**immatriculation** (objet de ce document) |
|---|---|---|
| Exemple | `OUAG 0000 210726 0001` | `2089 G3 03` |
| Rôle | Identifiant de la **demande** (traçabilité de saisie) | Identité **du véhicule** (la plaque physique) |
| Qui le génère | Backend (`DossierImmatriculationNumberHelper`) **ou** desktop (`NumeroGenerator`, hors-ligne) | **Exclusivement le serveur central** (décision clé, cf. §6.1) |
| Espace d'unicité | (site, machine, jour) → pas de coordination nécessaire | **National** → coordination obligatoire |
| Durée de vie | Le temps du traitement de la demande | **Permanente** (décret art. 3), jamais recyclée |

Le numéro de dossier peut être généré hors-ligne car son préfixe `site+machine+jour` partitionne
naturellement l'espace : deux machines ne peuvent pas se marcher dessus. **Le numéro d'immatriculation,
lui, vit dans un espace national séquentiel** (`0001 → 9999` par série) : il est impossible de le
partitionner par site sans violer la réglementation. C'est toute la difficulté que cette stratégie résout.

---

## 2. La réglementation décodée — anatomie d'un numéro

### 2.1 Structure générale

```
   2089        G3          03         (IT|AT)
  ┌────┐    ┌─────┐     ┌─────┐     ┌────────┐
  │NNNN│    │GROUPE│    │ RR  │     │MENTION │
  └────┘    └─────┘     └─────┘     └────────┘
 numéro     lettre(s)   code région  régime
 d'ordre    + chiffre   OU code      douanier
 0001-9999              mission      (optionnel)
```

Chaque composant est déterminé par un critère **précis** du décret (art. 2) :

| Composant | Déterminé par | Détail |
|---|---|---|
| **GROUPE** (lettre) | **Statut du propriétaire** | `A`=État/EPA · `B`=Collectivités · `C`=Parapublics · `P`=Police · `T`=Transporteurs publics/taxis · `CD`/`CC`=corps diplomatique/consulaire · `IN`=organisations internationales · `CMD`=chef de mission · **Privés = toute lettre SAUF `A,B,C,I,O,P,T,W`** (première lettre privée = `D`) |
| **GROUPE** (ordre lettre/chiffre) | **Type de véhicule** | Automobile = lettre puis chiffre (`G3`) ; **cycle à moteur = chiffre puis lettre (`3G`)** — deux espaces de numérotation distincts |
| **NNNN** | Séquence | 4 chiffres `0001→9999` ; chiffre du groupe `1→9` (`T` : `1→99`) ; après `9999`, passage au groupe suivant ; après épuisement des lettres simples : croisement deux à deux (`DE2`…`ZZ9`, art. 27). CD/CC/IN : bande **service `0001-0999`** / **personnel `1001-9999`**. Provisoires W/WW : 3 chiffres `001-999` |
| **RR** | **Résidence** ou **mission** | Code région `01-13` (séries ordinaires) ; **numéro d'identification de la mission/organisation** (CD/CC/IN/CMD, fixé par arrêté conjoint, art. 44) ; `99` final = export (WW) |
| **MENTION** | **Régime douanier** | (rien) = série normale · `IT` = franchise temporaire · `AT` = admission temporaire. La mention s'ajoute, le numéro vient du **même espace** que la série normale |

### 2.2 Règles de vie du numéro (décret art. 3)

- Le numéro est **permanent** tant que ne changent ni le statut du propriétaire, ni la résidence,
  ni le régime douanier.
- **Changement de région uniquement** → seul `RR` change, `NNNN + GROUPE` sont conservés.
- **Changement de statut ou de régime douanier** → ré-immatriculation complète (nouveau tirage).
- Un numéro n'est **jamais réutilisé** (les « trous » sont définitifs).

### 2.3 Ce que prouvent les données de production (352 922 numéros)

L'analyse statistique de `prod.txt` lève les ambiguïtés que les textes laissent ouvertes :

1. **Le compteur est NATIONAL par groupe** : sur toutes les séries testées (`D1`, `E5`, `F9`, `G1`, `G3`),
   **zéro** numéro n'existe avec deux régions différentes, et les numéros d'une même région sont
   entrelacés sur tout l'espace `0001-9999` (ex. dans `D1`, la région 01 possède 134 numéros éparpillés
   de 0113 à 9983 — un compteur par région aurait produit un bloc compact `0001-0134`).
   → **Clé d'unicité réelle d'une plaque ordinaire : `(NNNN, GROUPE)`. La région est descriptive.**
2. **Pour CD/CC/IN, le compteur est PAR MISSION** : `0001 CD 01`, `0001 CD 02`, `0001 CD 04`…
   coexistent. → Clé d'unicité diplomatique : `(NNNN, GROUPE, code mission)`.
3. **Remplissage séquentiel confirmé** : `D1…D9`, `E1…E9`, `F1…F9`, `G1`, `G2` sont remplies
   intégralement `0001→9999` ; la série privée active est **`G3` (dernier numéro observé : 2088)**.
   Autres compteurs vivants : `T1`≈8761, `A1`≈4237, `C1`≈1390, `P1`≈453, `B1`≈357.
4. **Bandes diplomatiques confirmées** : distribution bimodale de CD (`0001-0099` et `1000+`, rien entre).
5. **`IT`/`AT` consomment le même espace** : un numéro porté avec mention `IT` n'existe jamais sans elle.
6. **Les suffixes mission correspondent déjà aux référentiels SIGATT** : `IN 27` = UEMOA (308 véhicules),
   `IN 01` = PNUD, `IN 35` = HCR… — exactement les `code` seedés dans `organisation_internationale` ;
   idem `pays_mandataire` pour CD/CC.
7. Les doublons du fichier (42 705 lignes strictement identiques) sont des rééditions de carte grise,
   **toujours à région identique** — aucune contradiction avec les règles ci-dessus.
8. Les cycles (`1D`, `4E`…, chiffre avant lettre) sont quasi absents (4 lignes) : le parc moto n'est
   pas dans cette extraction — point à clarifier avec la DGTTM (cf. §12).

---

## 3. L'existant dans le code — ce qu'on réutilise, ce qu'on corrige

### 3.1 Backend (`sigatt-immatriculation-service`)

| Élément | État | Usage dans la stratégie |
|---|---|---|
| `VehiculeEntity.immatriculation` | Existe (String libre, **sans contrainte unique**, jamais généré) | **Champ cible** de la plaque — ajouter la contrainte d'unicité |
| `VehiculeEntity.numeroImmatriculationProvisoire` | Existe (saisie libre) | Plaques provisoires W/WW (phase ultérieure) |
| `DossierImmatriculationEntity.nouvelleImmatriculation` / `immatriculationPrecedente` | Existent (saisis, non exploités) | Déclencheur du tirage / ré-immatriculation |
| Patron **séquence sous verrou pessimiste** (`DossierImmatriculationSequenceEntity` + `findAndLockByPrefix` `@Lock(PESSIMISTIC_WRITE)` + retries) | Existant, éprouvé (testé sous concurrence par `testCreateConcurrentNominalCases`) | **Répliqué à l'identique** pour les compteurs de séries (cf. §7) |
| `StatutProprietaire.immatCode` | ⚠️ Seed **erroné** : énumération naïve `A→N` (Privé=`B`, Police=`D`, Org. internationale=`I` — contredit le décret : B=Collectivités, P=Police, I=lettre interdite) | **À re-seeder** avec les codes de catégorie réglementaires (cf. annexe B) — c'est la clé de résolution de la stratégie |
| `PaysMandataire.code` / `OrganisationInternationale.code` | Corrects (validés à 100 % contre prod) | Suffixe mission des plaques CD/CC/IN/CMD |
| `TypeVehicule` / `CategorieVehicule` (genre 1 = Motocycle) | Existent (`nombrePlaque` non seedé) | Distinction automobile/cycle (ordre lettre-chiffre) + nombre de plaques à produire (2 auto / 1 moto-remorque, décret art. 4) |
| `RegimeDouanier` | Référentiel existant, **non seedé** | Source de la mention `IT`/`AT` — ajouter la colonne de correspondance (cf. §8, fichier 9) |
| `Region` (referentiel-service) | Entité avec `code`, **table non seedée** | Source du code région `01-13` — seed à créer |
| Table `parametre` (referentiel-service, clé/valeur + écran admin) | Existante | Paramètres de l'algorithme (activation, marge d'amorçage) |

### 3.2 Desktop (`sigatt-desktop`) — la contrainte hors-ligne

Analyse complète du repo (JavaFX 25 + SQLite, sync par lots de 20 vers
`POST /immatriculation-service/api/dossier-immatriculation/bulk`) :

- **Le desktop ne génère AUCUNE plaque** — il ne connaît que le numéro de **dossier**
  (`NumeroGenerator` : `siteCode + codeMachine + ddMMyy + séquence journalière locale`), attribué à la
  soumission. Aucune table, aucun champ, aucune logique de plaque.
- La corrélation de la réponse bulk se fait **par numéro de dossier**
  (`DossierSyncSuccess { id, numero }` / `DossierSyncFailure { numeroDossier, errorMessage, dateErreur }`).
- Le desktop peut **rejouer** un lot (reprise après coupure : `IN_PROGRESS → PENDING_PUSH`) : le serveur
  doit être **idempotent** par dossier.
- Conséquence architecturale : la génération de plaque peut (et doit) être **100 % centrale**, il n'y a
  aucun générateur concurrent à coordonner côté clients. Si l'on veut afficher la plaque sur le poste de
  saisie, il faudra **étendre le contrat de retour** (`DossierSyncSuccess`) — champ additionnel, rétrocompatible.

---

## 4. Principes directeurs de la stratégie

1. **Génération exclusivement côté serveur central** — jamais en desktop, jamais en front. Un seul
   point d'attribution = un seul espace à protéger.
2. **Un numéro n'est jamais calculé, il est ALLOUÉ** : l'état des compteurs vit en base de données,
   modifié sous verrou. On ne déduit jamais « le prochain numéro » d'un `MAX()` sur les véhicules
   (fragile face aux imports et aux suppressions logiques).
3. **Défense en profondeur contre les doublons** (cf. §6) : verrou pessimiste + registre avec contrainte
   d'unicité en base + idempotence par dossier + amorçage par migration du legacy.
4. **Les trous sont légaux, le recyclage est interdit** : un échec de transaction peut consommer un
   numéro pour rien (rare) — c'est accepté (la prod en est pleine) ; réutiliser un numéro ne l'est jamais.
5. **Continuité du legacy** : les compteurs démarrent là où le système existant s'est arrêté
   (série privée `G3`, etc.) ; tous les numéros historiques sont importés dans le registre pour rendre
   toute collision physiquement impossible.
6. **Ouvert/fermé** : chaque famille réglementaire est une **stratégie** interchangeable (pattern
   Strategy) ; ajouter les séries W/WW ou les immatriculations particulières plus tard = ajouter une
   classe, sans toucher au moteur.
7. **Conventions du projet respectées** : socle `lib-commons`, nommage et Javadoc en français,
   checkstyle bloquant, `final` sur les paramètres, pas de méthode publique `final` dans les services
   proxifiés, clés i18n `<entité>.<cas>`, rôles Keycloak `{ressource}-write`.

---

## 5. Modèle de données cible

Deux nouvelles tables dans `sigatt-immatriculation-service`, une colonne d'unicité sur `vehicule`.

### 5.1 `serie_immatriculation` — l'état des compteurs vivants

Réplique du patron `dossier_immatriculation_sequence` (une ligne = un compteur, verrouillée à
l'allocation) enrichie du **groupe courant** :

```sql
CREATE TABLE serie_immatriculation (
    cle_serie       VARCHAR(64) PRIMARY KEY,  -- identifiant du compteur (cf. ci-dessous)
    groupe_courant  VARCHAR(8)  NOT NULL,     -- ex. 'G3', 'A1', 'T1' ; sigle pour les diplomatiques
    valeur_courante INTEGER     NOT NULL      -- dernier numéro d'ordre attribué (0 si vierge)
);
```

**Convention de clé (`cle_serie`)** — elle encode la portée d'unicité prouvée en §2.3 :

| Famille | Clé | Exemples de lignes |
|---|---|---|
| Privés | `PRIVE\|VEHICULE` et `PRIVE\|CYCLE` | `(PRIVE\|VEHICULE, G3, 2088)` |
| Lettre fixe (État, Collectivités, Parapublics, Police) | `<CAT>\|<SUPPORT>` | `(ETAT\|VEHICULE, A1, 4237)` |
| Transporteurs publics | `TRANSPORT\|<SUPPORT>` | `(TRANSPORT\|VEHICULE, T1, 8761)` |
| Diplomatiques (compteur **par mission et par bande**) | `<SIGLE>\|<BANDE>\|<code mission>` | `(CD\|SERVICE\|01, CD, 45)`, `(IN\|PERSONNEL\|27, IN, 1023)` |

Une ligne par compteur = **un point de verrouillage par compteur** : deux tirages simultanés sur des
séries différentes (ex. un privé et un taxi) ne se bloquent pas mutuellement.

### 5.2 `immatriculation_delivree` — le REGISTRE (cœur de l'anti-doublon)

Toute plaque qui existe — générée par l'algorithme, **migrée du legacy**, ou saisie manuellement
(source `INTERNE`) — a une ligne ici. C'est le filet de sécurité ultime, garanti par la base :

```sql
CREATE TABLE immatriculation_delivree (
    id              UUID PRIMARY KEY,
    cle_unicite     VARCHAR(32) NOT NULL,     -- la clé réglementaire, cf. ci-dessous
    numero_ordre    INTEGER     NOT NULL,     -- 2089
    groupe          VARCHAR(8)  NOT NULL,     -- 'G3', 'CD', 'CMD'
    code_suffixe    VARCHAR(4)  NOT NULL,     -- région '03' ou mission '27'
    mention         VARCHAR(2),               -- NULL | 'IT' | 'AT'
    numero_complet  VARCHAR(24) NOT NULL,     -- '2089 G3 03' (affichage)
    type_support    VARCHAR(16) NOT NULL,     -- VEHICULE | CYCLE
    source          VARCHAR(16) NOT NULL,     -- GENEREE | MIGREE | INTERNE
    vehicule_id     UUID,                     -- rattachement (nullable pour la migration)
    dossier_id      UUID,                     -- dossier à l'origine du tirage (idempotence)
    -- + colonnes d'audit AbstractAuditEntity (created_by, created_date, deleted, ...)
    CONSTRAINT uk_immatriculation_cle_unicite UNIQUE (cle_unicite)
);
CREATE INDEX idx_immat_delivree_dossier ON immatriculation_delivree (dossier_id);
```

**`cle_unicite`** matérialise la clé réglementaire, construite par la stratégie :

- séries ordinaires : `GROUPE|NNNN` → `G3|2089` (la région n'y figure PAS — art. 3 : elle peut changer) ;
- diplomatiques : `SIGLE|code mission|NNNN` → `CD|01|0045` (le compteur est par mission) ;
- CMD : `CMD|code mission` → `CMD|02` (un seul véhicule par mission, art. 20).

> **Pourquoi un registre et pas seulement une contrainte sur `vehicule.immatriculation` ?**
> (1) l'unicité réglementaire porte sur `(NNNN, GROUPE)` alors que la chaîne complète contient la
> région (qui peut changer) ; (2) le registre absorbe la **migration des 353 000 numéros legacy**
> sans créer de véhicules fantômes ; (3) il historise les plaques même après ré-immatriculation
> (jamais de recyclage) ; (4) il neutralise les collisions venant des saisies `INTERNE`/bulk.

### 5.3 Modifications d'entités existantes

- `VehiculeEntity.immatriculation` : ajouter `unique = true` (filet supplémentaire, la chaîne complète
  est de fait unique).
- `RegimeDouanierEntity` : ajouter la colonne **`mentionSerie`** (`NULL`/`IT`/`AT`) — c'est le régime
  douanier qui détermine la mention (décret art. 7) ; seed associé.
- Seed `statut_proprietaire` : **corriger `immat_code`** (cf. annexe B).
- Seed `region` (referentiel-service) : codes `01→13` (cf. annexe A, à faire valider).

---

## 6. La stratégie anti-doublon — défense en profondeur

Aucune couche ne se suffit à elle seule ; leur superposition rend le doublon **physiquement impossible**.

### Couche 1 — Un seul générateur (architectural)

Toute attribution passe par `ServiceGenerationImmatriculation` côté serveur central. Le desktop, le
back-office et le bulk **reçoivent** des plaques, ils n'en fabriquent jamais. (Vérifié : le desktop n'a
aucune logique de plaque ; sa saisie hors-ligne ne porte que le numéro de dossier.)

### Couche 2 — Sérialisation par verrou pessimiste (concurrence)

L'allocation incrémente `serie_immatriculation` sous `SELECT … FOR UPDATE`
(`@Lock(PESSIMISTIC_WRITE)`, patron déjà éprouvé dans le projet). Deux transactions concurrentes sur le
**même compteur** sont sérialisées par PostgreSQL lui-même — y compris entre **plusieurs instances**
du microservice (le verrou est en base, pas en JVM). L'avancement de série (9999 → groupe suivant)
se fait **dans la même transaction verrouillée** : aucune course possible sur le changement de groupe.

### Couche 3 — Contrainte UNIQUE du registre (filet base de données)

Dans la **même transaction** que l'incrément, l'algorithme insère la ligne du registre. Si un numéro
existe déjà (legacy plus avancé que le compteur, saisie manuelle passée avant l'amorçage…), l'INSERT
viole `uk_immatriculation_cle_unicite` → l'algorithme **avance le compteur et retente** (borné, 50
tentatives comme le patron existant). Le doublon ne peut pas atteindre la base, même en cas de bug
du compteur : c'est PostgreSQL qui a le dernier mot.

### Couche 4 — Idempotence par dossier (rejeu réseau)

Le desktop rejoue les lots interrompus. Avant tout tirage : si le registre contient déjà une ligne
`dossier_id = <dossier>` (ou si le véhicule du dossier porte déjà une plaque), **on renvoie la plaque
existante au lieu d'en tirer une nouvelle**. Un même dossier ne peut jamais consommer deux numéros.

### Couche 5 — Amorçage par migration + validation des saisies (données)

- Les 353 000 numéros de `prod.txt` sont importés dans le registre (`source = MIGREE`) et les compteurs
  amorcés à `max observé + marge paramétrable` (table `parametre`). Le générateur ne peut donc jamais
  réémettre un numéro historique — même si l'amorçage était mal calibré, la couche 3 le rattrape.
- Toute plaque **saisie** (source `INTERNE`, champ `immatriculationPrecedente`, bulk) est validée par
  les expressions régulières par catégorie (cf. §11) puis **enregistrée au registre** : elle entre dans
  l'espace protégé.

---

## 7. Architecture logicielle

### 7.1 Vue d'ensemble des composants

```
sigatt.immatriculation.service
├── domain/
│   ├── SerieImmatriculationEntity          (compteur : cleSerie PK, groupeCourant, valeurCourante)
│   └── ImmatriculationDelivreeEntity       (registre : cleUnicite UNIQUE + composants)
├── repository/
│   ├── SerieImmatriculationRepository      (findAndLockByCleSerie @Lock(PESSIMISTIC_WRITE))
│   └── ImmatriculationDelivreeRepository   (existsByCleUnicite, findByDossierId...)
├── service/immatriculation/
│   ├── ServiceGenerationImmatriculation    (interface : attribuer(contexte) -> NumeroImmatriculation)
│   ├── impl/ServiceGenerationImmatriculationImpl   (façade transactionnelle, idempotence, retries)
│   ├── ContexteGenerationImmatriculation   (record : tout ce qui détermine la plaque)
│   ├── NumeroImmatriculation               (value object : composants + format() + cleUnicite())
│   └── strategie/
│       ├── StrategieSerieImmatriculation   (interface : supporte(cat) / prochainNumero(ctx))
│       ├── StrategieSerieOrdinaire         (A, B, C, P, T, PRIVE — compteur national)
│       ├── StrategieSerieDiplomatique      (CD, CC, IN — compteur par mission, bandes)
│       ├── StrategieChefMissionDiplomatique(CMD — sans compteur, unicité par mission)
│       └── AlphabetSeriePrivee             (utilitaire : ordre des groupes privés, avancement)
└── enums (sigatt.commons.enums)
    ├── CategorieImmatriculation            (ETAT, COLLECTIVITE, PARAPUBLIC, POLICE, TRANSPORT_PUBLIC,
    │                                        PRIVE, CD_SERVICE, CD_PERSONNEL, CC_SERVICE, CC_PERSONNEL,
    │                                        IN_SERVICE, IN_PERSONNEL, CMD, PARTICULIER)
    ├── MentionSerie                        (IT, AT)
    └── TypeSupportPlaque                   (VEHICULE, CYCLE)
```

### 7.2 Résolution de la stratégie — d'où viennent les composants

Le **contexte** est intégralement constructible depuis un dossier d'immatriculation existant
(les FK sont déjà résolues par `DossierImmatriculationRelationHelper`) :

| Donnée du contexte | Source dans le modèle actuel |
|---|---|
| Catégorie (→ stratégie + lettre) | `dossier.statutProprietaire.immatCode` (re-seedé, annexe B) |
| Type de support (véhicule/cycle) | `vehicule.typeVehicule` / genre (catégorie `MOTOCYCLE` → `CYCLE`) |
| Code région `RR` | Résidence du propriétaire (localité → province → région, referentiel) ; repli : région du site d'enrôlement (`dossier.siteEnrollementId`) — **décision à valider, cf. §12** |
| Code mission (CD/CC/CMD) | `dossier.paysMandataire.code` |
| Code organisation (IN) | `dossier.organisationInternationale.code` |
| Mention `IT`/`AT` | `vehicule.regimeDouanier.mentionSerie` |

La façade choisit la stratégie par `strategies.stream().filter(s -> s.supporte(categorie))` —
l'ajout d'une famille (W/WW, PARTICULIER) est une nouvelle classe `@Component`, rien d'autre ne change.

### 7.3 L'algorithme d'allocation (pseudo-code du cœur)

```
attribuer(contexte) :                                    # @Transactional (REQUIRED)
  1. IDEMPOTENCE : si registre.findByDossierId(ctx.dossierId) existe -> retourner l'existant
  2. strategie = résoudre(ctx.categorie)
  3. cleSerie  = strategie.cleCompteur(ctx)              # ex. "PRIVE|VEHICULE" ou "CD|SERVICE|01"
  4. boucle (max 50 tentatives) :
       a. serie = repo.findAndLockByCleSerie(cleSerie)   # SELECT ... FOR UPDATE (création si absente,
                                                         #   avec rattrapage de collision d'init — patron existant)
       b. (groupe, ordre) = strategie.avancer(serie)     # ordre+1 ; si > borne : groupe suivant, ordre = min
       c. serie.groupeCourant = groupe ; serie.valeurCourante = ordre ; save
       d. numero = NumeroImmatriculation(ordre, groupe, ctx.codeSuffixe, ctx.mention, ctx.support)
       e. TENTER insert registre(numero.cleUnicite(), ..., dossierId)
            - succès  -> retourner numero               # le verrou tombe au commit de la transaction appelante
            - violation d'unicité -> continuer la boucle # le numéro existait (legacy/saisie) : on le saute
  5. échec après 50 -> SigattErrorResponse(CONFLICT, "immatriculation.generation.epuisement")
```

Choix transactionnel : la façade s'exécute dans la **transaction de l'appelant** (`REQUIRED`).
Si le traitement du dossier échoue après l'allocation, tout est annulé ensemble — ni plaque orpheline,
ni compteur incrémenté pour rien. Dans le flux bulk, chaque dossier est déjà isolé en
`REQUIRES_NEW` par `SingleImportRunner` (lib-commons) : l'attribution en hérite naturellement.
Le verrou sur la ligne de compteur ne dure que le temps du traitement d'un dossier (millisecondes).

### 7.4 Avancement des groupes (`AlphabetSeriePrivee`)

- Alphabet privé ordonné (18 lettres) : `D E F G H J K L M N Q R S U V X Y Z`
  (exclusions réglementaires : `A B C P T W` réservées, `I O` interdites — art. 27).
- Avancement : `G3` + ordre 9999 → `G4` ordre 0001 → … → `G9` → `H1` → … → `Z9` →
  croisement deux à deux `DD1` … `ZZ9` (mécanique isolée dans l'utilitaire ; l'ordre exact du
  croisement est une décision DGTTM, cf. §12 — sans urgence : ~1,3 million de numéros restent
  avant `Z9` au rythme actuel).
- Lettres fixes (A/B/C/P) : seule la partie chiffre avance (`A1→A2…A9` ; au-delà : décision DGTTM).
- Transport : chiffre `1→99` (`T1…T99`, art. 31).
- Cycles : mêmes règles, groupe **inversé à l'affichage** (`3G`) mais stocké normalisé
  (`groupe='G3'`, `type_support='CYCLE'`) — la clé d'unicité inclut le support via des compteurs séparés.

---

## 8. Diagrammes de séquence

### 8.1 Tirage nominal — dossier privé, régime normal, Ouagadougou

```mermaid
sequenceDiagram
    autonumber
    participant DSI as DossierImmatriculationServiceImpl
    participant GEN as ServiceGenerationImmatriculation
    participant STR as StrategieSerieOrdinaire
    participant SEQ as SerieImmatriculationRepository
    participant REG as ImmatriculationDelivreeRepository
    participant BD as PostgreSQL

    DSI->>GEN: attribuer(contexte[PRIVE, VEHICULE, region=03, dossier=D-42])
    GEN->>REG: findByDossierId(D-42)
    REG-->>GEN: vide (premier passage)
    GEN->>STR: cleCompteur(contexte)
    STR-->>GEN: "PRIVE|VEHICULE"
    GEN->>SEQ: findAndLockByCleSerie("PRIVE|VEHICULE")
    SEQ->>BD: SELECT ... FOR UPDATE
    BD-->>SEQ: ligne verrouillée (groupe=G3, valeur=2088)
    GEN->>STR: avancer(G3, 2088)
    STR-->>GEN: (G3, 2089)
    GEN->>SEQ: save(groupe=G3, valeur=2089)
    GEN->>REG: insert(cleUnicite="G3|2089", numero="2089 G3 03", dossier=D-42)
    REG->>BD: INSERT (contrainte UNIQUE ok)
    GEN-->>DSI: NumeroImmatriculation "2089 G3 03"
    DSI->>BD: vehicule.immatriculation = "2089 G3 03" puis COMMIT
    Note over BD: le verrou du compteur est relâché au COMMIT
```

### 8.2 Concurrence — deux demandes simultanées sur la même série

```mermaid
sequenceDiagram
    autonumber
    participant T1 as Transaction A (Ouaga)
    participant T2 as Transaction B (Bobo)
    participant BD as PostgreSQL (serie_immatriculation)

    par en parallèle
        T1->>BD: SELECT ... FOR UPDATE ("PRIVE|VEHICULE")
        BD-->>T1: verrou ACQUIS (G3, 2088)
    and
        T2->>BD: SELECT ... FOR UPDATE ("PRIVE|VEHICULE")
        Note over T2,BD: B est BLOQUÉE par le verrou de A (attente)
    end
    T1->>BD: valeur=2089 + INSERT registre "G3|2089" + COMMIT
    Note over BD: verrou relâché
    BD-->>T2: verrou ACQUIS (G3, 2089)
    T2->>BD: valeur=2090 + INSERT registre "G3|2090" + COMMIT
    Note over T1,T2: A obtient 2089 G3 03, B obtient 2090 G3 09.<br/>La sérialisation est assurée par la base,<br/>valable aussi entre plusieurs instances du microservice.
```

### 8.3 Fin de série — passage au groupe suivant

```mermaid
sequenceDiagram
    autonumber
    participant GEN as ServiceGeneration
    participant STR as StrategieSerieOrdinaire
    participant ALP as AlphabetSeriePrivee
    participant BD as PostgreSQL

    GEN->>BD: SELECT ... FOR UPDATE ("PRIVE|VEHICULE")
    BD-->>GEN: (groupe=G3, valeur=9999)
    GEN->>STR: avancer(G3, 9999)
    STR->>ALP: groupeSuivant("G3")
    ALP-->>STR: "G4"
    STR-->>GEN: (G4, 1)
    GEN->>BD: save(groupe=G4, valeur=1) + INSERT registre "G4|0001" + COMMIT
    Note over GEN,BD: l'avancement de groupe se fait DANS la même<br/>transaction verrouillée : aucune course possible,<br/>même si deux demandes arrivent pile à 9999.
```

### 8.4 Collision avec le registre — un numéro legacy devant le compteur

```mermaid
sequenceDiagram
    autonumber
    participant GEN as ServiceGeneration
    participant SEQ as Compteur (verrouillé)
    participant REG as Registre

    GEN->>SEQ: avancer -> (T1, 8762)
    GEN->>REG: INSERT cleUnicite="T1|8762"
    REG-->>GEN: VIOLATION UNIQUE (numéro délivré après l'extraction prod)
    Note over GEN: le numéro existe déjà : on le SAUTE (jamais réattribué)
    GEN->>SEQ: avancer -> (T1, 8763)
    GEN->>REG: INSERT cleUnicite="T1|8763"
    REG-->>GEN: OK
    GEN-->>GEN: retourne "8763 T1 03" (tentatives bornées à 50)
```

### 8.5 Flux complet desktop → synchronisation bulk → plaque (avec rejeu)

```mermaid
sequenceDiagram
    autonumber
    participant DK as Desktop (SQLite, hors-ligne)
    participant GW as API Gateway
    participant BULK as POST /dossier-immatriculation/bulk
    participant RUN as SingleImportRunner (REQUIRES_NEW / dossier)
    participant GEN as ServiceGeneration
    participant REG as Registre

    Note over DK: saisie hors-ligne, numéro de DOSSIER local<br/>(site+machine+jour+seq). AUCUNE plaque générée.
    DK->>GW: push lot de 20 dossiers SUBMITTED
    GW->>BULK: bulk(dossiers)
    loop pour chaque dossier
        BULK->>RUN: importSingle(dossier)
        RUN->>GEN: attribuer(contexte du dossier)
        GEN->>REG: findByDossierId(...)
        alt premier traitement
            GEN->>GEN: allocation sous verrou (cf. 8.1)
            GEN-->>RUN: plaque "2091 G3 09"
        else rejeu (coupure réseau, lot renvoyé)
            REG-->>GEN: plaque déjà attribuée à ce dossier
            GEN-->>RUN: MÊME plaque "2091 G3 09" (idempotence)
        end
        RUN-->>BULK: succès {id, numero, immatriculation}
    end
    BULK-->>DK: BulkSyncResultDto (succès + rejets détaillés)
    Note over DK: extension de contrat : DossierSyncSuccess porte<br/>la plaque -> affichable sur le poste de saisie.
```

### 8.6 Mutation de région — recomposition sans nouveau tirage (art. 3)

```mermaid
sequenceDiagram
    autonumber
    participant BO as Back-office
    participant GEN as ServiceGeneration
    participant REG as Registre

    BO->>GEN: muterRegion(vehicule["2089 G3 03"], nouvelleRegion=09)
    GEN->>REG: retrouver la ligne cleUnicite="G3|2089"
    Note over GEN: la clé d'unicité (G3, 2089) ne change PAS :<br/>aucun verrou de compteur, aucun tirage.
    GEN->>REG: codeSuffixe=09, numeroComplet="2089 G3 09"
    GEN-->>BO: "2089 G3 09" (vehicule.immatriculation mis à jour)
    Note over REG: "2089 G3 03" n'est PAS libéré pour autant :<br/>(G3, 2089) reste consommé à vie.
```

---

## 9. Exemples concrets — ce que produira l'algorithme

État d'amorçage supposé (repris de l'extraction prod) : `PRIVE|VEHICULE=(G3, 2088)`,
`TRANSPORT|VEHICULE=(T1, 8761)`, `ETAT|VEHICULE=(A1, 4237)`, `CD|SERVICE|01=(CD, 44)`,
`CD|PERSONNEL|01=(CD, 1022)`, `IN|SERVICE|27=(IN, 308)`.

| # | Scénario | Catégorie résolue | Compteur sollicité | Plaque générée |
|---|---|---|---|---|
| 1 | M. Ouédraogo, particulier à **Ouagadougou** (région 03), berline, mise à la consommation | `PRIVE` | `PRIVE\|VEHICULE` : 2088→2089 | **`2089 G3 03`** |
| 2 | Mme Sanou, particulière à **Bobo-Dioulasso** (région 09), 1 seconde plus tard | `PRIVE` | même compteur : 2089→2090 | **`2090 G3 09`** — même série : le compteur est national, seule la région diffère |
| 3 | Société privée à Kaya (région 05), camionnette **en admission temporaire** | `PRIVE` + mention | même compteur : 2090→2091 | **`2091 G3 05 AT`** — la mention s'ajoute, le numéro vient du même espace |
| 4 | **Ministère** (statut EPE), véhicule de projet **en franchise temporaire**, Ouaga | `ETAT` | `ETAT\|VEHICULE` : 4237→4238 | **`4238 A1 03 IT`** |
| 5 | **Taxi** (transporteur public) à Ouaga | `TRANSPORT_PUBLIC` | `TRANSPORT\|VEHICULE` : 8761→8762 | **`8762 T1 03`** |
| 6 | **Ambassade de France** (pays mandataire code 01), véhicule de **service** | `CD_SERVICE` | `CD\|SERVICE\|01` : 44→45 | **`0045 CD 01`** |
| 7 | **Diplomate** de cette même ambassade (véhicule personnel) | `CD_PERSONNEL` | `CD\|PERSONNEL\|01` : 1022→1023 | **`1023 CD 01`** — bande personnel : 1001-9999 |
| 8 | **UEMOA** (organisation code 27), véhicule de service | `IN_SERVICE` | `IN\|SERVICE\|27` : 308→309 | **`0309 IN 27`** |
| 9 | **Chef de mission diplomatique** du Ghana (code 02) | `CMD` | aucun compteur | **`02 CMD`** — un seul véhicule par mission ; un second tirage pour la mission 02 est REFUSÉ (contrainte `CMD\|02`) |
| 10 | **Moto** d'un particulier à Ouahigouya (région 10) | `PRIVE` (cycle) | `PRIVE\|CYCLE` (compteur distinct, amorçage à valider) | **`0001 1D 10`** — chiffre avant la lettre |
| 11 | La série privée atteint **9999** (`G3` pleine) | `PRIVE` | avancement dans la même transaction | `9999 G3 xx` puis **`0001 G4 xx`** |
| 12 | Le véhicule de l'exemple 1 **déménage à Bobo** | mutation (art. 3) | aucun tirage | `2089 G3 03` → **`2089 G3 09`** (recomposition ; `(G3, 2089)` reste consommé) |
| 13 | Le véhicule de l'exemple 1 est **racheté par un taxi** (changement de statut) | ré-immatriculation | `TRANSPORT\|VEHICULE` : 8762→8763 | **`8763 T1 03`** — nouveau tirage complet ; l'ancienne plaque n'est JAMAIS recyclée |
| 14 | Rejeu du lot bulk contenant le dossier de l'exemple 2 (coupure réseau) | idempotence | aucun tirage | **`2090 G3 09`** — la même plaque est renvoyée, pas de double consommation |

---

## 10. Amorçage et migration du legacy

1. **Import du registre** : job d'import de `prod.txt` (353 k lignes) → `immatriculation_delivree`
   avec `source = MIGREE`. Normalisation à l'import : dédoublonnage (42 705 rééditions), correction
   des 4 inversions de saisie connues (`1D`→ cycle assumé ou correction, à arbitrer), parsing des
   mentions IT/AT et du format court CMD. Import **idempotent** (`ON CONFLICT (cle_unicite) DO NOTHING`),
   exécutable en plusieurs fois.
2. **Amorçage des compteurs** : pour chaque compteur, `valeur_courante = max observé` + **marge de
   sécurité paramétrable** (clé `immatriculation.amorcage.marge` dans la table `parametre`, défaut
   proposé : 50) — l'extraction a une date, des numéros ont pu être délivrés depuis. La couche 3
   (registre) rattrape de toute façon tout numéro manqué.
3. **Rapprochement final** : avant l'activation en production, ré-extraire le legacy à date et rejouer
   l'import (idempotent) + recaler les compteurs. L'activation elle-même est gardée par un paramètre
   (`immatriculation.generation.active`), désactivable sans redéploiement.

---

## 11. Validation des plaques saisies (INTERNE / bulk / plaque précédente)

Toute plaque **entrée** dans le système (au lieu d'être générée) est validée puis enregistrée au registre.
Expressions régulières par famille (dérivées des art. 9-36 ; `RR` = `0[1-9]|1[0-3]`) :

| Famille | Regex (véhicule) |
|---|---|
| Privés | `^\d{4} [DEFGHJKLMNQRSUVXYZ][1-9] (0[1-9]\|1[0-3])( (IT\|AT))?$` |
| État / Collectivités / Parapublics / Police | `^\d{4} [ABCP][1-9] (0[1-9]\|1[0-3])( (IT\|AT))?$` |
| Transporteurs publics | `^\d{4} T[1-9]\d? (0[1-9]\|1[0-3])$` |
| Diplomatiques / consulaires / internationaux | `^\d{4} (CD\|CC\|IN) \d{2}$` (+ contrôle de bande service/personnel selon le statut) |
| Chef de mission | `^\d{2} CMD$` |
| Cycles | mêmes règles avec groupe inversé, ex. `^\d{4} [1-9][DEFGHJKLMNQRSUVXYZ] (0[1-9]\|1[0-3])( (IT\|AT))?$` |

En cas de non-conformité : `SigattErrorResponse(VALIDATION_FAILED)` avec clé i18n
`immatriculation.format.invalide` (+ variantes par famille), selon le modèle d'erreurs du socle.

---

## 12. Décisions à faire valider avant implémentation

| # | Question | Proposition par défaut |
|---|---|---|
| 1 | **Codes région 01-13** : la liste n'est dans aucun des textes fournis (mapping alphabétique déduit et corroboré par la prod : 03=Centre, 09=Hauts-Bassins). Position sur les régions créées après 2022 ? | Seed des 13 codes historiques (annexe A), à confirmer par la DGTTM |
| 2 | **Arrêté conjoint des codes mission** (art. 44) pour CD/CC/CMD/IN : à obtenir pour figer les référentiels `pays_mandataire` / `organisation_internationale` (prod observée jusqu'au code 64) | Conserver les codes seedés actuels (validés contre prod) |
| 3 | **Moment de l'attribution** : le dossier n'a pas encore de workflow (`statut`). Attribuer à la création consommerait des numéros pour des demandes avortées | Attribution à l'étape de **validation** du dossier (quand le workflow existera) ; en attendant, moteur exposé comme service interne + point d'appel unique |
| 4 | **Source du code région** : résidence du propriétaire (localité→région) ou site d'enrôlement ? | Résidence déclarée du propriétaire ; repli sur la région du site |
| 5 | **Ordre exact du croisement deux lettres** après `Z9` (art. 27 : « croisement deux à deux », exemple `DE2`) | Isolé dans `AlphabetSeriePrivee` ; sans urgence (~1,3 M de numéros de marge) |
| 6 | **Cycles/motos** : 4 lignes seulement dans prod — le parc moto est-il hors extraction ? Amorçage du compteur `PRIVE\|CYCLE` ? | Demander une extraction motos à la DGTTM avant activation des cycles |
| 7 | **Périmètre W/WW et immatriculations particulières** (`GOUVERNEUR 01`) | Hors phase 1 ; le pattern Strategy les accueillera sans refonte |
| 8 | **Retour de la plaque au desktop** : étendre `DossierSyncSuccess {id, numero}` d'un champ `immatriculation` + colonne d'affichage côté desktop | À planifier avec l'équipe desktop (rétrocompatible) |
| 9 | **Correction du seed `immat_code`** (annexe B) : impacte un référentiel déjà en dev | À coordonner (données existantes éventuelles) |
| 10 | **Fraîcheur de `prod.txt`** : date d'extraction et procédure de ré-extraction avant bascule | Cf. §10.3 |

---

## 13. Plan de construction (conventions du projet)

**Phase 1 — le moteur (indépendant du workflow)**
1. Enums dans `sigatt.commons.enums` : `CategorieImmatriculation`, `MentionSerie`, `TypeSupportPlaque`.
2. `domain/SerieImmatriculationEntity` + `domain/ImmatriculationDelivreeEntity`
   (cette dernière `extends AbstractAuditEntity`, `@Table` + contrainte unique).
3. `repository/SerieImmatriculationRepository` (`findAndLockByCleSerie` — copie du patron
   `DossierImmatriculationSequenceRepository`) + `repository/ImmatriculationDelivreeRepository`
   (finders `...AndDeletedFalse` conformément à la règle soft-delete du projet).
4. Package `service/immatriculation/` : value object, contexte, interface + façade
   `@Service @Transactional @Slf4j` (méthodes publiques non-`final` — services proxifiés),
   stratégies `@Component`, `AlphabetSeriePrivee`.
5. i18n `immatriculation.*` (fr + en) ; Javadoc française ; checkstyle vert.

**Phase 2 — les données**
6. Correction du seed `08_statut_proprietaire.sql` (`immat_code`, annexe B).
7. Seed `region` (referentiel-service, annexe A) + colonne/seed `regime_douanier.mention_serie`.
8. Seed `type_vehicule.nombre_plaque` (2 auto / 1 moto-remorque, décret art. 4).
9. Job d'import `prod.txt` → registre + amorçage compteurs (idempotent, rejouable).

**Phase 3 — le branchement**
10. Contrainte `unique` sur `vehicule.immatriculation` ; validation regex des saisies +
    enregistrement au registre.
11. Point d'appel : intégration à `DossierImmatriculationServiceImpl` (à l'étape de validation
    du futur workflow — décision #3) ; extension du retour bulk (décision #8).
12. Rôles Keycloak (`immatriculation-core-roles.json`) si un endpoint d'attribution manuelle est exposé.

**Tests (mêmes patrons que l'existant)**
- Unitaires : `AlphabetSeriePriveeTest` (avancements, exclusions, croisement), stratégies (formats,
  bandes, bornes), regex de validation.
- Intégration (`extends ImmatriculationApplicationTests`) : tirage nominal par catégorie ;
  **test de concurrence** (10 tirages / 5 threads → 10 numéros distincts consécutifs — copie de
  `testCreateConcurrentNominalCases`) ; collision registre (numéro pré-inséré → saut) ;
  idempotence (double appel même dossier → même plaque) ; fin de série (9999 → groupe suivant) ;
  mutation de région ; CMD en doublon refusé.
- Build : `.\mvnw.cmd -pl sigatt-immatriculation-service -am clean install`.

---

## Annexe A — Codes des 13 régions (à faire confirmer, cf. décision #1)

| Code | Région | Chef-lieu | | Code | Région | Chef-lieu |
|---|---|---|---|---|---|---|
| 01 | Boucle du Mouhoun | Dédougou | | 08 | Est | Fada N'Gourma |
| 02 | Cascades | Banfora | | **09** | **Hauts-Bassins** | **Bobo-Dioulasso** |
| **03** | **Centre** | **Ouagadougou** | | 10 | Nord | Ouahigouya |
| 04 | Centre-Est | Tenkodogo | | 11 | Plateau-Central | Ziniaré |
| 05 | Centre-Nord | Kaya | | 12 | Sahel | Dori |
| 06 | Centre-Ouest | Koudougou | | 13 | Sud-Ouest | Gaoua |
| 07 | Centre-Sud | Manga | | | | |

(Liste absente des textes fournis ; ordre alphabétique standard, corroboré par la prod : 03 ≈ 79 %
des immatriculations = Centre/Ouagadougou, 09 ≈ 11 % = Hauts-Bassins/Bobo-Dioulasso.)

## Annexe B — Correction du seed `statut_proprietaire.immat_code`

Le seed actuel est une énumération alphabétique naïve (A→N) sans valeur réglementaire.
Valeurs cibles (= `CategorieImmatriculation`, pilotes des stratégies) :

| code | libellé | immat_code actuel | **immat_code cible** | Série réglementaire |
|---|---|---|---|---|
| EPE | Établissements publics de l'État (EPA) | A | **ETAT** | `A1-A9` (art. 9) |
| COLLECT | Collectivités territoriales | E | **COLLECTIVITE** | `B1-B9` (art. 12) |
| PARAPUB | Organismes parapublics | F | **PARAPUBLIC** | `C1-C9` (art. 18) |
| POLICE | Police nationale | D | **POLICE** | `P1-P9` (art. 15) |
| PUB | Transporteurs publics routiers | N | **TRANSPORT_PUBLIC** | `T1-T99` (art. 31) |
| STD | Privé | B | **PRIVE** | lettres hors `A,B,C,I,O,P,T,W` (art. 27) |
| DIPL_MISS | Mission diplomatique | G | **CD_SERVICE** | `CD`, bande 0001-0999 (art. 21) |
| DIPL_STAFF | Personnel diplomatique | H | **CD_PERSONNEL** | `CD`, bande 1001-9999 (art. 22) |
| CONS_MISS | Mission consulaire | K | **CC_SERVICE** | `CC`, bande 0001-0999 (art. 21) |
| CONS_STAFF | Personnel consulaire | L | **CC_PERSONNEL** | `CC`, bande 1001-9999 (art. 22) |
| INT_MISS | Organisation internationale | I | **IN_SERVICE** | `IN`, bande 0001-0999 (art. 24) |
| INT_STAFF | Personnel org. internationale | J | **IN_PERSONNEL** | `IN`, bande 1001-9999 (art. 25) |
| DIPL_MAIN | Chef de mission diplomatique | M | **CMD** | `NN CMD` (art. 20) |
| SPECIAL | Immatriculation particulière | C | **PARTICULIER** | sigle institution (art. 42, hors phase 1) |

Rappel réglementaire (art. 23/26) : le personnel diplomatique/consulaire **sans statut de diplomate**
et le personnel national sont immatriculés comme des **personnes privées** — le statut saisi doit
alors être `STD`, pas `DIPL_STAFF`.

## Annexe C — Alphabet des séries privées

Lettres autorisées, dans l'ordre : `D E F G H J K L M N Q R S U V X Y Z` (18 lettres).
Exclusions : `A B C P T` (réservées), `W` (provisoires), `I O` (confusion avec 1 et 0).
Capacité d'un groupe : 9 999 numéros ; d'une lettre : 89 991 ; de l'alphabet simple restant
(depuis `G3`) : ≈ 1,26 million de numéros avant le croisement deux lettres.
