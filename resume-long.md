# Comment sont générés les numéros d'immatriculation (voitures et motos) — Document explicatif

> Document **non technique**, destiné aux échanges métier (DGTTM, équipe projet, partenaires).
> Il explique comment le système SIGATT attribuera les numéros d'immatriculation : les règles
> appliquées, les référentiels consultés, les vérifications effectuées, et des exemples concrets.
>
> **Sur quoi s'appuie ce document** :
> - le **décret n°2017-0114** du 17 mars 2017 (modalités d'immatriculation, 52 articles) ;
> - l'**arrêté conjoint n°2017-0101** du 21 juillet 2017 (normes des plaques) ;
> - l'analyse de **1,4 million de numéros réels** du système actuel : 352 922 voitures (`prod.txt`)
>   et 1 048 575 motos (`prod-moto.txt`) — chaque règle énoncée ici a été **vérifiée contre ces
>   données réelles**, pas seulement lue dans les textes.
>
> La version technique (architecture, base de données, diagrammes) est dans
> `strategie-generation-numero-immatriculation.md`.

---

## 1. Avant tout : ne pas confondre les deux numéros

Le système manipule deux numéros très différents, et la confusion est fréquente :

- **Le numéro de DOSSIER** (ex. `OUAG 0000 210726 0001`) : c'est le numéro de la **demande**,
  comme un numéro de ticket. Il est créé dès la saisie (au guichet ou sur l'application desktop),
  sert à suivre le dossier, et n'a aucune existence légale sur le véhicule.
- **Le numéro d'IMMATRICULATION** (ex. `2089 G3 03`) : c'est l'**identité du véhicule**, celle qui
  figure sur la plaque et la carte grise. Il est réglementé par décret, unique au niveau national,
  et attribué **une seule fois pour toujours**.

Ce document ne parle que du second.

---

## 2. L'anatomie d'un numéro, caractère par caractère

### 2.1 Le cas général

```
      2089        G3          03          IT
   ┌─────────┬──────────┬───────────┬───────────┐
   │ numéro  │  série   │  région   │  mention  │
   │ d'ordre │          │           │(optionnelle)│
   └─────────┴──────────┴───────────┴───────────┘
```

Prenons `2091 G3 05 AT` et lisons-le comme le fera le système :

- **`2091`** : le numéro d'ordre. C'est la position dans la file d'attente nationale de la série
  `G3` — ce véhicule est le 2 091ᵉ immatriculé dans cette série. Toujours 4 chiffres, de `0001`
  à `9999`. Quand `9999` est atteint, la série est pleine et la suivante s'ouvre.
- **`G3`** : la série. Elle raconte **qui est le propriétaire** (ici : une personne privée —
  particulier ou entreprise). La lettre avance très lentement au fil des années : l'ancien système
  a rempli `D1, D2… D9`, puis `E1… E9`, `F1… F9`, `G1`, `G2`, et travaille aujourd'hui dans `G3`.
- **`05`** : le code de la région de **résidence du propriétaire** (05 = Centre-Nord, Kaya).
  Attention : ce code est **une étiquette, pas un compteur** — la numérotation ne recommence pas
  à zéro dans chaque région (voir règle n°2).
- **`AT`** : la mention douanière — ce véhicule est en **admission temporaire** (exonération de
  droits de douane). Sans mention particulière, rien n'est affiché.

### 2.2 La lecture change pour les plaques diplomatiques

Pour `0045 CD 01` :

- **`CD`** = corps diplomatique (une ambassade). Il existe aussi `CC` (consulats), `IN`
  (organisations internationales) et `CMD` (chef de mission diplomatique).
- **`01`** n'est **pas une région** : c'est le **code de l'ambassade** (01 = France). Ces codes
  sont fixés par arrêté conjoint des ministères des Transports et des Affaires étrangères.
- **`0045`** : 45ᵉ véhicule de **service** de l'ambassade de France. La plage du numéro d'ordre
  a un sens : de `0001` à `0999` = véhicules de service de la mission ; de `1001` à `9999` =
  véhicules **personnels des diplomates**. On sait donc, rien qu'en lisant `1023 CD 01`, qu'il
  s'agit du véhicule personnel d'un diplomate en poste à l'ambassade de France.

### 2.3 Les motos se lisent à l'envers

Une moto s'écrit **chiffre d'abord** : `0042 5M 03` est une moto (série `5M`), là où
`0042 M5 03` serait une voiture. C'est voulu par le décret : l'ordre des caractères suffit à
distinguer une plaque de moto d'une plaque de voiture. Les motos ont leur **propre compteur**,
totalement séparé de celui des voitures. Le parc moto étant énorme (plus d'un million de numéros,
trois fois le parc voitures), il a déjà épuisé toutes les lettres simples et utilise des
**doubles lettres** : `1DD`, `2DE`, etc.

---

## 3. Qui obtient quelle série — les catégories de propriétaires

C'est le **statut du propriétaire** qui détermine la série. Le décret prévoit aussi une **couleur
de plaque** par catégorie, ce qui permet de vérifier visuellement la cohérence :

| Statut du propriétaire | Série | Exemple | Couleur de plaque |
|---|---|---|---|
| État et établissements publics administratifs | `A1` → `A9` | `4238 A1 03` | blanc sur fond rouge |
| Collectivités territoriales (mairies, régions) | `B1` → `B9` | `0358 B1 06` | rouge sur fond blanc |
| Organismes parapublics et sociétés d'État | `C1` → `C9` | `1391 C1 03` | bleu sur fond blanc |
| Police nationale | `P1` → `P9` | `0454 P1 03` | blanc sur fond violet |
| Transporteurs publics routiers et taxis | `T1` → `T99` | `8762 T1 03` | blanc sur fond bleu |
| **Personnes privées** (particuliers, entreprises, y compris transporteurs pour compte propre) | toute lettre **sauf** `A B C I O P T W`, en commençant par `D` | `2089 G3 03` | noir sur fond jaune citron |
| Ambassades (véhicules de service) | `CD` (n° 0001-0999) | `0045 CD 01` | orange sur fond vert |
| Diplomates (véhicules personnels) | `CD` (n° 1001-9999) | `1023 CD 01` | orange sur fond vert |
| Consulats et personnel consulaire | `CC` (mêmes plages) | `0012 CC 31` | orange sur fond vert |
| Organisations internationales (service / personnel) | `IN` (mêmes plages) | `0309 IN 27` | orange sur fond vert |
| Chef de mission diplomatique | `CMD` | `01 CMD` | orange sur fond vert |
| Garages / concessionnaires (essais, vente) | `W` (provisoire, 3 chiffres) | `001 W1 03` | noir sur fond blanc |
| Sortie de douane | `WW` (provisoire) | `001 WW1 03` (+ `99` si export) | noir sur fond blanc |

Précisions importantes :

- Les lettres **`I` et `O` n'existent jamais** sur une plaque (confusion avec les chiffres 1 et 0).
- Le personnel des ambassades **qui n'a pas le statut de diplomate** (personnel administratif,
  personnel local) est immatriculé comme une **personne privée** — pas en `CD`.
- Le chef de mission diplomatique a droit à **un seul véhicule** sous le format `CMD` ; sa plaque
  est simplement le code de sa mission suivi de `CMD` (le Ghana, code 02, donne `02 CMD`).
- L'**Armée et la Gendarmerie** ne sont pas concernées : le décret les exclut du système.
- Les « immatriculations particulières » (`GOUVERNEUR 01`, `PCR 01`…) existent pour les véhicules
  de fonction de hautes personnalités, sur autorisation expresse du Premier Ministre — elles ne
  passent pas par le générateur automatique.

---

## 4. Les règles d'or, expliquées

**Règle 1 — Un numéro, une fois, pour toujours.**
Un numéro attribué ne sera **jamais** réattribué à un autre véhicule, même si le premier est radié,
détruit ou ré-immatriculé. C'est ce qui garantit qu'un numéro retrouvé sur une infraction, une
assurance ou un acte de vente désigne toujours, sans ambiguïté, un seul véhicule dans l'histoire.

**Règle 2 — Le compteur est national, la région n'est qu'une étiquette.**
Il n'y a qu'**une seule file d'attente par série pour tout le pays**. Si un véhicule est immatriculé
à Ouagadougou à 10h00 (`2089 G3 03`) et un autre à Bobo-Dioulasso à 10h01, le second reçoit
`2090 G3 09` : le numéro suit, seule l'étiquette de région diffère. C'est vérifié sur les données
réelles : dans toute l'histoire du système, **aucun numéro d'une même série n'existe dans deux
régions différentes**. Conséquence pratique : il ne faut jamais « réserver des plages de numéros »
par région ou par site — cela n'existe pas dans ce système.

**Règle 3 — Le numéro est permanent, sauf trois événements.**
Le numéro reste attaché au véhicule tant que ne changent ni le **statut du propriétaire**, ni sa
**résidence**, ni le **régime douanier**. Trois cas :
- *Simple changement de propriétaire, même statut, même région* (un particulier de Ouaga vend à un
  autre particulier de Ouaga) → **le numéro ne change pas du tout**.
- *Déménagement dans une autre région* → seuls **les 2 chiffres de région changent** ; le numéro
  d'ordre et la série sont conservés (`2089 G3 03` devient `2089 G3 09`). L'ancienne combinaison
  n'est pas « libérée » pour autant.
- *Changement de statut ou de régime douanier* (le véhicule d'un particulier devient un taxi ; une
  admission temporaire est régularisée) → **ré-immatriculation complète** : nouveau tirage dans la
  série de la nouvelle catégorie, l'ancien numéro reste consommé à vie.

**Règle 4 — Les séries avancent toutes seules, dans un ordre fixé.**
Quand une série atteint `9999`, la suivante s'ouvre automatiquement : le chiffre avance d'abord
(`G3 → G4 → … → G9`), puis la lettre (`G9 → H1`), en sautant les lettres réservées ou interdites.
Quand toutes les lettres simples seront épuisées, on passera aux doubles lettres — c'est déjà le
quotidien des motos (`DD`, `DE`, `DF`, `DG`, `DH`…), où l'on voit que c'est la **seconde lettre qui
avance**. Il reste environ 1,26 million de numéros de voitures avant d'en arriver là.

**Règle 5 — Les mentions douanières ne créent pas de série à part.**
Un véhicule en franchise temporaire (`IT`) ou en admission temporaire (`AT`) prend son numéro dans
la **même file** que les autres ; la mention s'ajoute simplement en fin de plaque. Les données le
confirment : un numéro portant `IT` n'existe jamais « sans IT » ailleurs.

---

## 5. Les référentiels que le générateur consulte

Pour composer un numéro, le système lit **six référentiels** (des tables de valeurs administrées
dans SIGATT). Voici ce que chacun apporte, et son état de préparation :

| Référentiel | Rôle dans la génération | État actuel |
|---|---|---|
| **Statut du propriétaire** (14 valeurs : État, Privé, Police, Collectivités, Parapublics, Transporteurs publics, Mission/Personnel diplomatique, Mission/Personnel consulaire, Organisation internationale/son personnel, Chef de mission, Particulière) | Détermine la **série** et, pour les diplomatiques, la **plage service/personnel** | ⚠️ Présent, mais les codes de série enregistrés ne correspondent pas au décret — **à corriger** avant démarrage |
| **Type / genre du véhicule** (véhicule automobile, remorque, semi-remorque, motocycle, tricycle, quadricycle) | Distingue **voiture / moto** (ordre des caractères, choix du compteur) et fixe le **nombre de plaques** à produire : 2 pour une voiture, 1 pour une moto ou remorque | Présent ; le nombre de plaques n'est pas encore renseigné — à compléter |
| **Régions** (13 régions, codes 01 à 13) | Les **2 chiffres de fin** pour les séries ordinaires, d'après la résidence du propriétaire | ⚠️ La table existe mais est **vide** — à alimenter (liste proposée en fin de document) |
| **Pays mandataires** (~61 pays, chacun avec un code : France=01, Ghana=02, …) | Le **code de mission** pour les plaques `CD`, `CC` et `CMD` | Présent et **validé contre les données réelles** |
| **Organisations internationales** (~65 organisations : PNUD=01, UEMOA=27, HCR=35, …) | Le **code d'organisation** pour les plaques `IN` | Présent et **validé contre les données réelles** (ex. les 308 véhicules `IN 27` du système actuel = l'UEMOA, dont le siège est à Ouagadougou) |
| **Régimes douaniers** (mise à la consommation, franchise temporaire, admission temporaire…) | La **mention** de fin de plaque : rien, `IT` ou `AT` | ⚠️ La table existe mais est vide — à alimenter avec la correspondance régime → mention |

