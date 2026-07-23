# Dossier de soutenance — Stratégie de génération des numéros d'immatriculation

> **Objet** Chaque décision est présentée au
> format : **la décision → pourquoi → les alternatives étudiées → pourquoi elles ont été écartées
> → la preuve**. Les éléments complexes ou flous sont démystifiés en fin de document, suivis des
> questions pièges probables et de leurs réponses courtes.
>
> Documents sources : `strategie-generation-numero-immatriculation.md` (technique complet),
> `simulation/rapport-simulation.md` (20 788 attributions, 53/53 contrôles),
> `releve-bascule-valeurs-initiales.md` (valeurs de bascule).

---

## 1. Le flow de A à Z (rappel en une image)

```
Validateur clique « Valider » (US-CG-008)
   │
   ▼
effet = typeDemande.effetNumero          ── CONSERVATION ─────────► rien (duplicata, gage…)
   │                                     ── RECOMPOSITION_REGION ─► réécrire les 2 chiffres région
   ▼ (TIRAGE et variantes)
contexte = construireContexte(dossier)      5 valeurs : dossierId, catégorie, support,
   │                                        code région/mission, mention
   ▼
stratégie = résoudre(catégorie)             ordinaire | diplomatique | chef de mission
   │
   ▼
attribuer(contexte)                         idempotence → verrou du compteur → avancer
   │                                        → INSERT registre (la base tranche) → numéro
   ▼
vehicule.immatriculation = numéro  →  COMMIT (tout ou rien)  →  réponse (web / bulk desktop)
```

---

## 2. Les décisions, une par une

### D1 — La génération est 100 % centrale (serveur)

**Décision** : seul le serveur central attribue des numéros ; le desktop et le back-office en
*reçoivent*, jamais n'en fabriquent.
**Pourquoi** : l'espace de numérotation est **national et séquentiel** (décret art. 9 : 0001→9999
par série) — il ne se partitionne pas. Un seul point d'attribution = un seul espace à protéger.
**Alternatives écartées** :
- *Le desktop génère hors-ligne* → impossible sans partitionner l'espace (contraire au décret) ;
  et vérifié dans le code du desktop : il ne génère que le numéro de **dossier**
  (`site+machine+jour+séquence` — partitionnable, lui), aucune logique de plaque n'y existe.
