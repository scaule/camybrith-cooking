# Data Model: Appli web sur le téléphone, liste « à cuisiner » et courses vers Notion

Fichiers dans le dossier privé de l'appli (`/data/data/app.toque/files/`), écrits de façon
atomique (research D7). Les noms de fichiers et de champs sont le contrat (en français, comme
ceux de la spec 007).

## Téléphone associé — `telephones.json`

```json
{ "telephones": [ { "id": "t3f9a1", "nom": "Mon iPhone", "empreinte": "<sha256 hex>",
    "associe": "2026-11-27T18:02:11+01:00", "dernierAcces": "2026-11-28T08:15:40+01:00" } ] }
```

| Champ | Règle |
|---|---|
| `id` | 6 caractères `[a-z0-9]`, unique, sert à retirer le téléphone |
| `nom` | 1 à 40 caractères, saisi sur le téléphone ; texte libre, échappé à l'affichage |
| `empreinte` | SHA-256 hexadécimal du jeton (64 caractères) ; le jeton lui-même n'est jamais stocké |
| `associe`, `dernierAcces` | date ISO 8601 ; `dernierAcces` mis à jour au plus une fois par minute |

Au plus 10 téléphones ; au-delà, l'association est refusée avec un message.

## Code d'association (en mémoire seulement)

| Champ | Règle |
|---|---|
| `code` | 6 chiffres, tirés par `SecureRandom` |
| `expire` | 5 minutes après l'affichage |
| `essais` | 5 faux au plus ; au 5e, le code est annulé |

États : *aucun* → (Réglages › Téléphone › Associer) → *affiché* → (bon code) → *utilisé* (jeton
remis, code annulé) ; *affiché* → (5 minutes ou 5 erreurs) → *annulé*. Jamais écrit sur disque ni
dans un journal.

## Recette à cuisiner — `a-cuisiner.json`

```json
{ "rev": 12, "recettes": [ { "id": "mc-12345", "titre": "Curry de légumes", "portions": 6,
    "ajoute": "2026-11-27T18:05:00+01:00", "etat": "a-cuisiner" } ] }
```

| Champ | Règle |
|---|---|
| `rev` | entier croissant, +1 à chaque changement |
| `id` | identifiant de recette du robot (`mc-…`, `web-…`, recette perso) ; unique dans la liste |
| `titre` | copié à l'ajout, pour afficher une recette « plus disponible » |
| `portions` | 1 à 24 ; par défaut les portions de la recette (4 si elle n'en indique pas) |
| `etat` | `a-cuisiner` · `cuisinee` · `plus-disponible` |

Transitions : ajout → `a-cuisiner` ; fin d'une recette suivie jusqu'au bout → `cuisinee` (le
robot propose de la retirer ; si on la garde, elle reste `cuisinee` et n'entre plus dans les
courses par défaut) ; recette absente du robot à la lecture → `plus-disponible` (affichée, pas
de courses) ; retour sur le robot → `a-cuisiner`. Ajouter une recette déjà présente ne crée pas de
doublon (les portions sont mises à jour). Au plus 50 recettes.

## Liste de courses — `courses.json`

```json
{ "date": "2026-11-27", "recettes": ["mc-12345", "mc-678"],
  "articles": [ { "cle": "carotte", "nom": "carottes", "quantites": [ {"valeur": 500, "unite": "g"} ],
      "texte": [], "recettes": ["mc-12345", "mc-678"], "coche": true, "libre": false } ],
  "notion": { "page": "<id de page>", "date": "2026-11-27", "envoye": "2026-11-27T18:20:00+01:00" } }
```

| Champ | Règle |
|---|---|
| `cle` | nom normalisé (research D8) ; unique dans la liste |
| `quantites` | une entrée par unité canonique ; `valeur` décimale, arrondie à 2 décimales à l'affichage |
| `texte` | quantités non additionnables gardées telles quelles (« 1-2 », « quelques ») |
| `coche` | `true` = à acheter (envoyé dans Notion) ; décoché = déjà dans les placards |
| `libre` | `true` pour un article ajouté à la main (« papier toilette ») ; pas de recette |
| `notion` | absent tant que rien n'est envoyé ; `page` permet de remplacer la liste du jour (D9) |

Régénérer la liste depuis « à cuisiner » garde les articles libres et l'état décoché des articles
qui existent encore.

## Connexion Notion — réglages (`shared_prefs/toque.xml`)

| Clé | Règle |
|---|---|
| `notionToken` | secret ; jamais renvoyé par l'API ni journalisé ; hors des sauvegardes (`SAFE_KEYS`) |
| `notionPage`, `notionPageTitre` | page choisie pour ranger les listes |
| `webEnabled` | `false` par défaut (FR-015) |
| `webPort` | 8080 par défaut, 1024 à 65535 |

État affiché : *non configurée* (pas de jeton) · *connectée* (dernier appel réussi) · *accès
refusé* (401/403/404 au dernier appel).

## État du robot pour l'appli web (calculé, jamais stocké)

Construit à chaque `GET /api/etat` à partir du dernier `Snapshot` (mêmes valeurs que
`RobotStatus.values`) et de la recette suivie dans l'interface : `statut`, `programme`,
`recette` (id, titre), `etape` (numéro, total, texte court), `restant` (s), `temperature`,
`temperatureCible`, `vitesse`, `poids`, `couvercle`, `alerte`, `heure` (date de l'information).
Voir [contracts/api.md](contracts/api.md).