À noter : un même dossier de demande contient déjà toutes ces informations (statut du propriétaire,
véhicule et son type, adresse de résidence, pays ou organisation le cas échéant, régime douanier).
**Aucune saisie supplémentaire n'est nécessaire** pour générer le numéro.

---

## 6. Le parcours d'une attribution — ce qui est vérifié, dans l'ordre

Quand un dossier arrive à l'étape d'immatriculation, le système déroule ce parcours :

1. **Ce dossier a-t-il déjà reçu un numéro ?**
   Oui → on lui redonne **exactement le même** et on s'arrête là. (C'est la protection contre les
   doubles envois : une coupure réseau qui fait repartir le même dossier ne consommera jamais un
   deuxième numéro.)
2. **Le statut du propriétaire est-il renseigné et reconnu ?**
   Non → refus avec message explicite. Oui → il donne la catégorie (série).
3. **Le type de véhicule est-il renseigné ?**
   Non → refus. Oui → il détermine voiture ou moto (donc le compteur à utiliser) et le nombre de
   plaques à fabriquer.
4. **Selon la catégorie, l'information de fin de plaque est-elle disponible ?**
   - Catégorie ordinaire → une **région** doit être déterminable à partir de la résidence du
     propriétaire (à défaut, celle du site d'enrôlement).
   - Catégorie diplomatique → le **pays** (`CD`/`CC`/`CMD`) ou l'**organisation** (`IN`) est
     **obligatoire** ; son code devient la fin de plaque.
   Manquant → refus.
5. **Cas particulier du chef de mission** : sa mission a-t-elle déjà sa plaque `CMD` ?
   Oui → refus (un seul véhicule autorisé).
6. **Tirage du numéro** : le système prend le **prochain numéro de la file** de la série concernée.
   Si ce numéro figure déjà au registre national (par exemple un numéro délivré par l'ancien système
   après la date de reprise des données), il est **sauté automatiquement** et le suivant est essayé.
7. **Composition finale** : numéro d'ordre + série (inversée pour une moto) + région ou code de
   mission + mention douanière éventuelle. Le numéro est inscrit au **registre national**, attaché
   au véhicule et au dossier, et renvoyé au demandeur (y compris au poste desktop qui a saisi la
   demande, lors de la synchronisation).

Cas complémentaire — **une plaque saisie à la main** (reprise d'un véhicule déjà immatriculé,
mutation entrante) : elle n'est pas générée mais **contrôlée** (le format doit correspondre à sa
catégorie) puis **inscrite au registre national**, pour être protégée contre tout doublon futur au
même titre qu'une plaque générée.

