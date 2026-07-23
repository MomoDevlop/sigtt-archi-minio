# Fiche de validation métier — Génération des numéros d'immatriculation (voitures et motos)

| | |
|---|---|
| **Projet** | SIGATT — Système Intégré de Gestion Automatisée des Titres de Transport |
| **Objet** | Valider les données d'entrée, les référentiels et les règles de génération automatique des numéros d'immatriculation |
| **À valider par** | Acteurs métier DGTTM (immatriculation) |
| **Fondements** | Décret n°2017-0114 du 17/03/2017 · Arrêté n°2017-0101 du 21/07/2017 · 1,4 million de numéros réels analysés (extractions de mars-avril 2024) |
| **Documents de référence** | `resume-regles-generation-immatriculation.md` (explicatif) · `strategie-generation-numero-immatriculation.md` (technique) |

**Mode d'emploi de la fiche** : chaque tableau et chaque règle porte une case
`☐ Validé  ☐ À corriger`. Cocher, et en cas de correction, indiquer la valeur ou la règle attendue
dans la colonne / zone « Commentaire ». La fiche signée vaut validation du paramétrage de départ.

---

## 1. Le principe en cinq lignes

1. Le numéro d'immatriculation est **composé automatiquement** par le système central, à partir
   d'informations **déjà présentes dans le dossier de demande** — aucune saisie supplémentaire.
2. Il se lit : **numéro d'ordre** (4 chiffres) + **série** (qui est le propriétaire) +
   **région ou code de mission** (2 chiffres) + éventuelle **mention douanière** (`IT`/`AT`).
3. Une **voiture** s'écrit lettre d'abord (`2089 G3 03`), une **moto** chiffre d'abord (`0002 1DJ 03`).
4. Le compteur est **national** : les numéros se suivent pour tout le pays, la région n'est
   qu'une étiquette.
5. Un numéro attribué l'est **une seule fois, pour toujours** — jamais réattribué.

---

## 2. Les valeurs d'entrée utilisées (toutes issues du dossier de demande)

