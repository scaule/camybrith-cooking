# Quickstart : valider l'appli web sur le téléphone

## A. Tests du cœur (sans Android)

```
export JAVA_HOME=/opt/homebrew/opt/openjdk@17
./gradlew :core:test
```

Couvre : lecture des requêtes HTTP et limites (`WebServerTest`), routes fermées et contrôle
d'accès (`WebRoutesTest`), association et jetons (`PairingTest`), regroupement des ingrédients
sur les 20 recettes d'essai (`ShoppingListTest`, SC-004), corps envoyés à Notion
(`NotionTest`), textes de `app.js` traduits (`I18nTest`).

## B. Émulateur (banc d'essai de la spec 007)

1. `bash outils/banc-essai.sh preparer`, puis installer l'appli en mode démo.
2. Réglages › Téléphone : activer. Vérifier que l'adresse s'affiche.
3. Sur le Mac : `adb -s emulator-5580 forward tcp:8080 tcp:8080`, puis ouvrir
   `http://127.0.0.1:8080` dans Safari.

| # | Scénario | Attendu |
|---|---|---|
| W1 | ouvrir l'adresse sans association | écran « Saisissez le code affiché sur le robot », aucune donnée (`curl /api/etat` → 401) |
| W2 | Associer, saisir le code | accès à l'appli ; le téléphone apparaît dans la liste du robot |
| W3 | 5 codes faux | le code est annulé ; le bon code ne marche plus ; un nouveau code marche |
| W4 | cuisson manuelle 3 min en démo, couvercle ouvert | temps et température suivis, alerte en < 3 s (SC-001) |
| W5 | couper l'émulateur réseau (`adb emu network …` ou arrêt du service) | « Robot hors ligne » + heure de la dernière information |
| W6 | chercher « curry », ajouter, 6 portions | la recette est en tête de l'accueil cuisine du robot en < 5 s |
| W7 | retirer la recette sur le robot | elle disparaît du téléphone |
| W8 | 2 recettes avec carottes, générer, décocher sel | une ligne carottes additionnée ; sel décoché |
| W9 | Retirer le téléphone sur le robot | le navigateur revient à l'écran d'association |
| W10 | `curl -X POST` sans `X-Camybrith`, ou depuis une autre origine | 403 |
| W11 | chercher dans le code une route qui commande la machine | aucune (test `WebRoutesTest`) |

## C. Notion (compte de l'utilisateur)

1. notion.so/my-integrations : nouvelle intégration interne, copier le jeton.
2. Dans Notion, créer une page « Courses », menu ••• › Connexions › ajouter l'intégration.
3. Appli web › Réglages › Notion : coller le jeton, choisir « Courses », « Envoi d'essai » : une
   page « Essai Camybrith » apparaît.
4. Générer une liste, envoyer : page « Courses du <date> » avec des cases à cocher et les recettes.
5. Renvoyer le même jour : toujours une seule page du jour.
6. Retirer la connexion de l'intégration dans Notion, renvoyer : message « accès refusé » ;
   « Copier la liste » donne le texte.

## D. Sur le robot (après le root)

1. Réglages › Téléphone : activer, associer l'iPhone, l'ajouter à l'écran d'accueil (Safari ›
   Partager › Sur l'écran d'accueil).
2. Un deuxième téléphone non associé sur le même Wi-Fi : rien ne s'affiche (SC-005).
3. Cuisson manuelle de 3 minutes suivie par 2 téléphones : l'écran du robot reste fluide, la
   molette répond normalement (SC-006).
4. Redémarrer le robot : liste « à cuisiner » et téléphones conservés.
5. Retour à l'appli Lidl (spec 007) : l'adresse ne répond plus.
