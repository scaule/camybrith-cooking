# Research: Appli web sur le téléphone, liste « à cuisiner » et courses vers Notion

Décisions prises pour le plan. Chaque entrée : décision, raison, alternatives écartées.

## D1. Serveur web écrit à la main, dans `core/`

- **Décision** : un petit serveur HTTP/1.1 en Kotlin pur (`java.net.ServerSocket`), dans
  `core/web/` : lecture de la requête, routage, réponses JSON, fichiers statiques. L'appli
  (`app.toque`) le démarre sur un fil de basse priorité, comme la publication Home Assistant.
- **Raison** : le projet n'a aucune dépendance externe (JSON, MQTT, zip faits main) ; le besoin est
  minuscule (quelques routes, 1 à 3 téléphones) ; tout le traitement (analyse, routage, contrôle
  d'accès) est testable en JUnit sans Android (principe III).
- **Alternatives** : NanoHTTPD (dépendance, moins de contrôle sur les limites), Ktor (lourd pour la
  tablette, Kotlin récent), serveur dans le pilote (interdit, principe II).

## D2. Réseau local uniquement, HTTP simple, protections associées

- **Décision** : écoute sur le port **8080** (réglable), toutes interfaces, mais toute connexion
  dont l'adresse source n'est pas privée (10/8, 172.16/12, 192.168/16, 169.254/16, fe80::/10,
  fc00::/7, boucle locale) est fermée sans réponse. L'en-tête `Host` doit être une adresse IP ou
  `<nom>.local` : protection contre le *DNS rebinding*. Pas de HTTPS.
- **Raison** : FR-013 (pas d'accès depuis Internet) ; le robot est derrière la box ; un certificat
  valable pour une IP locale est impossible sans infrastructure, et un certificat auto-signé
  ferait peur à l'utilisateur à chaque ouverture.
- **Risque accepté** : sur le Wi-Fi de la maison, le trafic est lisible par un appareil du même
  réseau. Le jeton du téléphone n'en sort pas (FR-014 : le jeton Notion n'est jamais renvoyé au
  téléphone). Documenté dans le guide.
- **Alternatives** : HTTPS auto-signé (avertissements, iOS refuse l'ajout à l'écran d'accueil
  proprement), relais distant (c'est la spec 003).

## D3. Association par code à usage unique et jeton de téléphone

- **Décision** : Réglages › Téléphone affiche un **code de 6 chiffres**, valable 5 minutes, à
  usage unique, et l'adresse à ouvrir. Le téléphone envoie le code et un nom ; le robot répond
  avec un **jeton aléatoire de 256 bits** placé dans un cookie `HttpOnly; SameSite=Strict`, valable
  1 an. Le robot ne garde que l'**empreinte SHA-256** du jeton. 5 codes faux de suite : le code est
  annulé, il faut en afficher un nouveau sur le robot. Retirer un téléphone efface son empreinte.
- **Raison** : FR-012, SC-005 ; un code de 6 chiffres limité à 5 essais donne 1 chance sur
  200 000 ; un jeton volé dans les journaux est inutilisable puisqu'il n'y est jamais écrit.
- **Protection des actions** : toute requête qui modifie quelque chose est un `POST` avec
  `Content-Type: application/json` et l'en-tête `X-Camybrith: 1` (une page d'un autre site ne peut
  pas l'envoyer sans pré-vol CORS, que le serveur refuse).
- **Alternatives** : mot de passe (à retenir, faible), QR code (pas de bibliothèque ; l'adresse
  et le code se tapent en 15 secondes ; un QR code pourra venir plus tard).

## D4. Suivi en direct par interrogation chaque seconde

- **Décision** : l'appli web demande `GET /api/etat` toutes les secondes (pause quand l'onglet est
  caché). Réponse de moins de 1 Ko construite depuis le dernier `Snapshot`, avec les mêmes
  valeurs que `RobotStatus` (Home Assistant), plus la recette et l'étape en cours.
- **Raison** : SC-001 (< 2 s de retard) ; aucune connexion longue qui bloquerait un fil du
  serveur ; plus robuste que Server-Sent Events quand le Wi-Fi coupe ou que le téléphone se met en
  veille. Coût mesuré sur 2 téléphones : 2 petites requêtes par seconde.
- **Hors ligne** : l'appli web garde la dernière réponse et son heure ; après 5 s sans réponse,
  elle affiche « Robot hors ligne » et l'heure de la dernière information (US1, scénario 3).
- **Alternatives** : SSE (fil bloqué par téléphone, reconnexions), WebSocket (protocole à écrire,
  pas d'intérêt pour 1 Ko/s).

## D5. Lecture seule vis-à-vis de la machine, imposée par la construction

- **Décision** : le serveur ne reçoit **que** des fonctions de lecture : `() -> Snapshot?` et
  l'accès aux recettes et à la liste. Il n'a aucune référence à `MachineClient` ni au pilote. Un
  test JUnit vérifie la liste fermée des routes et qu'aucune ne commande la machine ; un test
  d'architecture vérifie que `core/web` n'importe pas `machine.MachineController`.
- **Raison** : FR-003, principe I : rien venant du réseau ne peut démarrer le moteur ou la chauffe.
  La spec 002 ne prépare même pas de programme : le téléphone ne fait que consulter.

## D6. Fichiers de l'appli web : HTML, CSS et JavaScript simples, dans les assets

- **Décision** : `app/src/main/assets/web/` : `index.html`, `app.css`, `app.js` (sans
  framework, sans étape de compilation), servis avec `Cache-Control` et une empreinte de version.
  Ajout à l'écran d'accueil par les balises `apple-mobile-web-app-*` et un `manifest.webmanifest`
  avec une icône.
- **Raison** : FR-001 ; rien à compiler, rien à télécharger ; quelques dizaines de Ko. Un
  *service worker* exige HTTPS, donc pas de mode hors ligne dans le navigateur ; ce n'est pas
  demandé (hors de la maison, la liste est dans Notion).
- **Langue** : l'appli web demande `GET /api/textes`, qui renvoie la traduction de ses textes
  dans la langue du robot. Les textes sont écrits `t("Texte en français")` dans `app.js` ; le test
  `I18nTest` est étendu pour lire aussi `assets/web/*.js` (FR-016, principe VI).

## D7. Liste « à cuisiner » et liste de courses : fichiers JSON, opérations unitaires

- **Décision** : `files/a-cuisiner.json` et `files/courses.json`, écrits de façon atomique
  (fichier temporaire, `fsync`, renommage). Le téléphone envoie des **opérations** (ajouter,
  retirer, changer les portions, cocher) et non la liste entière ; le robot les applique l'une
  après l'autre sous un verrou. Une version (`rev`) incrémentée à chaque changement permet aux
  écrans de se rafraîchir.
- **Raison** : cas limite « deux téléphones en même temps » : la dernière modification gagne
  sans perdre les ajouts de l'autre ; redémarrage : tout est conservé ; pas de migration de
  `recettes.db`.
- **Sauvegarde (spec 007)** : `a-cuisiner.json` est ajouté au contenu des sauvegardes (nouvelle
  entrée facultative ; une sauvegarde sans elle reste valable, pas de changement de format).
- **Alternatives** : table SQLite (migration de la base, peu d'intérêt pour quelques dizaines de
  lignes).

## D8. Regroupement des ingrédients

- **Décision** : `core/shopping/` :
  - **Quantités** lues : entiers, décimaux (`1,5` ou `1.5`), fractions (`1/2`, `½`), nombres mixtes
    (`1 1/2`). Une fourchette (`1-2`), « quelques », un texte libre : non additionnable, gardé tel
    quel.
  - **Portions** : facteur = portions voulues / portions de la recette (1 si la recette n'indique
    pas de portions). Appliqué aux quantités lisibles seulement.
  - **Unités** : forme canonique (`g`, `gramme`, `grammes` → `g` ; `c. à s.`, `cuillère à soupe`
    → `c. à s.`…). Seules les **conversions exactes du système métrique** sont faites : `kg`→`g`,
    `l`/`cl`/`dl`→`ml`. Aucune autre conversion (pièce ↔ grammes, cuillère ↔ ml) : FR-008.
    Affichage : `1 200 g` devient `1,2 kg` à partir de 1 000.
  - **Noms** : clé = `TextNorm` existant (minuscules, accents retirés) + pluriel simple retiré
    (`oignons` = `oignon`). « oignon jaune » reste une ligne à part (spec : sinon lignes
    séparées). Le nom affiché est celui de la première occurrence.
  - **Sans quantité** (« sel ») : une ligne, sans quantité.
  - Tri alphabétique sur la clé.
- **Jeu d'essai SC-004** : 20 recettes **écrites pour le test** au format des recettes Lidl
  (unités et quantités relevées dans les recettes réelles), dans `core/src/test/resources/` : les
  recettes de Lidl ne doivent pas être ajoutées au dépôt (constitution).

## D9. Notion : intégration interne, une page par liste

- **Décision** :
  - L'utilisateur crée une **intégration interne** sur notion.so (jeton `ntn_…` ou `secret_…`),
    la relie à une page de son espace (« Courses »), puis **colle le jeton dans l'appli web** sur
    le téléphone (le taper sur le robot serait pénible). Le robot le garde dans ses réglages ; il
    n'est plus jamais renvoyé au téléphone, ni écrit dans un journal (FR-014). Il est déjà exclu
    des sauvegardes par la liste fermée `SAFE_KEYS` (spec 007).
  - Choix de l'emplacement : le robot appelle `POST /v1/search` (filtre `page`) et le téléphone
    choisit parmi les pages partagées avec l'intégration.
  - Envoi : `POST /v1/pages` sous la page choisie, titre « Courses du 27/11/2026 », contenu : un
    bloc `to_do` par article (texte « 500 g carottes »), puis un paragraphe « Recettes : … ». Au
    plus 100 blocs par création (limite de l'API) ; au-delà, refus clair (une liste de 100
    articles n'arrive pas en pratique).
  - **Second envoi du même jour** : la page précédente du jour (son identifiant est gardé dans
    `courses.json`) est mise à la corbeille par `DELETE /v1/blocks/{id}`, puis la nouvelle est
    créée : il n'y a toujours qu'une liste du jour.
  - En-têtes : `Authorization: Bearer`, `Notion-Version: 2022-06-28`. Délai 15 s.
  - **Échec** : 401/403 → « accès refusé, vérifiez la connexion Notion » ; 404 → « page Notion
    introuvable » ; réseau → « Internet injoignable ». La liste reste sur le robot ; le téléphone
    propose « Réessayer » et « Copier la liste » (texte prêt à coller) : FR-011.
- **Raison** : FR-010, FR-017, clarification « une page par liste, cases à cocher ». Pas de
  `PATCH` nécessaire (le `HttpURLConnection` d'Android l'accepte, mais on l'évite).
- **Le message construit pour Notion** (corps JSON) est produit en `core/` et testé ; seul l'envoi
  HTTPS est côté Android (`Http.kt` complété par `post` et `delete`).
- **Alternatives** : OAuth public (application à enregistrer chez Notion, serveur de rappel),
  base de données Notion (plus lourd à choisir pour l'utilisateur ; la clarification a tranché).

## D10. Robot : où apparaît la liste « à cuisiner »

- **Décision** : en tête de l'accueil cuisine (`HomeScreen`), une rangée « À cuisiner » avec les
  recettes de la liste (photo, titre, portions). Sur l'écran d'une recette : bouton « À cuisiner »
  (ajouter ou retirer) et choix des portions. À la fin d'une recette de la liste suivie jusqu'au
  bout, une question : « Retirer de la liste à cuisiner ? ». Une recette disparue après une synchro
  Lidl reste dans la liste, marquée « plus disponible ».
- **Raison** : FR-005, FR-006, US2 scénario 3 ; complète l'écran Maison de la spec 001 sans en
  dépendre.

## D11. Sobriété et non-gêne de la cuisine

- **Décision** : service **désactivé par défaut** (FR-015) ; fils du serveur en
  `THREAD_PRIORITY_BACKGROUND`, **4 connexions** à la fois au plus (les suivantes attendent),
  délai de lecture 10 s, requête limitée à 64 Ko, réponses de recherche limitées à 30 recettes,
  photos servies depuis le cache déjà présent (`RecipeRepo.imageFile`) sans retraitement. Arrêté
  en mode Lidl (spec 007) et quand le réglage est coupé.
- **Mesure SC-006** : sur l'émulateur puis le robot, une cuisson manuelle avec 2 téléphones qui
  interrogent : aucun écart dans le journal de la boucle machine (`MachineLoop`), qui tourne
  dans le pilote, processus séparé (principe II).

## D12. Registre des modifications (spec 007)

- Ouvrir un port dans l'appli ne modifie ni Android, ni l'appli Lidl, ni un module : rien à
  inscrire dans le registre. Le réglage s'éteint avec l'appli ; le retour à l'appli Lidl arrête
  le service de fait.
