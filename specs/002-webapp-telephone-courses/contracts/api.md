# Contrat : API HTTP de l'appli web (robot, réseau local)

Servie par `app.toque` sur `http://<ip-du-robot>:8080/` quand Réglages › Téléphone est activé.
Toutes les réponses de l'API sont en JSON UTF-8, avec `Cache-Control: no-store`. Liste **fermée** :
toute autre route répond 404. Aucune route ne commande le moteur, la chauffe ou la balance
(FR-003, research D5).

## Règles communes

- **Adresse source non privée** : connexion fermée sans réponse (research D2).
- **En-tête `Host`** différent d'une adresse IP ou d'un nom `*.local` : `421`.
- **Accès** : toutes les routes sauf `/`, `/app.*`, `/manifest.webmanifest`, `/icone.png`,
  `GET /api/textes` et `POST /api/associer` demandent le cookie `camybrith=<jeton>` d'un téléphone
  associé ; sinon `401 {"erreur":"non-associe"}`, et l'appli web affiche l'écran d'association.
- **Modifications** : `POST` seulement, avec `Content-Type: application/json` et
  `X-Camybrith: 1` ; sinon `403`. Pas d'en-tête CORS ; les requêtes `OPTIONS` répondent `403`.
- **Limites** : corps ≤ 64 Ko (`413`), 4 connexions simultanées, lecture ≤ 10 s.
- **Erreurs** : `{"erreur": "<code>", "message": "<texte traduit>"}`. Jamais de secret dans un
  message.
- **Mode Lidl** (spec 007) ou réglage coupé : le serveur n'écoute pas.

## Fichiers statiques

| Route | Contenu |
|---|---|
| `GET /` | `index.html` |
| `GET /app.js`, `GET /app.css` | appli web (`?v=<versionCode>` pour le cache) |
| `GET /manifest.webmanifest`, `GET /icone.png` | ajout à l'écran d'accueil |
| `GET /photo/<nom>` | photo de recette depuis le cache du robot (associé seulement), `Cache-Control: max-age=86400` |

## Textes

`GET /api/textes` → `{"langue":"fr","textes":{"Robot hors ligne":"Robot offline", …}}` : les
traductions des textes de `app.js` dans la langue du robot (vide en français).

## Association (US4)

`POST /api/associer` `{"code":"482913","nom":"Mon iPhone"}`

- `200 {"ok":true,"id":"t3f9a1"}` + `Set-Cookie: camybrith=<jeton>; HttpOnly; SameSite=Strict;
  Max-Age=31536000; Path=/`
- `400 {"erreur":"code-faux"}` (essais restants non indiqués) · `410 {"erreur":"code-expire"}`
  (pas de code affiché, expiré ou annulé) · `409 {"erreur":"trop-de-telephones"}`

`POST /api/dissocier` `{}` → `200` : retire le téléphone qui fait la demande et efface le cookie.

## État du robot (US1)

`GET /api/etat` →

```json
{ "enLigne": true, "heure": "2026-11-27T18:12:03+01:00", "statut": "running",
  "programme": "Soupe", "recette": {"id": "mc-12345", "titre": "Velouté de potiron"},
  "etape": {"numero": 3, "total": 7, "texte": "Mixer 1 min vitesse 8"},
  "restant": 1260, "temperature": 92, "temperatureCible": 100, "vitesse": 2, "poids": 1240,
  "couvercle": "ferme", "alerte": null, "rev": 12 }
```

`statut` : `idle` · `running` · `paused` · `fault` · `disconnected` (mêmes valeurs que Home
Assistant). `alerte` : `null` ou `{"type": "couvercle|fin|erreur", "message": "…"}`. `rev` : version
de la liste « à cuisiner », pour que le téléphone la recharge quand elle change. `enLigne: false`
quand la base ne répond plus (le téléphone affiche aussi « hors ligne » s'il n'obtient pas de
réponse pendant 5 s).

## Recettes (US2)