---

## 7. Voitures et motos : deux histoires différentes, deux démarrages différents

L'analyse des données réelles a révélé que l'ancien système ne gère pas les deux parcs de la
même façon — et cela change la manière de démarrer SIGATT :

**Les voitures : une file bien tenue.** Les séries ont été remplies dans l'ordre, complètement,
l'une après l'autre (`D1` pleine, puis `D2`… jusqu'à `G2` pleine). La série en cours est `G3`,
arrêtée au numéro **2088** dans l'extraction fournie.
→ **SIGATT continue simplement la file** : le prochain numéro voiture sera `2089 G3 …`
(avec une marge de sécurité, car des numéros ont pu être délivrés depuis l'extraction).

**Les motos : des plaques fabriquées d'avance.** Côté motos, les numéros ne se suivent pas :
environ 135 séries sont remplies « en gruyère », à un tiers chacune, avec des numéros éparpillés
partout. C'est la signature d'un circuit de **plaques pré-fabriquées par lots** puis écoulées au
fil de l'eau par les guichets et les concessionnaires. Conséquence redoutable : **un numéro moto
absent des données n'est pas un numéro libre** — la plaque physique existe peut-être déjà,
en stock, et sera posée sur une moto le mois prochain.
→ **SIGATT ne reprend PAS la file des motos là où elle semble s'arrêter.** Le compteur moto démarre
sur une **série neuve, jamais touchée par l'ancien système** : la série `DJ`. La toute première
moto immatriculée par SIGATT recevra `0001 1DJ …`. Toutes les anciennes séries motos sont
considérées comme intégralement consommées ; une plaque de stock posée après la bascule sera
enregistrée au registre comme une plaque existante, sans conflit possible.

| En résumé | Voitures | Motos |
|---|---|---|
| Écriture | `2089 G3 03` (lettre d'abord) | `0042 5M 03` (chiffre d'abord) |
| Compteur | File nationale voitures | File nationale motos, **séparée** |
| Ancien système | Séquentiel et propre, front à `G3` n° 2088 | Éparpillé (stocks pré-imprimés), doubles lettres déjà atteintes (`DD`…`DH`) |
| Démarrage SIGATT | **Continuité** : `2089 G3 …` | **Série neuve** : `0001 1DJ …` |
| Plaques produites | 2 (avant + arrière) | 1 |

---

## 8. Exemples détaillés

### 8.1 Trois récits pas à pas

**« M. Ouédraogo achète sa première voiture. »**
Particulier (statut Privé), résidant à Ouagadougou (région Centre = 03), berline neuve dédouanée
normalement. Parcours : dossier jamais servi ✓ → statut Privé → série en cours `G3` → voiture →
compteur voitures → région 03 → pas de mention. La file est à 2088, il reçoit **`2089 G3 03`**,
et deux plaques sont à produire. Sa carte grise porte ce numéro à vie.

**« L'ambassade de France renouvelle un véhicule de service, et un diplomate achète le sien. »**
Deux dossiers le même jour, tous deux rattachés au pays France (code 01). Le premier a le statut
*Mission diplomatique* → plage service (0001-0999) : la file de l'ambassade de France est à 44,
il reçoit **`0045 CD 01`**. Le second a le statut *Personnel diplomatique* → plage personnel
(1001-9999) : sa file est à 1022, il reçoit **`1023 CD 01`**. Les deux plaques sont orange sur
fond vert. Un troisième dossier, déposé par la même ambassade pour la voiture de fonction de
l'ambassadeur, passe par le format spécial : **`01 CMD`** — et si une plaque `01 CMD` existe déjà,
le dossier est refusé, car le chef de mission n'a droit qu'à un véhicule.

**« La demande moto de Ouahigouya passe deux fois. »**
Une moto est saisie sur l'application desktop d'un guichet de Ouahigouya (région Nord = 10),
hors connexion. À la synchronisation, le serveur attribue **`0001 1DJ 10`** — première moto du
nouveau système, série neuve `DJ`, chiffre devant car c'est une moto. La connexion coupe avant que
la réponse n'arrive au guichet ; le poste renvoie le même lot le lendemain. Le serveur reconnaît
le dossier déjà servi et **renvoie le même numéro** `0001 1DJ 10` : aucun numéro n'a été gaspillé,
aucun doublon créé.

### 8.2 Tableau récapitulatif

| # | Situation | Numéro généré | Point illustré |
|---|---|---|---|
| 1 | Particulier à Ouagadougou, voiture | **`2089 G3 03`** | Série privée en cours, région 03 |
| 2 | Particulière à Bobo, une minute après | **`2090 G3 09`** | Compteur national : le numéro suit, la région change |
| 3 | Société de Kaya, camionnette en admission temporaire | **`2091 G3 05 AT`** | La mention douanière s'ajoute, même file |
| 4 | Ministère, véhicule de projet en franchise | **`4238 A1 03 IT`** | Série État `A`, mention `IT` |
| 5 | Mairie de Koudougou, benne à ordures | **`0358 B1 06`** | Série Collectivités `B` |
| 6 | Société d'État, véhicule de direction | **`1391 C1 03`** | Série Parapublics `C` |
| 7 | Police nationale, véhicule d'intervention | **`0454 P1 03`** | Série Police `P` |
| 8 | Taxi à Ouagadougou | **`8762 T1 03`** | Série Transporteurs `T` |
| 9 | Ambassade de France, véhicule de service | **`0045 CD 01`** | Diplomatique, plage service, code pays |
| 10 | Diplomate de cette ambassade | **`1023 CD 01`** | Plage personnel (1001-9999) |
| 11 | UEMOA, véhicule de service | **`0309 IN 27`** | Organisation internationale, code 27 |
| 12 | Chef de mission du Ghana | **`02 CMD`** | Format spécial, un seul véhicule par mission |
| 13 | Première moto du nouveau système (Ouahigouya) | **`0001 1DJ 10`** | Série moto neuve, chiffre devant |
| 14 | Moto suivante (Ouagadougou) | **`0002 1DJ 03`** | Compteur moto national, séparé des voitures |
| 15 | La série `G3` atteint 9999 | **`0001 G4 …`** | Ouverture automatique de la série suivante |
| 16 | Le véhicule n°1 déménage à Bobo | `2089 G3 03` → **`2089 G3 09`** | Seule la région change |
| 17 | Le véhicule n°1 est revendu à un particulier de Bobo | **inchangé** | Même statut, même région : rien ne bouge |
| 18 | Le véhicule n°1 devient un taxi | **`8763 T1 03`** | Changement de statut → ré-immatriculation ; `2089 G3` reste consommé à vie |
| 19 | Renvoi du même dossier (coupure réseau) | **le même numéro** | Un dossier = un numéro, toujours |

---

## 9. Comment on garantit « jamais deux fois le même numéro »

Quatre protections indépendantes, superposées — il faudrait qu'elles échouent **toutes en même
temps** pour produire un doublon :

1. **Un seul guichet d'attribution.** Seul le serveur central attribue des numéros. Les postes de
   saisie (desktop, back-office) envoient des dossiers et **reçoivent** un numéro en réponse ;
   aucun numéro n'est jamais inventé localement. (Vérifié : l'application desktop actuelle ne
   contient aucune logique de plaque.)
2. **Une seule file d'attente à la fois.** Quand deux demandes arrivent au même instant pour la
   même série, la base de données les sert **strictement l'une après l'autre** — la deuxième
   attend son tour quelques millisecondes et reçoit le numéro suivant. Cela reste vrai même si
   plusieurs serveurs tournent en parallèle.
3. **Le registre national.** Chaque numéro attribué est inscrit dans un registre qui contient
   aussi **tout l'historique de l'ancien système** (1,4 million de numéros repris). Ce registre
   **refuse physiquement** l'inscription d'un numéro déjà présent : même en cas de bug ou de
   reprise de données incomplète, un numéro déjà pris ne peut pas ressortir — il est sauté et le
   suivant est attribué.
4. **Un dossier = un numéro.** Le numéro est attaché au dossier qui l'a demandé. Si le même dossier
   revient (double clic, coupure réseau, resynchronisation), le système renvoie le numéro déjà
   attribué au lieu d'en tirer un nouveau.

Et une cinquième précaution propre au démarrage : la **reprise de l'existant**. Avant l'activation,
les extractions des deux parcs sont rechargées à jour (celle des motos devra être refaite : le
fichier actuel a été tronqué par la limite du tableur Excel), les compteurs voitures sont calés
au-delà du dernier numéro connu avec une marge de sécurité, et les motos démarrent sur une série
neuve. Les « trous » de l'historique ne sont **jamais comblés**.

---

## 10. Ce qui n'est pas couvert par la première version

- Les plaques **provisoires `W` / `WW`** (véhicules d'essai des garages, sorties de douane,
  mention `99` pour l'export) : prévues par le décret, elles suivront le même moteur, dans une
  version ultérieure. En attendant, le champ « immatriculation provisoire » reste saisi.
- Les **immatriculations particulières** (`GOUVERNEUR 01`, `PCR 01`…) : hors générateur,
  sur autorisation du Premier Ministre.
- L'**Armée et la Gendarmerie** : exclues du dispositif par le décret.
- Les **duplicata** de carte grise : ils ne changent pas le numéro (rééditions du même numéro,
  très fréquentes dans les données historiques).

---

## 11. Les décisions attendues de la DGTTM avant la mise en service

| # | Décision | Pourquoi c'est nécessaire |
|---|---|---|
| 1 | Confirmer la **liste officielle des 13 codes région** (proposition ci-dessous, déduite des textes de décentralisation et confirmée par les données : 03 = Centre concentre 79 % des voitures) | La table des régions est vide ; le code figure sur chaque plaque ordinaire |
| 2 | Fournir l'**arrêté conjoint des codes de missions et organisations** (Transports + Affaires étrangères) | Base légale des fins de plaque `CD`/`CC`/`IN`/`CMD` ; les codes observés vont jusqu'à 64 |
| 3 | Valider la **correction des codes de série** du référentiel Statut du propriétaire | Les valeurs actuellement enregistrées ne correspondent pas au décret (ex. Police enregistrée « D » alors que le décret dit « P ») |
| 4 | Confirmer le **démarrage des motos sur la série neuve `DJ`** et l'hypothèse des stocks de plaques pré-fabriquées | Évite tout conflit avec les plaques en stock ; c'est la conclusion de l'analyse du million de numéros motos |
| 5 | Fournir des **ré-extractions complètes et récentes** des deux parcs (format brut, sans passage par Excel) à la date de bascule | Le fichier motos actuel est tronqué à la limite d'Excel ; les compteurs doivent être calés sur l'état réel du jour J |
| 6 | Arbitrer le **moment de l'attribution** dans le parcours de la demande (recommandation : à la **validation** du dossier, pas à la saisie) | Attribuer trop tôt consommerait des numéros pour des demandes abandonnées |

### Proposition de codes région (à confirmer)

| Code | Région | Chef-lieu | | Code | Région | Chef-lieu |
|---|---|---|---|---|---|---|
| 01 | Boucle du Mouhoun | Dédougou | | 08 | Est | Fada N'Gourma |
| 02 | Cascades | Banfora | | 09 | Hauts-Bassins | Bobo-Dioulasso |
| 03 | Centre | Ouagadougou | | 10 | Nord | Ouahigouya |
| 04 | Centre-Est | Tenkodogo | | 11 | Plateau-Central | Ziniaré |
| 05 | Centre-Nord | Kaya | | 12 | Sahel | Dori |
| 06 | Centre-Ouest | Koudougou | | 13 | Sud-Ouest | Gaoua |
| 07 | Centre-Sud | Manga | | | | |
