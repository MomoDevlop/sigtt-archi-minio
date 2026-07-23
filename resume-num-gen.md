# Comment sont générés les numéros d'immatriculation — Résumé non technique

> Version simplifiée de la stratégie, destinée aux échanges métier (DGTTM, équipe projet).
> Basée sur le décret n°2017-0114, l'arrêté n°2017-0101 et l'analyse de 1,4 million de numéros
> réels du système existant (353 000 voitures + 1 050 000 motos).
> Document technique complet : `strategie-generation-numero-immatriculation.md`.

---

## 1. À quoi ressemble un numéro

```
      2089        G3          03         IT
   ┌─────────┬──────────┬──────────┬──────────┐
   │ numéro  │  série   │  région  │ mention  │
   │ d'ordre │          │          │(optionnel)│
   └─────────┴──────────┴──────────┴──────────┘
```

Chaque partie répond à une question simple sur le dossier :

| Partie | Question posée | Exemple |
|---|---|---|
| **Série** (lettre + chiffre) | **Qui est le propriétaire ?** | Particulier → `G3` · État → `A1` · Taxi → `T1` · Ambassade → `CD` |
| **Ordre des caractères** | **Voiture ou moto ?** | Voiture : lettre d'abord (`G3`) · Moto : chiffre d'abord (`3G`) |
| **Numéro d'ordre** (4 chiffres) | **Où en est le compteur ?** | `0001` → `9999`, puis la série suivante s'ouvre |
| **Région** (2 chiffres) | **Où réside le propriétaire ?** | `03` = Centre (Ouaga) · `09` = Hauts-Bassins (Bobo) |
| **Mention** | **Quel régime douanier ?** | (rien) = normal · `IT` = franchise temporaire · `AT` = admission temporaire |

Cas à part : pour les plaques diplomatiques (`CD`, `CC`, `IN`, `CMD`), les 2 derniers chiffres ne sont
pas une région mais le **code de l'ambassade ou de l'organisation** (ex. `IN 27` = UEMOA).

## 2. Les règles d'or (issues du décret)

1. **Un numéro est unique au niveau national** et attribué **une seule fois, pour toujours**.
   Même radié, un numéro n'est **jamais redonné** à un autre véhicule.
2. Le compteur est **national par série** : deux véhicules immatriculés au même moment à Ouaga et à
   Bobo reçoivent deux numéros qui se suivent dans la même série — seule la région affichée diffère.
3. Le numéro est **permanent** tant que trois choses ne changent pas : le **statut du propriétaire**,
   sa **résidence**, le **régime douanier**.
   - Déménagement dans une autre région → on ne change **que** les 2 chiffres de région.
   - Changement de statut (ex. un particulier vend son véhicule à un taxi) ou de régime douanier
     → **ré-immatriculation complète** (nouveau numéro, l'ancien reste consommé).
4. Quand une série atteint `9999`, on ouvre la suivante : `G3 → G4 → … → G9 → H1 → …`
   Les lettres `A, B, C, P, T, W` sont réservées à des catégories précises, et `I, O` sont interdites
   (confusion avec 1 et 0).
5. Ambassades et organisations : véhicules de **service** numérotés de `0001 à 0999`, véhicules du
   **personnel** de `1001 à 9999`. Le **chef de mission** a un format spécial (`01 CMD`) et
   **un seul véhicule** par mission.

## 3. Les référentiels utilisés pour générer

| Référentiel | Ce qu'il apporte à la génération |
|---|---|
| **Statut du propriétaire** | La série : État=`A`, Collectivités=`B`, Parapublics=`C`, Police=`P`, Transporteurs publics=`T`, Diplomatique=`CD`/`CC`, Organisation internationale=`IN`, Chef de mission=`CMD`, Privé=lettres `D` à `Z` |
| **Type / genre du véhicule** | Voiture ou moto (ordre des caractères) + nombre de plaques à fabriquer (2 pour une voiture, 1 pour une moto ou remorque) |
| **Région** (13 régions, codes 01-13) | Les 2 chiffres de fin, d'après la résidence du propriétaire |
| **Pays mandataire** | Le code de l'ambassade/consulat pour les plaques `CD`/`CC`/`CMD` (ex. France = 01) |
| **Organisation internationale** | Le code de l'organisation pour les plaques `IN` (ex. UEMOA = 27) |
| **Régime douanier** | La mention `IT` ou `AT` (ou rien) |

 Trois référentiels doivent être complétés avant le démarrage : les **codes des 13 régions**
(table vide aujourd'hui), la correspondance **régime douanier → mention**, et la **correction des
codes série** du référentiel Statut du propriétaire (les valeurs actuelles ne correspondent pas au décret).

## 4. Les conditions vérifiées avant chaque attribution

1. Le **statut du propriétaire** est renseigné → il détermine la série. Sinon : refus.
2. Le **type de véhicule** est renseigné → voiture ou moto. Sinon : refus.
3. La **région de résidence** est déterminable (ou le code de mission pour les diplomatiques —
   obligatoire dans ce cas). Sinon : refus.
4. Ce **dossier n'a pas déjà reçu un numéro** → si oui, on lui redonne **le même** (aucun dossier
   ne peut consommer deux numéros, même si la demande est renvoyée deux fois par le réseau).
5. Pour un chef de mission : la mission **n'a pas déjà** sa plaque `CMD`. Sinon : refus.
6. Le numéro candidat **n'existe pas déjà** dans le registre national (qui contient aussi tout
   l'historique de l'ancien système) → s'il existe, on passe au suivant, automatiquement.