- `GET /api/recettes?q=curry&page=0` → `{"recettes":[{"id","titre","photo","note","notes",
  "minutes","portions","aCuisiner"}], "suite": true}` : 30 par page, recherche du robot
  (`SearchIndex`, accents, pluriels, fautes). Sans `q` : les mieux notées.
- `GET /api/recettes/<id>` → la recette en lecture : titre, photo, portions, ingrédients groupés,
  étapes (texte et réglages affichés), note. **Aucun bouton de cuisson** dans l'appli web.
  `404` si elle n'existe plus.

## Liste « à cuisiner » (US2)

- `GET /api/a-cuisiner` → `{"rev":12,"recettes":[{"id","titre","photo","portions",
  "portionsRecette","etat"}]}`
- `POST /api/a-cuisiner/ajouter` `{"id":"mc-12345","portions":6}` → la liste
- `POST /api/a-cuisiner/retirer` `{"id":"mc-12345"}` → la liste
- `POST /api/a-cuisiner/portions` `{"id":"mc-12345","portions":6}` → la liste (1 à 24, sinon `400`)

## Liste de courses (US3)

- `POST /api/courses/generer` `{}` → la liste (data-model `courses.json`), à partir des recettes
  `a-cuisiner`
- `GET /api/courses` → la liste en cours (`404 {"erreur":"aucune-liste"}` si jamais générée)
- `POST /api/courses/cocher` `{"cle":"sel","coche":false}` → la liste
- `POST /api/courses/ajouter` `{"nom":"papier toilette"}` → la liste (article libre, 1 à
  60 caractères)
- `POST /api/courses/retirer` `{"cle":"papier toilette"}` → la liste (articles libres seulement)
- `GET /api/courses/texte` → `text/plain` : la liste cochée, une ligne par article, prête à copier
- `POST /api/courses/envoyer` `{}` → `200 {"ok":true,"page":"<url Notion>","remplacee":false}` ;
  `502 {"erreur":"notion-refus|notion-page|notion-reseau|notion-trop-long","message":…}` ;
  `409 {"erreur":"notion-non-configure"}`

## Notion (US4)

- `GET /api/notion` → `{"etat":"non-configuree|connectee|acces-refuse","page":{"id","titre"}}`
  (**jamais le jeton**)
- `POST /api/notion/jeton` `{"jeton":"ntn_…"}` → vérifie le jeton (`GET /v1/users/me`), le garde,
  `200 {"etat":"connectee"}` ou `400 {"erreur":"notion-refus"}`. Le jeton n'est jamais renvoyé.
- `GET /api/notion/pages` → `{"pages":[{"id","titre"}]}` : pages partagées avec l'intégration
- `POST /api/notion/page` `{"id":"…"}` → l'emplacement choisi
- `POST /api/notion/essai` `{}` → crée une page « Essai Camybrith » sous l'emplacement, avec un
  article coché, et renvoie son URL (US4 scénario 3)
- `POST /api/notion/oublier` `{}` → efface le jeton et l'emplacement

## Côté robot (Réglages › Téléphone)

- interrupteur « Appli web sur le téléphone » (désactivée par défaut), adresse à ouvrir
  (`http://192.168.1.42:8080`) ;
- « Associer un téléphone » : affiche le code, le temps restant, et se ferme seul après
  l'association ;
- liste des téléphones associés (nom, date, dernier accès) avec « Retirer » ;
- état de la connexion Notion et « Oublier la connexion Notion ».

## Notion : requêtes envoyées par le robot

`Authorization: Bearer <jeton>`, `Notion-Version: 2022-06-28`, `Content-Type: application/json`,
délai 15 s, `https://api.notion.com` seulement.

| Usage | Requête |
|---|---|
| vérifier le jeton | `GET /v1/users/me` |
| lister les pages | `POST /v1/search` `{"filter":{"property":"object","value":"page"},"page_size":50}` |
| créer la liste | `POST /v1/pages` `{"parent":{"page_id":…},"properties":{"title":{"title":[…]}},"children":[to_do…, paragraph]}` |
| remplacer la liste du jour | `DELETE /v1/blocks/<page du jour>` puis création |