- *Des plages réservées par site/région* → les données réelles prouvent que le compteur est
  national (zéro numéro multi-région sur 353 000 plaques voitures ; plages régionales
  entrelacées sur tout l'espace). Réserver des plages créerait des trous artificiels massifs et
  violerait la continuité observée.
**Preuve** : analyse de `prod.txt` (test « numéros multi-régions » = 0 sur toutes les séries) ;
audit du repo `sigatt-desktop`.

### D2 — L'attribution a lieu à la VALIDATION, pas à la saisie

**Décision** : le numéro est tiré quand le validateur SIM clique « Valider ».
**Pourquoi** : c'est la spécification produit (US-CG-008 : « *Valider une demande — le système
attribue automatiquement un numéro d'immatriculation* ») ; et un numéro n'est **jamais recyclé**
(art. 3) — attribuer à la saisie gaspillerait des numéros pour chaque demande rejetée, corrigée ou
abandonnée.
**Alternatives écartées** :
- *À la saisie/soumission* → consommation irréversible pour des demandes qui n'aboutissent pas.
- *À la production de la carte* → trop tard : la carte à produire porte le numéro.
**Preuve** : texte de l'US-CG-008 ; règle « jamais recyclé » validée sur l'historique (les trous
des séries pleines n'ont jamais été réutilisés).

### D3 — L'aiguillage par nature est une DONNÉE (`type_demande.effet_numero`), pas du code

**Décision** : chaque nature de demande porte, en base, l'un des 6 effets
(`TIRAGE, RECOMPOSITION_REGION, CONSERVATION, CONDITIONNEL_STATUT_REGIME, TIRAGE_SERIE_PRIVEE,
TIRAGE_SERIE_STATUT`) ; le Java implémente les 6 comportements une seule fois.
**Pourquoi** : ajouter une 15ᵉ nature = une ligne dans l'écran d'administration existant, zéro
redéploiement ; le métier voit et gouverne la règle ; défaut sûr `CONSERVATION` (on ne tire jamais
un numéro par accident).
**Alternatives écartées** :
- *`if (code.equals("PMC"))` en dur* → couple le code Java à des codes de référentiel
  administrables ; un renommage dans l'écran casse l'aiguillage silencieusement ; chaque nouvelle
  nature rouvre le cœur validé.
- *Demander l'effet au validateur à chaque dossier* → source d'erreurs graves (un duplicata qui
  reçoit un nouveau numéro par mégarde) et charge inutile.
**Preuve** : patron déjà en production dans le projet (`StatutProprietaire.immatCode` porte la
série ; même principe). Implémenté le 22/07 (enum + colonne + seed des 14 natures des US, build
vert 127 tests).

### D4 — Le moteur consomme un CONTEXTE de 5 valeurs, pas le dossier

**Décision** : `attribuer(ContexteGenerationImmatriculation)` — un `record` immuable :
`dossierId, categorie, support, codeSuffixe, mention`.
**Pourquoi** : c'est la frontière entre « le monde du dossier » (riche, en chantier : workflow,
seeds, écrans) et « le monde du compteur » (5 valeurs, règles closes, prouvées par simulation).
Testable en une ligne, insensible aux chantiers alentour, immuable donc inaltérable entre la
validation et le tirage.
**Alternatives écartées** :
- *Passer l'entité `DossierImmatriculationEntity`* → le moteur dépendrait de la moitié du modèle
  (véhicule, propriétaire, quittance…), serait intestable sans tout monter, et chaque évolution du
  dossier menacerait le moteur.
- *Des paramètres en vrac (5 arguments)* → pas de validation groupée, signatures fragiles ;
  le record nomme le concept.
**Preuve** : même philosophie que `generateUniqueNumero(dto, batch)` du helper existant ;
c'est ce qui a permis au simulateur Python de valider la logique sans AUCUNE dépendance au reste.

### D5 — Une STRATÉGIE par famille réglementaire (pattern Strategy)

**Décision** : `StrategieSerieOrdinaire` (A/B/C/P/T + privés), `StrategieSerieDiplomatique`
(CD/CC/IN, bandes), `StrategieChefMission` (CMD) — résolues par
`List<StrategieSerieImmatriculation>` injectée par Spring, filtrée par `supporte(categorie)`.
**Pourquoi** : les familles ont des règles **structurellement différentes** — compteur national vs
par mission ; alphabet avec croisement vs bandes 0001-0999/1001-9999 vs pas de compteur du tout.
Ouvert/fermé : les provisoires W/WW (si IMPRO est un jour remplacé) ou les immatriculations
particulières = une nouvelle classe, le moteur validé n'est pas rouvert. Chaque stratégie se teste
isolément.
**Alternatives écartées** :
- *Un `switch` monolithique* → ~200 lignes, intestable par morceaux, rouvert à chaque évolution.
- *Héritage (une sous-classe de moteur par famille)* → fige la hiérarchie, complique l'injection
  et le partage du cycle verrou/registre qui, lui, est commun.
**Preuve** : le projet vient d'adopter le même choix côté desktop (refactor des formulaires en
Strategy) ; les 3 stratégies simulées couvrent les 44 scénarios sans un seul `if` par nature.

### D6 — Le compteur = UNE LIGNE EN BASE par série (`serie_immatriculation`)

**Décision** : table `cle_serie (PK), groupe_courant, valeur_courante` — ~175 lignes au démarrage
(12 ordinaires + 163 diplomatiques), initialisées par le fichier de calage.
**Pourquoi** : le compteur est un **état**, jamais un calcul. Il est lu/incrémenté/réécrit sous
verrou, dans la même transaction que le reste.
**Alternatives écartées** :
- *`SELECT MAX(numero) FROM vehicule` + 1* → fragile : soft-delete, plaques saisies (INTERNE),
  ré-immatriculations, imports — le MAX ment dès que les données bougent ; et il ne sait pas
  « quel groupe est en cours ».