| # | Donnée du dossier | Sert à déterminer | Obligatoire ? |
|---|---|---|---|
| E1 | **Statut du propriétaire** | La **série** (`A`, `B`, `C`, `P`, `T`, `CD`, `CC`, `IN`, `CMD`, ou lettres privées) | Toujours |
| E2 | **Type / genre du véhicule** | **Voiture ou moto** (sens d'écriture, compteur utilisé) et le **nombre de plaques** à produire | Toujours |
| E3 | **Région de résidence du propriétaire** | Les **2 chiffres de fin** (plaques ordinaires) | Toujours, sauf catégories diplomatiques |
| E4 | **Pays mandataire** | Le code de fin des plaques **`CD` / `CC` / `CMD`** (ex. France = 01) | Uniquement statuts diplomatiques/consulaires |
| E5 | **Organisation internationale** | Le code de fin des plaques **`IN`** (ex. UEMOA = 27) | Uniquement statuts « organisation internationale » |
| E6 | **Régime douanier** | La **mention** de fin : rien, `IT` (franchise temporaire) ou `AT` (admission temporaire) | Toujours (valeur « normale » par défaut) |

`☐ Validé  ☐ À corriger` — Commentaire : ______________________________________________

---

## 3. Les référentiels et leurs valeurs — à valider ligne par ligne

### 3.1 Statut du propriétaire → série attribuée

| Statut (référentiel SIGATT) | Série attribuée | Particularité | Validation |
|---|---|---|---|
| État et étab. publics administratifs (EPE) | `A1` → `A9` | Plaque blanc sur rouge | ☐ Validé ☐ À corriger |
| Collectivités territoriales (COLLECT) | `B1` → `B9` | Rouge sur blanc | ☐ Validé ☐ À corriger |
| Organismes parapublics (PARAPUB) | `C1` → `C9` | Bleu sur blanc | ☐ Validé ☐ À corriger |
| Police nationale (POLICE) | `P1` → `P9` | Blanc sur violet | ☐ Validé ☐ À corriger |
| Transporteurs publics routiers, taxis (PUB) | `T1` → `T99` | Blanc sur bleu | ☐ Validé ☐ À corriger |
| **Privé** — particuliers et entreprises (STD) | Lettres `D` → `Z` (sauf `I, O` interdites et `A, B, C, P, T, W` réservées) ; série en cours : `G3` | Noir sur jaune citron | ☐ Validé ☐ À corriger |
| Mission diplomatique (DIPL_MISS) | `CD`, numéros **0001 à 0999** | Orange sur vert ; fin = code du pays | ☐ Validé ☐ À corriger |
| Personnel diplomatique (DIPL_STAFF) | `CD`, numéros **1001 à 9999** | Réservé aux diplomates ; le personnel non diplomate est immatriculé en Privé | ☐ Validé ☐ À corriger |
| Mission consulaire (CONS_MISS) | `CC`, numéros 0001 à 0999 | | ☐ Validé ☐ À corriger |
| Personnel consulaire (CONS_STAFF) | `CC`, numéros 1001 à 9999 | | ☐ Validé ☐ À corriger |
| Organisation internationale (INT_MISS) | `IN`, numéros 0001 à 0999 | Fin = code de l'organisation | ☐ Validé ☐ À corriger |
| Personnel org. internationale (INT_STAFF) | `IN`, numéros 1001 à 9999 | | ☐ Validé ☐ À corriger |
| Chef de mission diplomatique (DIPL_MAIN) | `CMD` (format `01 CMD`) | **Un seul véhicule** par mission | ☐ Validé ☐ À corriger |
| Immatriculation particulière (SPECIAL) | Sigle de l'institution (`GOUVERNEUR 01`) | **Hors génération automatique** (autorisation PM) | ☐ Validé ☐ À corriger |

> Vérification effectuée : cette correspondance est celle observée dans les données réelles
> (ex. les véhicules de statut COLLECT sont en série `B` à 98 %, POLICE en `P` à 91 %).

### 3.2 Codes des 13 régions (fin de plaque ordinaire)

Correspondance **vérifiée à 99 %** en croisant, sur un million d'immatriculations réelles, le code
de la plaque et la région de résidence déclarée du propriétaire :

| Code | Région | Validation | | Code | Région | Validation |
|---|---|---|---|---|---|---|
| 01 | Boucle du Mouhoun | ☐ V ☐ C | | 08 | Est | ☐ V ☐ C |
| 02 | Cascades | ☐ V ☐ C | | 09 | Hauts-Bassins | ☐ V ☐ C |
| 03 | Centre | ☐ V ☐ C | | 10 | Nord | ☐ V ☐ C |
| 04 | Centre-Est | ☐ V ☐ C | | 11 | Plateau-Central | ☐ V ☐ C |
| 05 | Centre-Nord | ☐ V ☐ C | | 12 | Sahel | ☐ V ☐ C |
| 06 | Centre-Ouest | ☐ V ☐ C | | 13 | Sud-Ouest | ☐ V ☐ C |
| 07 | Centre-Sud | ☐ V ☐ C | | | | |

**Question associée** : quelle position pour les régions créées après 2022 ?
Réponse métier : ______________________________________________

### 3.3 Codes des pays mandataires et organisations internationales (fin de plaque diplomatique)

Les codes déjà présents dans SIGATT correspondent aux plaques réelles (vérifié :
`IN 27` = UEMOA, `IN 01` = PNUD, `IN 35` = HCR, `CD 01` = France…). La liste complète relève de
l'**arrêté conjoint Transports / Affaires étrangères** — à fournir pour validation finale.

`☐ Validé (codes actuels)  ☐ À corriger` — Commentaire : _______________________________

### 3.4 Régime douanier → mention en fin de plaque

| Régime douanier | Mention ajoutée | Validation |
|---|---|---|
| Mise à la consommation (régime normal) | *(aucune)* | ☐ Validé ☐ À corriger |
| Franchise temporaire | `IT` | ☐ Validé ☐ À corriger |
| Admission temporaire | `AT` | ☐ Validé ☐ À corriger |

> La liste des régimes douaniers de SIGATT est aujourd'hui vide : le métier doit fournir la liste
> des régimes et, pour chacun, la mention applicable (aucune / IT / AT).

### 3.5 Type de véhicule → écriture et nombre de plaques

| Type / genre | Écriture du numéro | Plaques produites | Validation |
|---|---|---|---|
| Véhicule automobile (voiture, camion, bus…) | Lettre d'abord : `2089 G3 03` | **2** (avant + arrière) | ☐ Validé ☐ À corriger |
| Motocycle (vélomoteur, moto, tricycle, quadricycle) | Chiffre d'abord : `0002 1DJ 03` | **1** | ☐ Validé ☐ À corriger |
| Remorque / semi-remorque | Lettre d'abord | **1** | ☐ Validé ☐ À corriger |

---

## 4. Les règles de gestion — à valider une par une

| N° | Règle | Validation |
|---|---|---|
| R1 | Un numéro est attribué **une seule fois, pour toujours** ; il n'est jamais réattribué, même après radiation du véhicule. | ☐ Validé ☐ À corriger |
| R2 | Le compteur est **national par série** : pas de plages réservées par région, par site ou par guichet. Deux immatriculations simultanées à Ouaga et à Bobo reçoivent deux numéros consécutifs. | ☐ Validé ☐ À corriger |
| R3 | Le numéro est **permanent** ; seuls trois événements le modifient : changement de **statut du propriétaire** ou de **régime douanier** → nouveau numéro complet ; changement de **région de résidence** → seuls les 2 chiffres de région changent. | ☐ Validé ☐ À corriger |
| R4 | Un simple **changement de propriétaire** (même statut, même région) ne change pas le numéro. | ☐ Validé ☐ À corriger |
| R5 | Quand une série atteint 9999, la **série suivante s'ouvre automatiquement** (`G3 → G4 → … → G9 → H1`), selon l'alphabet du décret. | ☐ Validé ☐ À corriger |
| R6 | Plaques diplomatiques : numéros **0001-0999 = véhicules de service**, **1001-9999 = véhicules personnels** des diplomates/agents. | ☐ Validé ☐ À corriger |
| R7 | Le **chef de mission diplomatique** a droit à **un seul** véhicule au format `CMD` ; toute seconde demande est refusée. | ☐ Validé ☐ À corriger |
| R8 | **Motos : le nouveau système démarre sur une série neuve (`DJ`)**, jamais utilisée. Motif : les plaques motos de l'ancien système étaient pré-fabriquées par lots — un numéro « libre » peut correspondre à une plaque en stock (fait prouvé par l'analyse des données). **Chiffré par simulation** : reprendre la file existante provoquerait ~8 % de doublons physiques sur les plaques de stock posées après la bascule ; la série neuve en provoque zéro. | ☐ Validé ☐ À corriger |
| R9 | Les **numéros non utilisés** (trous) de l'ancien système ne sont **jamais réattribués**. | ☐ Validé ☐ À corriger |
| R10 | **Un dossier = un numéro** : si un dossier est renvoyé deux fois (incident réseau), il reçoit le même numéro. Le numéro est attribué **quand le validateur SIM valide la demande** sur l'interface web centrale (conforme US-CG-008), pas à la saisie. | ☐ Validé ☐ À corriger |
| R11 | Seul le **système central** attribue les numéros ; les postes de saisie (desktop, guichets) reçoivent le numéro en retour, ils n'en créent jamais. | ☐ Validé ☐ À corriger |
| R12 | L'« immatriculation précédente » d'un dossier peut être au **format d'avant 2017** (ex. `11 AA5302`) : elle est acceptée et conservée telle quelle. | ☐ Validé ☐ À corriger |

