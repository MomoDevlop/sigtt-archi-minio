# sigtt-archi-minio
# SIGATT — Stockage documentaire MinIO partagé

> **Document d'architecture et de vulgarisation** — proposition de mise en place du stockage des documents
> scannés (demandes d'immatriculation, de permis de conduire, …) sur une instance **MinIO partagée**, avec la
> logique centralisée dans une **librairie commune** (`sigatt-lib-storage-commons`) réutilisable par tous les
> microservices métier.
>
> Ce document est volontairement **pédagogique** : chaque décision technique est expliquée avec son *pourquoi*
> et les alternatives écartées. Il peut être lu par une personne qui découvre MinIO.

---

## Table des matières

1. [Objectif](#1-objectif)
2. [MinIO pour les débutants](#2-minio-pour-les-débutants)
3. [La solution proposée — vue d'ensemble](#3-la-solution-proposée--vue-densemble)
4. [Chaque décision expliquée (et pourquoi)](#4-chaque-décision-expliquée-et-pourquoi)
5. [Les configurations expliquées ligne à ligne](#5-les-configurations-expliquées-ligne-à-ligne)
6. [Intégration dans SIGATT pas à pas](#6-intégration-dans-sigatt-pas-à-pas)
7. [Bonnes pratiques et évolutions futures](#7-bonnes-pratiques-et-évolutions-futures)
8. [Sources](#8-sources)

---

## 1. Objectif

### 1.1 Le besoin métier

Les microservices **métier** de la plateforme SIGATT (aujourd'hui `immatriculation-service`, demain
`permis-conduire-service` et d'autres) traitent des **demandes** d'usagers. Chaque demande s'accompagne de
**documents scannés** : carte d'identité, facture d'achat, ancienne carte grise, certificat de dédouanement, etc.

Il faut donc pouvoir, pour chaque demande :

| Action | Description |
|---|---|
| **Enregistrer** | téléverser **un ou plusieurs** documents en une seule opération |
| **Lister** | récupérer la liste de **tous les documents d'une demande** donnée |
| **Consulter** | récupérer les informations (métadonnées) d'**un document** précis |
| **Télécharger** | récupérer le **fichier binaire** d'un document |
| **Supprimer** | retirer un document d'une demande (suppression logique) |

### 1.2 Les exigences techniques

- **Formats acceptés** : PDF, JPEG, PNG — **liste configurable** (on doit pouvoir ajouter un format sans recompiler).
- **Taille maximale** : **10 Mo par fichier** — **configurable** également.
- **Une instance MinIO partagée** pour tous les microservices (mutualisation de l'infrastructure).
- **Toute la logique MinIO centralisée dans les librairies communes** : un microservice ne doit PAS réécrire du
  code MinIO ; il ajoute une dépendance et consomme des services prêts à l'emploi — exactement comme il consomme
  aujourd'hui le socle CRUD générique de `sigatt-lib-commons` ou la sécurité de `sigatt-lib-security-commons`.

### 1.3 Ce qui existe déjà (état des lieux vérifié)

- **Aucun code de gestion de fichiers** n'existe dans le backend (pas de `MultipartFile`, pas de MinIO/S3, pas de
  configuration multipart). Le terrain est vierge : on conçoit proprement dès le départ.
- **Pas encore d'entité `Demande`** dans les services métier — l'API documentaire sera donc conçue autour d'un
  identifiant de demande (`demandeId`, UUID) et se branchera naturellement sur la future entité.
- Le **référentiel** possède déjà le *catalogue* des pièces : `TypePieceJointe` (quels types de pièces existent)
  et `TypePieceJointeTypeDemande` (quelles pièces sont requises/obligatoires pour quel type de demande). Le
  module documentaire s'y raccordera en référençant `typePieceJointeId`.
- Le **frontend** possède déjà le composant d'upload (`app-file-input`, PrimeNG, 10 Mo par défaut, mode multiple)
  et le champ `file` du formulaire dynamique — seul un petit service Angular « multipart » manquera le moment venu.

---

## 2. MinIO pour les débutants

### 2.1 C'est quoi, le « stockage objet » ?

Il existe trois grandes façons de stocker des fichiers :

| Approche | Comment ça marche | Limites |
|---|---|---|
| **Système de fichiers** (disque local du serveur) | On écrit le fichier dans un dossier (`/var/sigatt/docs/…`) | Lié à UNE machine : si le service tourne sur 2 instances, elles ne voient pas les mêmes fichiers ; sauvegarde/réplication artisanales ; pas de contrôle d'accès fin |
| **Base de données** (colonne `bytea`/BLOB) | Le fichier devient une ligne en base | La base grossit énormément, les sauvegardes deviennent lentes et lourdes, les performances de la base se dégradent — les SGBD ne sont pas faits pour servir des binaires |
| **Stockage objet** (MinIO, Amazon S3, …) | Le fichier devient un **objet** identifié par une **clé**, dans un **bucket**, accessible via une **API HTTP** | ✅ C'est l'approche standard moderne pour les fichiers |

Le **stockage objet** est un service dédié, séparé de l'application et de la base de données, dont le seul métier
est de stocker et servir des binaires de façon **fiable**, **scalable** et **sécurisée**.

### 2.2 C'est quoi, MinIO ?

**MinIO** est un serveur de stockage objet **open-source**, très léger (un seul binaire), qu'on installe **sur ses
propres serveurs** (*on-premise*). Sa grande force : il implémente **l'API Amazon S3**, le standard de fait du
stockage objet. Tout outil ou SDK qui sait parler à S3 sait parler à MinIO.

**Pourquoi MinIO pour SIGATT ?**
- **Souveraineté** : les documents administratifs (cartes grises, permis) restent sur l'infrastructure de l'État —
  pas de cloud étranger.
- **Open-source et gratuit**, déploiement simple (binaire ou conteneur Docker).
- **Compatible S3** : compétences et outillage standard, migration possible vers un autre stockage S3 sans tout
  réécrire.
- **Fonctions avancées** disponibles quand on en aura besoin : versioning, verrouillage d'objets (immuabilité),
  chiffrement, réplication.

### 2.3 Le vocabulaire à connaître

| Terme | Définition | Analogie |
|---|---|---|
| **Bucket** | Conteneur de plus haut niveau, qui regroupe des objets. On lui attache des règles (droits, versioning, quotas) | Un « disque » ou une « racine de dossier » |
| **Objet (object)** | Un fichier stocké + ses métadonnées | Un fichier |
| **Object key** | Le « chemin/nom » unique de l'objet dans son bucket, ex. `3fa8…/carte-grise/9b1c….pdf`. Les `/` sont une convention de préfixe, pas de vrais dossiers | Le chemin du fichier |
| **Access key / Secret key** | Le couple identifiant/mot de passe d'un compte technique MinIO | Login/mot de passe applicatif |
| **Policy** | Règle d'autorisation attachée à un compte : « peut lire/écrire dans TEL bucket seulement » | Droits d'accès |
| **ETag** | Empreinte technique calculée par MinIO à l'upload (souvent un MD5, mais pas garanti) | Numéro de version technique |
| **Presigned URL** | URL temporaire signée qui donne accès direct à UN objet pendant N minutes, sans authentification supplémentaire | Lien de partage à durée limitée |
| **API S3** | L'API HTTP standard (héritée d'Amazon S3) que MinIO implémente | Le « langage » commun |

### 2.4 Prérequis pour utiliser MinIO dans le projet

1. **Une instance MinIO** qui tourne (en dev : un conteneur Docker ; en prod : un déploiement dédié, idéalement
   distribué sur plusieurs disques).
2. **Un endpoint réseau** joignable par les microservices (ex. `http://minio.sigatt.gov.bf:9000`) — en production,
   impérativement en **TLS (https)**.
3. **Des credentials** (access key / secret key) par microservice, fournis par variables d'environnement — jamais
   en clair dans un dépôt Git.
4. **Des buckets** créés et configurés (droits, versioning) — provisionnés par l'exploitation en production,
   auto-créés par l'application uniquement en développement.

---

## 3. La solution proposée — vue d'ensemble

### 3.1 Architecture

```
                       ┌───────────────────────────────────────────────────────────┐
                       │                    SIGATT backend                          │
                       │                                                            │
 [Angular]  ──JWT──▶ [API Gateway] ──▶ [immatriculation-service] ──API S3──▶ ┌─────────────┐
  front                 :8081           │  ▲                                  │    MinIO     │
                                        │  │ dépendance Maven                │  (partagé)   │
                                        │  │                                  │              │
                                        │  └── sigatt-lib-storage-commons     │ bucket :     │
                                        │      (toute la logique MinIO)       │  sigatt-     │
                                        │                                     │  immatricu-  │
                                        ▼                                     │  lation      │
                                 [PostgreSQL du service]                      │              │
                                  table `document`                            │ bucket :     │
                                  (métadonnées = source                       │  sigatt-     │
                                   de vérité)                                 │  permis-…    │
                                                                              └─────────────┘
        [permis-conduire-service] (futur) ── même dépendance, son propre bucket ──▲
```

Points clés :
- Le **navigateur ne parle JAMAIS à MinIO**. Tout passe par la gateway puis le microservice (sécurité JWT/Keycloak
  conservée de bout en bout).
- Chaque microservice a **son bucket** et **ses credentials**, mais tous utilisent **la même librairie**.
- Le **binaire** vit dans MinIO ; les **métadonnées** (quel fichier, pour quelle demande, quel type de pièce,
  quelle taille, quel checksum…) vivent dans la **base PostgreSQL du microservice**.

### 3.2 Cycle de vie d'un document

1. **Upload** : le front envoie le(s) fichier(s) en `multipart/form-data` au microservice avec le `demandeId`.
2. **Validation** : la librairie vérifie pour CHAQUE fichier : extension autorisée, **type réel** du contenu
   (lecture des premiers octets — « magic bytes »), taille ≤ maximum configuré, fichier non vide.
3. **Stockage** : la librairie génère une clé unique (`{demandeId}/{typePiece}/{uuid}.pdf`), calcule le
   **SHA-256** du contenu, envoie l'objet dans le bucket du service.
4. **Métadonnées** : une ligne est insérée dans la table `document` locale (nom d'origine, type MIME, taille,
   bucket, clé, checksum, auteur, date — l'audit est automatique via le socle existant).
5. **Consultation** : liste des documents d'une demande = simple requête SQL sur la table `document`.
6. **Téléchargement** : le microservice vérifie les droits, lit l'objet dans MinIO et **le streame** au client
   avec les bons en-têtes (`Content-Type`, `Content-Disposition` avec le nom d'origine).
7. **Suppression** : **logique** (le champ `deleted` du socle passe à `true`) ; le binaire est conservé dans
   MinIO (traçabilité administrative).

---

## 4. Chaque décision expliquée (et pourquoi)

### 4.1 Une librairie dédiée `sigatt-lib-storage-commons`

**Décision** : créer un **nouveau module Maven** `sigatt-lib-storage-commons`, sur le modèle de
`sigatt-lib-security-commons`.

**Pourquoi pas du code MinIO dans chaque microservice ?** Duplication massive : chaque service réécrirait la
connexion, la validation, la gestion d'erreurs — avec des divergences inévitables et des bugs corrigés à un
endroit mais pas à l'autre. C'est exactement ce que le projet a évité pour le CRUD avec le socle générique.

**Pourquoi pas directement dans `sigatt-lib-commons` ?** Deux raisons :
1. **Opt-in** : tous les services n'ont pas besoin de stockage (la gateway, le referentiel, uaa n'en ont pas
   l'usage). Une lib dédiée = seuls les services métier l'ajoutent, les autres n'embarquent ni MinIO ni Tika.
2. **Cycle de vie** : la lib de stockage évoluera à son rythme (versioning indépendant, comme
   `lib-security-commons` en 0.0.5) sans forcer la reconstruction de tout ce qui dépend de `lib-commons`.

La lib suit le **pattern d'auto-configuration déjà en place** dans le projet (voir §6.1) : le service ajoute la
dépendance, et les beans s'activent automatiquement — zéro configuration Java côté service.

### 4.2 SDK `io.minio:minio` (et non AWS SDK v2)

**Décision** : utiliser le **SDK Java officiel de MinIO** (`io.minio:minio`, branche 9.x), **masqué derrière une
abstraction maison** (`DocumentStorageService`).

**Les deux candidats** :
- `io.minio:minio` — SDK dédié MinIO : API simple et directe (`putObject`, `getObject`, `statObject`,
  `listObjects`, `removeObject`), documentation alignée sur MinIO.
- AWS SDK v2 (`software.amazon.awssdk:s3`) — le SDK générique S3 d'Amazon, pointable sur MinIO
  (`endpointOverride` + `forcePathStyle`) : très mature, portable vers n'importe quel stockage S3, mais plus
  verbeux et conceptuellement « AWS ».

**Pourquoi io.minio ?** Notre besoin est **MinIO-only** (infrastructure on-premise décidée) ; l'API dédiée est
plus simple à lire et à maintenir pour l'équipe. Et surtout : **aucun microservice ne verra le SDK**. Toute la
plateforme ne connaîtra que l'interface maison `DocumentStorageService`. Si un jour il fallait migrer vers AWS
SDK v2 ou un autre stockage, **une seule classe** serait réécrite, dans la lib — pas les microservices. C'est
l'abstraction qui protège, pas le choix du SDK.

> ⚠️ Vigilance : la branche 9.x du SDK est une refonte (beaucoup d'exemples web basés sur 8.5.x ne compilent
> plus tel quel) et repose sur OkHttp. La compatibilité avec notre stack très récente (Spring Boot 4, Java 25)
> sera validée dès le premier build ; en cas de friction, l'abstraction permet de basculer sur AWS SDK v2 sans
> impact sur les services.

### 4.3 Un bucket par microservice (et non un bucket partagé)

**Décision** : sur l'instance MinIO **partagée**, chaque microservice a **son propre bucket**
(`sigatt-immatriculation`, `sigatt-permis-conduire`, …) et **ses propres credentials** limités à ce bucket.

**Pourquoi pas un bucket unique `sigatt-documents` avec des préfixes `{service}/…` ?**
- **Isolation** : avec un bucket unique, une seule paire de credentials donne accès aux documents de TOUS les
  services (ou il faut des policies par préfixe, plus complexes et plus faciles à rater). Avec un bucket par
  service + une policy « moindre privilège », une fuite de credentials du service immatriculation ne compromet
  jamais les documents du permis de conduire.
- **Gouvernance par service** : quotas, versioning, règles de rétention, métriques — tout se règle **au niveau du
  bucket**. Chaque domaine métier peut avoir ses propres règles.
- **Révocation** : on peut couper l'accès d'un service sans toucher les autres.

C'est la recommandation officielle MinIO pour le multi-applications : *« concevez les buckets autour des
applications — une application = un bucket »*.

### 4.4 Convention de nommage des clés d'objets

**Décision** : `"{demandeId}/{typePieceJointeId}/{uuid}.{extension}"` (le segment type est omis si non fourni).

**Pourquoi ?**
- **`{demandeId}/` en préfixe** : MinIO liste efficacement par préfixe → retrouver tous les objets d'une demande
  est une opération native (utile pour l'audit et le futur job de réconciliation), même si au quotidien c'est la
  base locale qui répond.
- **`{uuid}` comme nom de fichier** : garantit l'unicité (deux usagers peuvent téléverser `scan.pdf`), et évite
  d'injecter le nom fourni par l'utilisateur dans un chemin (risque de caractères dangereux / *path traversal*).
  Le **nom d'origine** est conservé en métadonnée et resservi au téléchargement.
- **Noms de buckets** : conformes S3 (minuscules, tirets, 3–63 caractères) — d'où `sigatt-immatriculation` et
  non `sigatt_immatriculation`.

### 4.5 Téléchargement en proxy streaming (et non presigned URLs)

**Décision** : le front télécharge via `GET /api/documents/{id}/download` **sur le microservice**, qui vérifie le
JWT et les droits, lit l'objet dans MinIO et le **streame** dans la réponse HTTP.

**L'alternative presigned URL** : le service génère une URL signée temporaire, et le navigateur télécharge
**directement depuis MinIO**. Avantage : le backend n'est pas dans le chemin du transfert. **Mais** :
1. L'URL signée **contourne la gateway et le JWT** : quiconque possède l'URL (log, historique, partage) accède au
   document jusqu'à expiration — c'est un « porteur de droit » hors de notre système d'auth.
2. Il faudrait **exposer MinIO au navigateur** (DNS public, TLS, reverse-proxy) — surface d'attaque supplémentaire
   pour une infrastructure qu'on préfère interne.

**Pourquoi le proxy est le bon choix ICI** : nos fichiers font **≤ 10 Mo** et le volume est administratif
(modéré). Le coût CPU/réseau du streaming par le backend est négligeable à cette échelle, et on conserve **un seul
modèle de sécurité** (Keycloak/JWT/gateway) et une traçabilité complète. Les presigned URLs restent une évolution
documentée (§7) si la volumétrie explose un jour.

### 4.6 Les métadonnées en base locale (source de vérité)

**Décision** : chaque microservice a une table **`document`** dans SA base PostgreSQL ; MinIO ne stocke que le
binaire (+ quelques métadonnées « de secours »).

**Pourquoi ne pas tout mettre dans les user-metadata MinIO ?** Les métadonnées objet MinIO ne sont **pas
requêtables** (impossible de faire « donne-moi tous les documents de la demande X triés par date » sans lister et
inspecter les objets un à un), **pas transactionnelles** avec les écritures métier, et **immuables** après upload.
Une table SQL offre les requêtes, les index, les jointures futures avec `Demande`, l'audit du socle
(`AbstractAuditEntity` : auteur, dates, soft-delete, verrou optimiste) — gratuitement.

**Le modèle de la table `document`** (chaque colonne justifiée) :

| Colonne | Type | Pourquoi |
|---|---|---|
| `id` | UUID (PK) | Convention du projet (UUID partout) |
| `demande_id` | UUID, indexé | La clé de regroupement métier — toutes les recherches partent de là |
| `type_piece_jointe_id` | UUID, nullable | Raccordement au **catalogue referentiel** (`TypePieceJointe`) : permet de savoir « ce document est la carte d'identité » et de contrôler plus tard les pièces obligatoires |
| `nom_original` | varchar | Le nom du fichier tel que fourni par l'usager — resservi au téléchargement (`Content-Disposition`) |
| `content_type` | varchar | Le type MIME **réel validé** (pas celui déclaré par le navigateur) — resservi au téléchargement (`Content-Type`) |
| `taille` | bigint | Affichage et contrôles sans interroger MinIO |
| `bucket` | varchar | Auto-descriptif : la ligne dit où est le binaire, même si la config change |
| `object_key` | varchar, unique | Le lien vers l'objet MinIO — la colonne pivot |
| `checksum_sha256` | varchar | **Intégrité de bout en bout** : recalculable à tout moment pour prouver que le document n'a pas été altéré ; permet aussi la déduplication |
| `etag` | varchar | Référence technique renvoyée par MinIO (vérifications de cohérence) — ⚠️ pas fiable comme empreinte de contenu (en multipart ce n'est pas le MD5), d'où le SHA-256 applicatif ci-dessus |
| *(hérité du socle)* | | `created_by`/`created_date` (= **auteur** et date du dépôt), `deleted` (suppression logique), `is_active`, `row_version` |

**Pourquoi le SHA-256 en plus de l'ETag ?** L'ETag est calculé par MinIO et son format n'est pas garanti
(notamment en upload multipart). Le SHA-256 calculé **par notre application** au moment de l'upload est une
empreinte cryptographique fiable : c'est notre preuve d'intégrité pour des documents administratifs.

### 4.7 Validation « défense en profondeur » (Tika et non l'extension seule)

**Décision** : trois contrôles cumulés par fichier, dans la lib :
1. **Extension** ∈ liste configurable (`pdf`, `jpg`, `jpeg`, `png`) ;
2. **Type MIME réel** détecté par **Apache Tika** (lecture des premiers octets du fichier — les « magic bytes »)
   ∈ liste configurable, ET cohérent avec l'extension ;
3. **Taille** ≤ maximum configurable (10 Mo par défaut), fichier non vide.

**Pourquoi l'extension seule ne suffit pas ?** L'attaque classique : renommer `virus.exe` en `photo.png`.
L'extension et le `Content-Type` envoyés par le navigateur sont **déclaratifs** — l'utilisateur (ou un attaquant)
les contrôle entièrement. Les **magic bytes**, eux, sont les premiers octets du contenu réel : un PDF commence
par `%PDF`, un PNG par `\x89PNG`… Tika lit ces signatures et renvoie le vrai type. Rejeter toute divergence
(extension `.png` mais contenu exécutable) ferme cette porte.

**Et le double contrôle de taille ?** La limite `spring.servlet.multipart.max-file-size` (conteneur) rejette
très tôt les requêtes énormes (protection anti-DoS), et la validation applicative applique la limite **métier
configurable** (avec un message d'erreur i18n propre, format `document.taille.max`).

### 4.8 Upload MinIO d'abord, base ensuite

**Décision** : à l'upload, on écrit **d'abord l'objet dans MinIO**, **puis** la ligne `document` en base (dans la
transaction du service). En cas d'échec de la base, on tente de supprimer l'objet (nettoyage best effort).

**Pourquoi cet ordre ?** Il n'existe pas de transaction commune entre MinIO et PostgreSQL. Il faut donc choisir
le moindre mal :
- *Base d'abord, MinIO ensuite* → si l'upload échoue, on a une métadonnée qui pointe vers **rien** : lien mort
  visible par l'utilisateur. Grave.
- *MinIO d'abord, base ensuite* → si la base échoue, on a un objet **orphelin** invisible de l'application :
  un peu d'espace disque perdu, aucune incidence fonctionnelle. Bénin — et rattrapable par un job de
  réconciliation (§7).

### 4.9 Suppression logique, binaire conservé

**Décision** : `DELETE /api/documents/{id}` fait un **soft-delete** de la métadonnée (champ `deleted` du socle) ;
l'objet MinIO n'est **pas** effacé.

**Pourquoi ?** (1) Cohérence avec tout le projet : aucune entité SIGATT n'est supprimée physiquement.
(2) Contexte administratif : un document rattaché à une demande officielle peut devoir être produit
ultérieurement (contentieux, audit). La purge physique éventuelle (RGPD/rétention) sera une règle
d'exploitation explicite (règles de cycle de vie par bucket), pas un effet de bord d'un clic.

### 4.10 Auto-création du bucket : en développement seulement

**Décision** : la lib peut créer le bucket au démarrage (`bucketExists` → `makeBucket`), mais **uniquement si
`sigatt.storage.auto-create-bucket=true`** — activé en dev, désactivé en production.

**Pourquoi ?** En production, les buckets doivent être provisionnés par l'exploitation (avec versioning,
policies, credentials dédiés) de façon contrôlée et auditée — pas créés implicitement par une application avec
des droits larges. En développement, l'auto-création évite une étape manuelle à chaque nouveau poste.

---

## 5. Les configurations expliquées ligne à ligne

### 5.1 Les propriétés `sigatt.storage.*` (nouvelles, portées par la lib)

Définies dans `sigatt-config-repos` (le dépôt de configuration servi par le config-server), liées côté code par
une classe `@ConfigurationProperties(prefix = "sigatt.storage")` — même mécanisme que `sigatt.cors.*` (gateway)
et `keycloak.*` (sécurité).

**Partie commune — `microservice-common-dev.yaml`** (tous les microservices héritent) :

```yaml
sigatt:
  storage:
    # Adresse de l'API S3 de MinIO. Variable d'env en prod, localhost en dev.
    endpoint: ${SIGATT_MINIO_URL:http://localhost:9000}
    # Credentials du service : TOUJOURS par variable d'environnement, jamais en clair dans Git
    # (même convention que SIGATT_DB_USERNAME / SIGATT_EUREKA_PASSWORD).
    access-key: ${SIGATT_MINIO_ACCESS_KEY}
    secret-key: ${SIGATT_MINIO_SECRET_KEY}
    # Taille maximale PAR FICHIER (exigence : 10 Mo, configurable). Format DataSize Spring (10MB, 15MB…).
    max-file-size: 10MB
    # Extensions autorisées (exigence : configurable). Ajout d'un format = modifier cette liste, redémarrer.
    allowed-extensions: [pdf, jpg, jpeg, png]
    # Types MIME réels autorisés (contrôle Tika). Doit rester cohérent avec la liste d'extensions.
    allowed-content-types: [application/pdf, image/jpeg, image/png]

spring:
  servlet:
    multipart:
      # Garde-fou du conteneur web : rejette la requête AVANT de la traiter si un fichier dépasse.
      max-file-size: 10MB
      # Taille max de la REQUÊTE ENTIÈRE : doit couvrir un upload multi-fichiers (ex. 5 fichiers de 10 Mo + marge).
      max-request-size: 60MB
```

**Partie par service — ex. `immatriculation-service.yaml`** :

```yaml
sigatt:
  storage:
    # LE bucket de ce microservice (décision « un bucket par service »).
    bucket: sigatt-immatriculation
    # Auto-création du bucket au démarrage : true en DEV uniquement (voir §4.10). En prod : false/absent.
    auto-create-bucket: true
```

**Pourquoi cette répartition commun / par service ?** L'endpoint, les credentials (variables), les règles de
validation sont **les mêmes pour tous** → fichier commun (une seule modification pour changer la taille max
partout). Le **bucket** est **l'identité de chaque service** → fichier du service. Un futur
`permis-conduire-service.yaml` n'aura à déclarer que `bucket: sigatt-permis-conduire`.

> À noter : chaque propriété a une **valeur par défaut raisonnable dans le code** (`10MB`, `[pdf, jpg, jpeg,
> png]`…) — la config centralisée ne fait que **surcharger**. Un service qui oublie une propriété reste
> fonctionnel et sûr.

### 5.2 Les variables d'environnement (nouvelles)

| Variable | Rôle |
|---|---|
| `SIGATT_MINIO_URL` | Endpoint de l'API MinIO (défaut dev : `http://localhost:9000`) |
| `SIGATT_MINIO_ACCESS_KEY` | Identifiant du compte technique du service |
| `SIGATT_MINIO_SECRET_KEY` | Secret du compte technique du service |

Même logique que les variables existantes (`SIGATT_DB_*`, `SIGATT_EUREKA_*`, `SIGATT_SYSTEM_*`) : les secrets
vivent dans l'environnement d'exécution, pas dans les dépôts.

### 5.3 Le `docker-compose.minio.yml` de développement, expliqué

```yaml
services:
  minio:
    image: minio/minio:RELEASE.2025-06-13T11-33-47Z   # tag FIGÉ (jamais :latest) → builds reproductibles
    command: server /data --console-address ":9001"    # démarre le serveur ; console web d'admin sur 9001
    ports:
      - "9000:9000"   # 9000 = l'API S3 (celle que les microservices appellent)
      - "9001:9001"   # 9001 = la console web (interface d'administration, pratique en dev)
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-sigatt-admin}         # compte administrateur racine —
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:?à définir}    # à surcharger, jamais les défauts minioadmin
    volumes:
      - minio-data:/data   # volume persistant : les documents survivent au redémarrage du conteneur
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]  # endpoint de vie natif MinIO
      interval: 30s
      timeout: 5s
      retries: 3

  # Conteneur d'initialisation (dev) : crée le bucket et active le versioning, puis s'arrête.
  minio-init:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set sigatt http://minio:9000 $$MINIO_ROOT_USER $$MINIO_ROOT_PASSWORD &&
      mc mb --ignore-existing sigatt/sigatt-immatriculation &&
      mc version enable sigatt/sigatt-immatriculation
      "

volumes:
  minio-data:
```

- **Pourquoi un tag figé ?** `latest` change sans prévenir → un poste qui fonctionne et un autre qui casse.
- **Pourquoi 2 ports ?** 9000 est l'API que consomment les applications ; 9001 est l'interface web humaine
  (voir les buckets, parcourir les objets — très utile pour déboguer en dev).
- **Pourquoi un volume ?** Sans volume, tout upload disparaît quand le conteneur est recréé.
- **Pourquoi `mc` ?** C'est le client en ligne de commande MinIO : il permet de scripter la création du bucket et
  l'activation du versioning — l'équivalent dev de ce que fera l'exploitation en prod.

En production : déploiement MinIO **distribué** (≥ 4 disques, prérequis du versioning/verrouillage), **TLS**,
comptes par service avec policies restreintes, supervision via `/minio/health/{live,ready}`.

---

## 6. Intégration dans SIGATT pas à pas

### 6.1 Le mécanisme d'auto-configuration (comment « ajouter la dépendance suffit »)

Le projet utilise déjà ce pattern pour ses deux libs : une classe de configuration listée dans
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` est chargée automatiquement
par Spring Boot **chez tout service qui a la lib dans son classpath** — les classes des libs n'étant pas
component-scannées (packages hors du package racine des apps), c'est LE mécanisme officiel.

La nouvelle lib déclarera :

```
sigatt.storage.commons.config.StorageAutoConfiguration
```

avec les garde-fous habituels du projet :
- `@ConditionalOnClass(MinioClient.class)` — ne s'active que si le SDK est là ;
- `@ConditionalOnProperty(prefix="sigatt.storage", name="enabled", matchIfMissing=true)` — désactivable par config ;
- `@ConditionalOnMissingBean` sur chaque bean — un service peut toujours surcharger un comportement ;
- `@EnableConfigurationProperties(StorageProperties.class)` — active la liaison `sigatt.storage.*`.

Beans fournis automatiquement : le client MinIO (singleton — il est thread-safe, une seule instance suffit),
le service de stockage `DocumentStorageService`, le validateur de fichiers (Tika), l'initialiseur de bucket
(dev), et un `HealthIndicator` (l'état de MinIO remonte dans `/actuator/health` du service).

### 6.2 Contenu de la lib `sigatt-lib-storage-commons`

```
sigatt-lib-storage-commons/
└─ src/main/java/sigatt/storage/commons/
   ├─ config/      StorageProperties, StorageAutoConfiguration, StorageBucketInitializer, StorageHealthIndicator
   ├─ service/     DocumentStorageService      (abstraction MinIO : televerser / telecharger / supprimer / existe)
   │               FichierValidator            (extension + magic bytes Tika + taille)
   │               AbstractDocumentService<E>  (orchestration : validation → MinIO → métadonnées)
   ├─ domain/      AbstractDocumentEntity      (@MappedSuperclass, hérite d'AbstractAuditEntity)
   ├─ dto/         DocumentWebDto
   └─ web/         AbstractDocumentController  (les 5 endpoints REST, Swagger + @PreAuthorize("@perm.canXxx(this)"))
└─ src/main/resources/META-INF/spring/…AutoConfiguration.imports
```

Philosophie identique au socle CRUD : **tout le comportement est dans la lib**, le microservice ne fournit que
des déclarations.

### 6.3 La recette « ajouter le stockage documentaire à un microservice » (~4 fichiers)

Pour `immatriculation-service` (et demain, à l'identique, `permis-conduire-service`) :

1. **`pom.xml`** — ajouter la dépendance `sigatt-lib-storage-commons` (opt-in explicite, règle des libs du projet).
2. **Entité** — `domain/DocumentEntity.java` : `@Entity @Table(name = "document") extends AbstractDocumentEntity`
   (la table est créée par Hibernate comme pour les référentiels).
3. **Repository** — `repository/DocumentRepository.java extends AbstractRepository<DocumentEntity, UUID>`
   (+ finder `findByDemandeIdAndDeletedFalse`).
4. **Service** — `service/impl/DocumentServiceImpl.java extends AbstractDocumentService<DocumentEntity>`
   (`@Service @Transactional`, fournit l'instanciation de l'entité concrète).
5. **Controller** — `web/DocumentController.java extends AbstractDocumentController`, `@RequestMapping("/api/documents")`,
   `@RequirePermissions(create/delete = {"document-write"}, read = {"default-roles-sigatt"})`.
6. **i18n** — clés `document.*` dans `i18/messages.properties` (+ `_en`) : `document.fichier.vide`,
   `document.extension.invalide`, `document.type.invalide`, `document.taille.max`.
7. **Rôles** — `document-write` dans `immatriculation-core-roles.json` (synchronisé dans Keycloak par uaa,
   comme tous les rôles).
8. **Config** — `bucket` + `auto-create-bucket` dans `immatriculation-service.yaml` (config-repos).

### 6.4 Le contrat REST (exposé par chaque service consommateur)

Toutes les routes passent par la gateway : `http://localhost:8081/immatriculation-service/api/documents/...`
(Bearer JWT obligatoire, comme le reste de l'API).

| Méthode & chemin | Rôle | Requête | Réponse |
|---|---|---|---|
| `POST /api/documents` | Enregistrer **1..n** documents | `multipart/form-data` : champ `fichiers` répété (1 à n fichiers), champ `demandeId` (UUID), champ `typePieceJointeId` (UUID, optionnel) | `201` + `[ {métadonnées}, … ]` |
| `GET /api/documents/demande/{demandeId}` | **Tous les documents d'une demande** | — | `200` + `[ {métadonnées}, … ]` |
| `GET /api/documents/{id}` | Métadonnées d'un document | — | `200` + `{métadonnées}` |
| `GET /api/documents/{id}/download` | **Télécharger** le fichier | — | `200`, corps binaire streamé, `Content-Type` réel, `Content-Disposition: attachment; filename="nom-original.pdf"` |
| `DELETE /api/documents/{id}` | Supprimer (logique) | — | `204` |

Exemple de métadonnées retournées :

```json
{
  "id": "9b1c2f30-…",
  "demandeId": "3fa85f64-…",
  "typePieceJointeId": "c56a4180-…",
  "nomOriginal": "carte-identite.pdf",
  "contentType": "application/pdf",
  "taille": 245812,
  "checksumSha256": "a3f5…",
  "createdBy": "agent.traore",
  "createdDate": "2026-07-01T10:15:30Z"
}
```

Erreurs : le modèle standard du projet (`SigattErrorResponse`) — `400` avec la clé i18n du problème
(`document.extension.invalide`, `document.taille.max`…) champ par champ, `404` via `error.ressource.not.found`,
`403` si le rôle `document-write` manque pour écrire.

### 6.5 Sécurité

- **Authentification** : inchangée — JWT Keycloak validé par la lib de sécurité existante, `/api/**` exige
  `default-roles-sigatt`.
- **Autorisation** : `@RequirePermissions` sur le contrôleur, appliqué par le bean `@perm` existant — écriture et
  suppression réservées au rôle **`document-write`**, lecture aux utilisateurs authentifiés du realm.
- **MinIO n'est jamais exposé** au navigateur ; ses credentials ne quittent jamais le backend.
- Les métadonnées d'audit (qui a déposé quoi, quand) sont automatiques via le socle.

### 6.6 Tests

Même stratégie que les 26 `*ControllerTest` existants (MockMvc + DbUnit + H2) : un `DocumentControllerTest` avec
le **`DocumentStorageService` mocké** — on valide toute la chaîne HTTP/validation/métadonnées (201 multi-fichiers,
400 extension/type/taille, listing par demande, download avec les bons en-têtes, delete 204) sans dépendre d'un
MinIO réel. Le test d'intégration avec un vrai MinIO (docker-compose) reste un test manuel/E2E documenté.

---

## 7. Bonnes pratiques et évolutions futures

| Évolution | Quand / pourquoi |
|---|---|
| **Versioning de bucket** | Activé dès le départ (dev via `mc`, prod au provisioning) : protège contre l'écrasement/suppression accidentels — chaque version d'objet est conservée |
| **Object Lock / rétention (WORM)** | Pour l'immuabilité réglementaire des pièces officielles : à activer **à la création du bucket** (impossible après), modes Governance/Compliance. Prérequis : MinIO distribué (≥ 4 disques) |
| **Chiffrement SSE + TLS** | En production : TLS vers MinIO obligatoire ; chiffrement au repos (SSE-S3/KMS) recommandé pour les données personnelles. Éviter SSE-C |
| **Presigned URLs** | Seulement si la volumétrie l'exige un jour : TTL 1–5 min, générées après contrôle JWT, MinIO exposé via reverse-proxy — jamais de TTL longs |
| **Antivirus (ClamAV)** | Si des documents proviennent directement du grand public : scanner à l'upload |
| **Job de réconciliation** | Tâche planifiée qui compare `listObjects` ↔ table `document` et signale/purge les objets orphelins (conséquence assumée du choix §4.8) |
| **Purge / rétention RGPD** | Règles de cycle de vie par bucket (expiration) pilotées par l'exploitation, alignées sur la politique de conservation des documents administratifs |
| **Métriques** | Compteurs/timers Micrometer par opération (upload/download/erreurs) exposés via Actuator |

---

## 8. Sources

- SDK Java MinIO — [github.com/minio/minio-java](https://github.com/minio/minio-java) (branche 9.x) ·
  [central.sonatype.com/artifact/io.minio/minio](https://central.sonatype.com/artifact/io.minio/minio)
- Organisation multi-applications des buckets — [MinIO Blog : Single vs Multi-Tenant](https://blog.min.io/single-vs-multi-tenant/) ·
  [docs.min.io — Multi-Tenancy](https://docs.min.io/aistor/administration/multi-tenancy/)
- Proxy vs presigned URLs — [G. Schwarz : Securing Private S3 Objects](https://georg-schwarz.com/blog/securing-s3-objects-backend-proxy-gateway-auth-presigned-urls/)
- Upload sécurisé (magic bytes, Tika) — [Secure File Upload with Spring Boot](https://www.oussemasahbeni.com/blog/secure-file-upload-spring-boot) ·
  [Baeldung : File MIME Type](https://www.baeldung.com/java-file-mime-type)
- Starter Spring Boot MinIO (inspiration) — [jlefebure/spring-boot-starter-minio](https://github.com/jlefebure/spring-boot-starter-minio)
- Versioning & immuabilité — [docs.min.io — Object Locking](https://docs.min.io/aistor/administration/object-locking-and-immutability/) ·
  [docs.min.io — Objects & Versioning](https://docs.min.io/aistor/administration/objects-and-versioning/)
- Image Docker — [hub.docker.com/r/minio/minio](https://hub.docker.com/r/minio/minio)