7. Si une plaque est **saisie manuellement** (reprise d'un véhicule déjà immatriculé), son format
   est contrôlé et elle est ajoutée au registre national pour être protégée à son tour.

## 5. Voitures et motos : deux compteurs, deux démarrages

|  | **Voitures** | **Motos** |
|---|---|---|
| Écriture | `2089 G3 03` (lettre d'abord) | `0042 5M 03` (chiffre d'abord) |
| Compteur | Compteur national voitures | Compteur national motos, **totalement séparé** |
| Où en est l'ancien système | Séries remplies dans l'ordre jusqu'à **`G3` (n° 2088)** | Séries **éparpillées** : les plaques étaient pré-fabriquées en stock et posées au fil de l'eau (l'alphabet est déjà en lettres doubles : `DD` à `DH`) |
| Démarrage du nouveau système | **On continue la suite** : prochain numéro `2089 G3 …` | **On repart sur une série neuve (`DJ`)** jamais touchée : premier numéro `0001 1DJ …`. Raison : dans les anciennes séries motos, un numéro « libre » peut correspondre à une plaque déjà fabriquée qui dort dans un stock — on ne prend aucun risque |

## 6. Exemples

| Situation | Numéro généré | Pourquoi |
|---|---|---|
| M. Ouédraogo, particulier à Ouagadougou, achète une voiture | **`2089 G3 03`** | Privé → série en cours `G3` ; compteur à 2088 → 2089 ; résidence Centre → 03 |
| Mme Sanou, particulière à Bobo, une minute plus tard | **`2090 G3 09`** | Même compteur national (2090) ; seule la région change |
| Société de Kaya, camionnette en admission temporaire | **`2091 G3 05 AT`** | Même série ; la mention `AT` s'ajoute à la fin |
| Un ministère immatricule un véhicule de projet en franchise | **`4238 A1 03 IT`** | État → série `A` ; mention `IT` |
| Un taxi à Ouagadougou | **`8762 T1 03`** | Transporteur public → série `T` |
| L'ambassade de France, véhicule de service | **`0045 CD 01`** | Diplomatique service (0001-0999) ; France = code 01 |
| Un diplomate de cette ambassade, véhicule personnel | **`1023 CD 01`** | Bande personnel (1001-9999) |
| L'UEMOA, véhicule de service | **`0309 IN 27`** | Organisation internationale ; UEMOA = code 27 |
| Le chef de mission diplomatique du Ghana | **`02 CMD`** | Format spécial, un seul véhicule ; Ghana = code 02 |
| Une moto à Ouahigouya (première du nouveau système) | **`0001 1DJ 10`** | Moto → série neuve `DJ`, chiffre devant ; Nord → 10 |
| La moto suivante, à Ouagadougou | **`0002 1DJ 03`** | Même compteur national motos |
| La série `G3` atteint 9999 | **`0001 G4 …`** | Ouverture automatique de la série suivante |
| Le véhicule de M. Ouédraogo déménage à Bobo | `2089 G3 03` → **`2089 G3 09`** | Seule la région change, pas de nouveau tirage |
| Ce même véhicule est racheté par un taxi | **`8763 T1 03`** | Changement de statut → ré-immatriculation complète ; `2089 G3` reste consommé à vie |

## 7. Comment on garantit « jamais deux fois le même numéro »

Quatre protections superposées, en langage simple :

1. **Un seul guichet** : seul le serveur central attribue les numéros — jamais les postes de saisie
   (l'application desktop envoie les dossiers, le serveur répond avec le numéro).
2. **Une seule file d'attente par série** : quand deux demandes arrivent en même temps, la base de
   données les sert **l'une après l'autre** — la deuxième reçoit automatiquement le numéro suivant.
3. **Le registre national** : chaque numéro attribué (y compris les 1,4 million de l'ancien système)
   est inscrit dans un registre qui **refuse physiquement** tout doublon. Même en cas de bug, un
   numéro déjà pris ne peut pas ressortir : il est sauté.
4. **Un dossier = un numéro** : si le même dossier revient (coupure réseau, renvoi), il reçoit
   **le même** numéro qu'à la première fois.

## 8. Hors périmètre de la première version

- Les plaques **provisoires** `W` / `WW` (garages, sortie de douane) — prévues, non prioritaires.
- Les **immatriculations particulières** (`GOUVERNEUR 01`…) — sur autorisation du Premier Ministre.
- L'Armée et la Gendarmerie ne sont **pas concernées** (exclues par le décret).

## 9. Décisions attendues de la DGTTM

1. Confirmer la **liste officielle des codes des 13 régions** (01=Boucle du Mouhoun … 13=Sud-Ouest).
2. Fournir l'**arrêté des codes des ambassades et organisations** (base des plaques CD/CC/IN/CMD).
3. Confirmer le **démarrage des motos sur la série neuve `DJ`** (et l'hypothèse des stocks pré-imprimés).
4. Fournir des **ré-extractions complètes et à jour** des deux parcs avant la bascule.
5. Valider le **moment de l'attribution** dans le parcours de la demande (à la validation du dossier).