---

## 4 bis. Effet de chaque nature de demande sur le numéro — à valider

| N° | Nature de demande | Effet sur le numéro | Validation |
|---|---|---|---|
| N1 | Première mise en circulation | **Nouveau numéro** attribué à la validation | ☐ Validé ☐ À corriger |
| N2 | Renouvellement (papier / polycarbonate) | Numéro conservé — sauf plaque au format pré-2017 (passage au format actuel) ou changement de statut/régime | ☐ Validé ☐ À corriger |
| N3 | Changement d'adresse | Seuls les **2 chiffres de région** changent — pas de tirage | ☐ Validé ☐ À corriger |
| N4 | Changement de propriétaire | Numéro conservé si même statut et même régime douanier ; **nouveau numéro** sinon | ☐ Validé ☐ À corriger |
| N5 | Gage / levée de gage | Numéro conservé | ☐ Validé ☐ À corriger |
| N6 | Duplicata (perte, vol) | Numéro conservé (réédition de la carte) | ☐ Validé ☐ À corriger |
| N7 | Banalisation | **Nouveau numéro en série privée** (plaque banalisée) | ☐ Validé ☐ À corriger |
| N8 | Levée de banalisation | **Nouveau numéro** dans la série du statut réel | ☐ Validé ☐ À corriger |
| N9 | Transformation | Numéro conservé (la série n'est pas modifiable) | ☐ Validé ☐ À corriger |

---

## 5. Exemples de contrôle (à vérifier par le métier)

| Situation présentée au système | Numéro attendu | Conforme ? |
|---|---|---|
| Particulier de Ouagadougou, voiture, régime normal (compteur à 2088) | `2089 G3 03` | ☐ Oui ☐ Non |
| Particulière de Bobo-Dioulasso, juste après | `2090 G3 09` | ☐ Oui ☐ Non |
| Société de Kaya, camionnette en admission temporaire | `2091 G3 05 AT` | ☐ Oui ☐ Non |
| Ministère, véhicule de projet en franchise, Ouaga | `4238 A1 03 IT` | ☐ Oui ☐ Non |
| Taxi à Ouagadougou | `8762 T1 03` | ☐ Oui ☐ Non |
| Ambassade de France, véhicule de service | `0045 CD 01` | ☐ Oui ☐ Non |
| Diplomate de cette ambassade, véhicule personnel | `1023 CD 01` | ☐ Oui ☐ Non |
| UEMOA, véhicule de service | `0309 IN 27` | ☐ Oui ☐ Non |
| Chef de mission du Ghana | `02 CMD` | ☐ Oui ☐ Non |
| Première moto du nouveau système, Ouahigouya | `0001 1DJ 10` | ☐ Oui ☐ Non |
| Le véhicule `2089 G3 03` déménage à Bobo | devient `2089 G3 09` | ☐ Oui ☐ Non |
| Le véhicule `2089 G3 03` devient un taxi | nouveau numéro `8763 T1 03` | ☐ Oui ☐ Non |

*(Les valeurs de compteur sont celles des extractions d'avril 2024, à titre d'illustration ;
elles seront relevées à la date exacte de mise en service.)*

---

## 6. Points ouverts nécessitant une décision métier

| # | Point | Décision / réponse métier |
|---|---|---|
| D1 | Confirmation officielle de la table des 13 codes région (§3.2) et position sur les régions créées après 2022 | |
| D2 | Fourniture de l'**arrêté conjoint** des codes de missions et organisations (§3.3) | |
| D3 | Liste des **régimes douaniers** et mention associée pour chacun (§3.4) | |
| D4 | Confirmation du **démarrage des motos en série `DJ`** (R8) | |
| D5 | **Ré-extractions complètes et récentes** des deux parcs (format brut, sans Excel) à la date de bascule | |
| D6 | Validation du tableau **« Effet de chaque nature de demande »** (§4 bis) — le moment de l'attribution est déjà fixé par US-CG-008 (à la validation) | |

---

## 6 bis. Prérequis avant mise en service — état de fourniture

| # | Prérequis | Porteur | État |
|---|---|---|---|
| P1 | Écran de validation des demandes (l'action « Valider » déclenche l'attribution — US-CG-008) | Équipe projet | ☐ Fourni ☐ En attente |
| P2 | Présente fiche signée (statuts, régions, natures de demande) | DGTTM | ☐ Fourni ☐ En attente |
| P3 | Arrêté conjoint des codes de missions et organisations | DGTTM / MAE | ☐ Fourni ☐ En attente |
| P4 | Liste des régimes douaniers + mention (aucune / IT / AT) | DGTTM / Douanes | ☐ Fourni ☐ En attente |
| P5 | Relevé des derniers numéros à l'**arrêt de l'ancien système** (toutes séries + chaque ambassade/organisation, par requête) + **PV contradictoire signé** — pièce critique : la numérotation SIGATT continue à partir de ces valeurs | DGTTM / exploitant legacy | ☐ Fourni ☐ En attente |
| P6 | Remplissage des tables de paramétrage (13 régions, nombre de plaques, codes de série corrigés) | Équipe projet | ☐ Fourni ☐ En attente |
| P7 | Décision moto : série `DH` (continuité stricte — simulation : ~8 % de doublons physiques sur les stocks) ou série neuve `DJ` (recommandé — simulation : 0 conflit) | DGTTM | ☐ Fourni ☐ En attente |
| P8 | Affichage du numéro sur les postes de guichet (évolution desktop — non bloquant) | Équipe projet | ☐ Fourni ☐ En attente |

---

## 7. Validation

| Nom et prénom(s) | Fonction | Date | Visa |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

*Périmètre de cette fiche : génération des immatriculations définitives voitures et motos.
Hors périmètre de la première version : plaques provisoires `W`/`WW`, immatriculations
particulières (autorisation PM), Armée et Gendarmerie (exclues par le décret).*