- *Séquences PostgreSQL natives (`CREATE SEQUENCE`)* → une séquence ne sait faire que `+1` :
  ni passer de `G3` à `G4` à 9999, ni gérer les bandes diplomatiques, ni sauter un numéro déjà
  pris, ni s'arrêter sur « bande pleine ». Et les séquences PG sont **non transactionnelles**
  (pas de rollback) → trous systématiques à chaque échec. Notre table EST une séquence, mais avec
  les règles du décret dedans, et transactionnelle.
- *UUID / numéro aléatoire* → le format est réglementaire (décret art. 9-36) : séquentiel, par
  séries, porteur de sens (statut, région, régime). Non négociable.
- *Compteur en mémoire JVM (`AtomicLong`)* → perdu au redémarrage, faux dès la 2ᵉ instance.
**Preuve** : patron identique au `DossierImmatriculationSequenceEntity` **déjà en production**
pour le numéro de dossier.

### D7 — Verrou PESSIMISTE (`SELECT … FOR UPDATE`), pas optimiste

**Décision** : `findAndLockByCleSerie` avec `@Lock(PESSIMISTIC_WRITE)` ; l'avancement (y compris
le changement de groupe) se fait à l'intérieur du verrou.
**Pourquoi** : sur le compteur moto (~75 % du flux), **chaque tirage touche la même ligne** — le
conflit est le cas nominal, pas l'exception. Le pessimiste fait faire la queue quelques
millisecondes ; il est porté par PostgreSQL donc **valable multi-instances** sans infrastructure
supplémentaire. Une transaction ne prend jamais qu'un seul verrou de compteur → **aucun deadlock
possible** (pas de cycle). Une ligne par compteur → les files sont indépendantes (une moto ne
bloque pas un taxi, l'ambassade de France ne bloque pas le Ghana).
**Alternatives écartées** :
- *Verrou optimiste (`@Version`)* — le choix du socle partout ailleurs, à juste titre : il est
  fait pour les conflits **rares**. Ici, quasi tous les tirages entreraient en conflit → tempête
  de retries. On choisit le verrou selon la probabilité de conflit.
- *`synchronized` / verrou applicatif JVM* → faux dès la 2ᵉ instance du microservice (le CI
  docker existe déjà).
- *Redis/ZooKeeper (verrou distribué)* → une infrastructure de plus à opérer, et surtout le
  compteur sortirait de la transaction du dossier (incohérences en cas d'échec partiel).
- *File de messages (sérialiser par un consommateur unique)* → asynchrone : le validateur
  n'aurait pas le numéro dans sa réponse ; complexité massive pour un débit de quelques
  centaines/jour.
**Preuve** : simulation V3 — 2 000 tirages sur 8 fils d'exécution → suite **parfaitement
continue** 2089→4088, zéro doublon, zéro erreur. Patron déjà éprouvé en prod par
`testCreateConcurrentNominalCases`.

### D8 — Un REGISTRE dédié (`immatriculation_delivree`) avec contrainte UNIQUE

**Décision** : chaque numéro qui existe (généré, saisi, CMD migré) a une ligne dans un registre,
avec `cle_unicite` UNIQUE ; l'INSERT se fait dans la même transaction que l'incrément.
**Pourquoi** : c'est le **filet que le code ne peut pas contourner** — même un bug du compteur ne
peut pas produire un doublon : l'INSERT échoue, le numéro est sauté. Le registre garde les numéros
à vie (jamais recyclés, même après ré-immatriculation) et absorbe les plaques saisies.
**Alternatives écartées** :
- *Seulement une contrainte unique sur `vehicule.immatriculation`* → triple insuffisance :
  (1) la clé réglementaire est `(groupe, ordre)` **sans la région** — un déménagement change la
  chaîne (`2089 G3 03` → `2089 G3 09`) mais pas la clé : la contrainte sur la chaîne laisserait
  passer `(G3, 2089)` sous deux régions ; (2) une ré-immatriculation **remplace** le champ du
  véhicule — l'ancien numéro sortirait de l'espace protégé alors qu'il est consommé à vie ;
  (3) rien n'y accueillerait les plaques saisies sans véhicule créé ni les CMD migrées.
  (La contrainte sur `vehicule.immatriculation` est posée quand même — en ceinture, pas en
  mécanisme principal.)
- *Vérification applicative (`existsBy…`) sans contrainte* → course entre le check et l'insert ;
  le socle du projet lui-même documente que le filet DB reste le secours (`JpaExceptionHandler`).
**Preuve** : simulation S30 (numéro pré-inséré → sauté automatiquement) et S33 (50 collisions →
erreur propre, pas de boucle infinie).

### D9 — La clé d'unicité inclut le SUPPORT (groupe « tel qu'affiché »)

**Décision** : `cle_unicite` = `G3|2089` (voiture) vs `1DJ|0001` (moto) — le groupe est stocké
tel qu'imprimé, pas normalisé.
**Pourquoi** : `0739 D1 03` (voiture) et `0739 1D 03` (moto) sont **deux plaques physiques
distinctes** (le décret distingue les cycles par l'inversion chiffre/lettre).
**Alternative écartée** : clé normalisée (`D1|0739` pour les deux) → collision massive dès
l'import de test du legacy — **faille détectée puis prouvée par le simulateur** (l'import réel
échouait) ; c'est l'illustration de la valeur de la campagne de simulation.
**Preuve** : §5.2 de la stratégie, campagne V2 (import de 1 325 799 plaques sans erreur après
correction).

### D10 — Idempotence garantie par une contrainte `UNIQUE(dossier_id)`

**Décision** : le registre porte aussi `dossier_id UNIQUE` ; un rejeu (bulk desktop, double clic)
relit la plaque déjà attribuée au lieu d'en tirer une nouvelle.
**Pourquoi** : le desktop **rejoue réellement** les lots interrompus (vérifié dans son code :
`IN_PROGRESS → PENDING_PUSH` au redémarrage) ; et deux traitements simultanés du même dossier
passent tous deux le contrôle applicatif « déjà servi ? » avant d'allouer.
**Alternative écartée** : contrôle applicatif seul → la course reste ouverte (fenêtre entre le
check et l'insert) → deux numéros pour une moto. Seule la contrainte ferme la fenêtre.
**Preuve** : faille identifiée à la conception du simulateur ; scénario S31 (rejeu → même plaque,
compteur intact) validé en campagne, y compris sous 2 % de rejeux injectés dans le volet de masse.

### D11 — UNE transaction (`REQUIRED`), tout ou rien

**Décision** : compteur + registre + `vehicule.immatriculation` + statut du dossier commitent
ensemble, dans la transaction de l'appelant.
**Pourquoi** : si la validation échoue à mi-chemin, tout s'annule — ni plaque orpheline, ni
compteur incrémenté pour rien. Le verrou ne vit que quelques millisecondes. Dans le bulk, chaque
dossier est déjà isolé en `REQUIRES_NEW` par `SingleImportRunner` (lib-commons) : un dossier en
échec n'annule pas le lot.
**Alternative écartée** : `REQUIRES_NEW` sur l'attribution (verrou plus court) → un rollback du
dossier laisserait une plaque attribuée à un dossier inexistant ; le gain de latence est
négligeable au débit réel.
**Preuve** : scénario S32 (couvert par conception, testé en intégration Java) ; les « trous »
post-commit sont légaux (l'historique en est plein).

### D12 — Boucle de collision BORNÉE (50) avec saut définitif

**Décision** : un numéro dont l'INSERT échoue est sauté à jamais ; après 50 échecs consécutifs,
erreur `CONFLICT` explicite.
**Pourquoi** : le saut respecte « jamais réattribué » ; la borne interdit la boucle infinie et
transforme une situation anormale (50 collisions d'affilée = compteur très mal calé) en alerte
humaine plutôt qu'en comportement silencieux.
**Alternatives écartées** : échec à la première collision (trop fragile : une seule plaque saisie
dans le chemin bloquerait le guichet) ; boucle non bornée (risque de boucle infinie si l'espace
est saturé).
**Preuve** : même valeur (50) que le helper de numéro de dossier en prod ; scénarios S30/S33.

### D13 — Les trous ne sont JAMAIS comblés

**Décision** : le compteur ne redescend jamais ; les numéros non utilisés de l'historique restent
inutilisés.
**Pourquoi** : art. 3 (numéro permanent, jamais réattribué) ; et un « trou » n'est pas forcément
libre — côté motos, c'est souvent une plaque **fabriquée en stock**.
**Alternative écartée** : réutiliser les trous pour « économiser » l'espace → risque de doublon
physique avec une plaque en circulation ou en stock ; gain inutile (capacité restante : ~1,26 M
voitures avant croisement, ~29 M ensuite).
**Preuve** : analyse des stocks motos (corrélation numéro↔date ≈ 0) ; capacité chiffrée en
annexe C de la stratégie.

### D14 — Les motos démarrent sur une SÉRIE VIERGE (`DJ`)

**Décision** : `PRIVE|CYCLE = (DJ1, 0)` — le groupe suivant de l'alphabet réglementaire, jamais
mis en fabrication — plutôt que la reprise du front legacy (`DH1` @ 6615).
**Pourquoi** : l'ancien système moto écoulait des **plaques pré-fabriquées par lots** : un numéro
absent des données peut correspondre à une plaque qui dort chez un concessionnaire et sera posée
après la bascule.
**Alternative écartée** : continuité stricte à `DH1+marge` → la simulation l'a **chiffrée** :
sur 5 000 plaques de stock posées après bascule, **384 (7,7 %) voient leur numéro redonné à une
autre moto** (deux motos porteraient physiquement le même numéro) ; la série vierge : **0
conflit**. C'est une décision métier (D4 de la fiche) mais elle arrive avec son chiffre.
**Preuve** : campagne V4, deux variantes comparées toutes choses égales par ailleurs ; hypothèse
des stocks prouvée par les dates (r ≈ +0,02 à +0,08 entre numéro et date de demande).

### D15 — Bascule par RELEVÉ à l'arrêt, sans import massif (décision projet)

**Décision** : arrêt de l'ancien système → relevé automatisé des derniers numéros (fichier de
calage : 12 compteurs ordinaires + 163 diplomatiques + 24 plaques CMD) → PV contradictoire →
configuration → activation. Le registre démarre vide et se reconstitue au fil de l'eau.
**Pourquoi** : bascule simple et rapide ; le relevé à l'arrêt est **exact** (plus de problème de
fraîcheur d'extraction) ; l'import de l'historique reste possible plus tard, en option différée,
sans impact sur la génération.
**Alternative écartée** *(par le projet)* : import massif de l'historique dans le registre —
plus lourd à la bascule ; il apportait toutefois deux protections qu'il faut compenser et
assumer : (1) le filet du registre contre une **erreur de relevé** → compensé par le relevé par
script (jamais manuel), le double relevé, le PV et la marge ; (2) le contrôle d'existence des
anciennes plaques saisies → dégradé en contrôle de **format** (l'import différé le rétablira).
**Preuve** : `releve-bascule-valeurs-initiales.md` (généré par script, contrôle « une valeur du
jour J ne peut pas être inférieure à celle d'avril 2024 ») ; la logique d'allocation validée en
simulation est identique dans les deux modes.

### D16 — Les résolutions du contexte : des choix assumés

- **Série ← `statutProprietaire.immatCode`** : le statut porte son code de catégorie (validé par
  les données : COLLECT→B à 98 %, POLICE→P à 91 %…). Alternative écartée : déduire la série du
  type d'usager ailleurs dans le dossier → redondant et incohérent avec le décret (art. 2 : le
  statut détermine la série). ⚠️ Le seed actuel (A→N naïf) est à corriger — connu, tracé.
- **Support ← `typeVehicule`** (codes motocycle/tricycle/quadricycle → CYCLE) : seule information
  fiable du dossier sur la nature du véhicule ; `nombre_plaque` (2/1) portée par le même
  référentiel (art. 4).
- **Région ← résidence du propriétaire** (localité → province → région), repli site
  d'enrôlement : c'est la lettre du décret (art. 2 : « lieu de résidence ») ; le site n'est qu'un
  lieu de dépôt. Alternative écartée : région du site → un usager de Kaya déposant à Ouaga
  recevrait `03` au lieu de `05`, faux au sens du décret. (Décision métier à confirmer — D4 fiche.)
- **Mention ← `regimeDouanier.mentionSerie`** : le régime douanier détermine la série
  normale/spéciale (art. 7) ; même patron data-driven que D3. Défaut sûr : pas de mention.

### D17 — Paramétrage : table `parametre` + flag d'activation + marge

**Décision** : `immatriculation.generation.active` (défaut `false`), `immatriculation.amorcage.marge`
(défaut 10), clés dans la table `parametre` (écran d'admin existant).
**Pourquoi** : activer/désactiver sans redéploiement (bascule réversible) ; la marge est une
ceinture contre l'imprévu du relevé — faible car l'ancien système est arrêté, jamais nulle.
**Alternative écartée** : config yaml (config-repos) seule → modifiable uniquement par
redéploiement/redémarrage et invisible du métier ; la table `parametre` est administrable et déjà
outillée.

### D18 — Conformité aux conventions du projet (le « rien d'exotique »)

Enums **nus** dans `sigatt.commons.enums` (pas de libellés backend) ; DTO dans lib-commons ;
erreurs `SigattErrorResponse` + clés i18n `<entité>.<cas>` ; Javadoc française ; checkstyle
bloquant respecté ; finders `…AndDeletedFalse` ; pas de méthode publique `final` dans les services
proxifiés. **Pourquoi c'est un argument** : le moteur n'introduit AUCUN paradigme nouveau dans le
projet — verrou pessimiste (déjà en prod), auto-config socle, MapStruct, patrons de test existants.
Le risque d'intégration est minimal.

---

## 3. Les éléments complexes, démystifiés

**Le croisement à deux lettres (`Z9 → DD1 → DE1 …`)** — flou dans le décret (art. 27 : « croisement
deux à deux », un seul exemple). Levé **empiriquement** : le parc moto l'utilise déjà, et les
données montrent l'ordre réel — première lettre fixe, seconde parcourant l'alphabet
(`DD→DE→DF→DG→DH`). La mécanique est isolée dans un utilitaire unique (`AlphabetSeriePrivee`) :
si la DGTTM formalise un autre ordre, une seule classe change.

**Les bandes diplomatiques (0001-0999 / 1001-9999)** — ce ne sont pas deux séries mais **deux
plages du même espace par mission** : service en bas, personnel en haut, le numéro seul dit qui
est quoi (`0045 CD 01` = service, `1023 CD 01` = diplomate). D'où **deux compteurs par mission**,
et une erreur dédiée « bande pleine » (pas de débordement d'une bande sur l'autre — escalade
métier).

**CMD sans compteur** — `02 CMD` n'a pas de numéro d'ordre : la plaque EST le code de la mission.
La règle « un seul véhicule par mission » est portée par la clé `CMD|02` du registre — c'est
pourquoi les 24 plaques CMD existantes sont le **seul** élément à insérer au registre à la bascule.

**Pourquoi aucun deadlock n'est possible** — un deadlock exige un cycle (A tient X et veut Y,
B tient Y et veut X). Ici une transaction ne verrouille jamais qu'**un seul** compteur, et le
registre s'insère (pas de verrou croisé). Pas de cycle possible, par construction.

**Numéro de dossier ≠ numéro d'immatriculation** — deux espaces totalement disjoints : le premier
(`OUAG 0000 220726 0001`) est un identifiant de traitement, partitionné par site+machine+jour,
générable hors-ligne ; le second est l'identité légale du véhicule, nationale et séquentielle,
générable uniquement au centre. Le moteur des plaques **copie le patron** du helper de dossier
(verrou + retries) mais ne partage pas son code : les règles divergent (groupes, bandes, registre).

**D'où viennent les chiffres avancés** — tout est traçable : les règles viennent des décrets
2017-0114/0101 (articles cités) ; les comportements réels de 1,4 M de numéros analysés ; les
valeurs de bascule du calcul automatisé (`releve-bascule-valeurs-initiales.md`) ; les garanties
anti-doublon de la campagne de simulation (20 788 attributions, 53/53, rejouable graine fixée).

---

## 4. Questions pièges probables — réponses courtes

| Question | Réponse en une phrase |
|---|---|
| « Pourquoi pas des UUID ? » | Le format est réglementaire (décret art. 9-36) : séquentiel, par séries, porteur de sens. |
| « Pourquoi pas une séquence PostgreSQL ? » | Elle ne sait faire que +1 — ni changement de groupe, ni bandes, ni saut de collision — et elle ne rollback pas. |
| « Le verrou va être un goulot. » | Une ligne par compteur, quelques ms par tirage, débit métier ≈ centaines/jour ; simulé : 2 000 tirages / 8 threads sans une erreur. |
| « Et avec plusieurs instances ? » | Le verrou est dans PostgreSQL, pas en JVM — la sérialisation est identique à N instances. |
| « Deadlock ? » | Impossible par construction : un seul verrou de compteur par transaction, pas de cycle. |
| « Deux nœuds traitent le même dossier ? » | `UNIQUE(dossier_id)` au registre : le second relit la plaque du premier. |
| « Compteur mal calé à la bascule ? » | Relevé par script + double relevé + PV + marge ; et toute plaque saisie entre au registre où le générateur la sautera. |
| « Pourquoi une série vierge pour les motos ? » | Stocks pré-fabriqués prouvés par les données ; simulé : reprise du front = 7,7 % de doublons physiques, série vierge = 0. |
| « Comment vous testez sans le workflow ? » | Le moteur consomme un contexte de 5 valeurs : 40 des 44 scénarios se testent en JUnit avec le patron existant (mocks Feign + dataset), l'oracle étant la simulation. |
| « Qu'est-ce qui est nouveau/risqué ? » | Rien d'exotique : verrou pessimiste déjà en prod (numéro de dossier), conventions socle partout ; la seule nouveauté est le registre — et c'est une table + une contrainte. |
| « Et les plaques provisoires W/WW ? » | Hors périmètre : gérées par IMPRO (confirmé par les US) ; le pattern Strategy les accueillera si IMPRO est remplacé. |
| « Ça tient la charge de la reprise nationale (art. 50) ? » | Volet V2 : 11 746 attributions en 0,3 s en logique pure ; le facteur limitant sera la base, pas l'algorithme. |

---

## 5. Les 5 preuves à projeter en réunion

1. **La réglementation** : décrets 2017-0114 + arrêté 2017-0101, articles cités à chaque règle.
2. **Les données** : 1,4 M de numéros réels analysés — compteur national prouvé, mapping régions
   validé à 99 %, statut→série validé, stocks motos prouvés.
3. **Le produit** : US-CG-008 fixe le déclencheur mot pour mot ; les 14 natures ont chacune leur
   effet, seedées.
4. **La simulation** : 20 788 attributions, **53/53 contrôles OK**, prototype fidèle au
   pseudo-code, rejouable (`simulation/`).
5. **Les précédents du projet** : verrou pessimiste + retries déjà en production ; conventions
   socle respectées à la lettre — le moteur est une extension du système, pas un corps étranger.
