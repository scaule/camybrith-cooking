---

description: "Task list for 002-webapp-telephone-courses"
---

# Tasks: Appli web sur le téléphone, liste « à cuisiner » et courses vers Notion

**Input**: Design documents from `specs/002-webapp-telephone-courses/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.md, quickstart.md

**Tests**: pas de TDD demandé par la spec. Sont prévus : les tests JUnit imposés par la
constitution (principe III) pour tout ce qui est dans `core/`, et les scénarios W1–W11 sur
émulateur (quickstart §B), qui servent de tests d'intégration.

**Organization**: une phase par parcours de la spec. L'association (US4, P2) est une condition
d'accès à tout le reste (FR-012) : sa partie serveur est dans la phase Foundational, son écran
sur le robot et la connexion Notion dans la phase US4.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: parallélisable (fichiers différents, pas de dépendance sur une tâche non terminée)
- **[Story]**: parcours de la spec (US1 à US4)

## Path Conventions

Dépôt existant (plan.md › Project Structure) : `core/` (Kotlin pur, paquets `web` et `shopping`),
`app/` (interface), `app/src/main/assets/web/` (appli web). Rien dans `pilote/`. Code et
commentaires nouveaux en anglais ; textes visibles, fichiers et champs JSON en français.

---

## Phase 1: Setup

- [ ] T001 [P] Créer les paquets `core/src/main/kotlin/app/toque/core/web/` et `core/src/main/kotlin/app/toque/core/shopping/`, et `app/src/main/kotlin/app/toque/web/`
- [ ] T002 [P] Créer le squelette de l'appli web `app/src/main/assets/web/` : `index.html` (balises `apple-mobile-web-app-*`, viewport), `app.css`, `app.js` (fonction `t()` de traduction), `manifest.webmanifest`, `icone.png` (à partir de l'icône de l'appli)
- [ ] T003 [P] Étendre `core/src/test/kotlin/app/toque/core/I18nTest.kt` pour lire aussi les appels `t("…")` de `app/src/main/assets/web/*.js` (research D6)

---

## Phase 2: Foundational (serveur, accès, association)

**Purpose**: un serveur local, fermé aux inconnus, qui ne sait rien faire sur la machine.

- [ ] T004 Écrire `core/src/main/kotlin/app/toque/core/web/Http.kt` : `Request` (méthode, chemin, paramètres, en-têtes, cookies, corps), `Response` (code, type, en-têtes, corps texte ou octets), JSON et erreurs `{"erreur","message"}` (contracts/api.md › Règles communes)
- [ ] T005 Écrire `core/src/main/kotlin/app/toque/core/web/HttpServer.kt` : `ServerSocket`, lecture HTTP/1.1, limites (corps 64 Ko → 413, lecture 10 s, 4 connexions), refus des adresses non privées (research D2), contrôle de `Host` (421), fil de basse priorité fourni par l'appelant
- [ ] T006 Écrire `core/src/main/kotlin/app/toque/core/web/Pairing.kt` : code à 6 chiffres (5 min, 5 essais, usage unique), jeton 256 bits, empreinte SHA-256, 10 téléphones au plus, lecture/écriture de `telephones.json` via une interface de stockage (data-model.md › Téléphone associé, Code d'association)
- [ ] T007 Écrire `core/src/main/kotlin/app/toque/core/web/Routes.kt` : table **fermée** des routes du contrat, accès par cookie, `POST` + `application/json` + `X-Camybrith: 1` pour les modifications, `OPTIONS` → 403, fichiers statiques, `/api/textes`, `/api/associer`, `/api/dissocier` ; le serveur ne reçoit qu'une interface `Backend` en lecture (research D5)
- [ ] T008 [P] Tests `core/src/test/kotlin/app/toque/core/WebServerTest.kt` : requêtes valides, tronquées, trop grandes, `Host` refusé, adresse publique refusée, 5e connexion mise en attente
- [ ] T009 [P] Tests `core/src/test/kotlin/app/toque/core/PairingTest.kt` : bon code, code faux, 5 essais, expiration, réutilisation, jeton reconnu par son empreinte, retrait, limite de 10
- [ ] T010 [P] Tests `core/src/test/kotlin/app/toque/core/WebRoutesTest.kt` : 401 sans cookie sur chaque route protégée, 403 sans `X-Camybrith`, 404 hors liste, et vérification qu'aucune route ni le paquet `core/web` ne référence `MachineController` ou une commande machine (FR-003)
- [ ] T011 Ajouter `webEnabled` (faux par défaut) et `webPort` (8080) dans `app/src/main/kotlin/app/toque/Prefs.kt`
- [ ] T012 Écrire `app/src/main/kotlin/app/toque/web/WebService.kt` : démarre/arrête le serveur selon `webEnabled`, le mode Lidl (`Prefs.lidlMode`, spec 007) et le réseau ; stockage atomique (temporaire, `fsync`, renommage) dans `filesDir` ; journal sans secret ; branché dans `app/src/main/kotlin/app/toque/ToqueApp.kt`
- [ ] T013 Écrire `app/src/main/kotlin/app/toque/web/AndroidWebBackend.kt` : implémentation de `Backend` (fichiers statiques depuis les assets, textes traduits dans la langue du robot)
- [ ] T014 Écran d'association de l'appli web dans `app/src/main/assets/web/app.js` : saisie du code et du nom, message selon `code-faux`, `code-expire`, `trop-de-telephones` ; retour à cet écran sur toute réponse 401

**Checkpoint**: `curl` sans cookie → 401 ; avec le code affiché (Réglages provisoires) → cookie reçu.

---

## Phase 3: User Story 1 — Suivre le robot depuis le téléphone (P1) 🎯 MVP

**Goal**: l'état du robot en direct sur le téléphone.

**Independent Test**: quickstart W4 et W5.

- [ ] T015 [P] [US1] Écrire `core/src/main/kotlin/app/toque/core/web/LiveState.kt` : construit la réponse de `GET /api/etat` depuis `Snapshot` (mêmes valeurs que `RobotStatus.values`), la recette et l'étape suivies, l'alerte (couvercle, fin, erreur) et `rev` (data-model.md › État du robot)
- [ ] T016 [P] [US1] Tests `core/src/test/kotlin/app/toque/core/LiveStateTest.kt` : repos, cuisson, pause, alerte couvercle, base déconnectée (`enLigne: false`)
- [ ] T017 [US1] Exposer la recette et l'étape suivies depuis `app/src/main/kotlin/app/toque/ui/screens/CookScreen.kt` (lecture seule, sans changer la cuisson) et brancher `/api/etat` dans `AndroidWebBackend.kt` sur `machine.onSnapshot`
- [ ] T018 [US1] Écran « Robot » de l'appli web dans `app/src/main/assets/web/app.js` et `app.css` : interrogation chaque seconde (pause onglet caché), statut, programme, recette, étape, temps restant, températures, vitesse, poids, couvercle, alerte en évidence, « Robot hors ligne » + heure après 5 s sans réponse

**Checkpoint**: W4 et W5 passent sur l'émulateur.

---

## Phase 4: User Story 2 — Constituer sa liste « à cuisiner » (P1)

**Goal**: une liste partagée entre le téléphone et le robot, en tête de l'accueil cuisine.

**Independent Test**: quickstart W6 et W7.

- [ ] T019 [P] [US2] Écrire `core/src/main/kotlin/app/toque/core/web/ToCook.kt` : opérations ajouter, retirer, portions (1 à 24), marquer cuisinée, plus disponible, `rev`, 50 recettes au plus, lecture/écriture de `a-cuisiner.json` (data-model.md › Recette à cuisiner)
- [ ] T020 [P] [US2] Tests `core/src/test/kotlin/app/toque/core/ToCookTest.kt` : doublon, portions hors limites, deux suites d'opérations entrelacées sans perte, recette disparue, fichier illisible
- [ ] T021 [US2] Routes `/api/recettes`, `/api/recettes/<id>`, `/photo/<nom>`, `/api/a-cuisiner*` dans `Routes.kt`, branchées sur `RecipeRepo` et `SearchIndex` dans `AndroidWebBackend.kt` (30 par page, photos depuis `RecipeRepo.imageFile`)
- [ ] T022 [US2] Écrans « Recettes » (recherche, fiche en lecture seule, sans bouton de cuisson) et « À cuisiner » (portions, retrait) dans `app/src/main/assets/web/app.js` ; rechargement quand `rev` change
- [ ] T023 [US2] Rangée « À cuisiner » en tête de `app/src/main/kotlin/app/toque/ui/screens/HomeScreen.kt` (photo, titre, portions, « plus disponible »), utilisable à la molette
- [ ] T024 [US2] Bouton « À cuisiner » et choix des portions dans `app/src/main/kotlin/app/toque/ui/screens/RecipeScreen.kt`
- [ ] T025 [US2] Fin d'une recette de la liste suivie jusqu'au bout dans `CookScreen.kt` : question « Retirer de la liste à cuisiner ? » (oui → retirée, non → `cuisinee`)
- [ ] T026 [US2] Ajouter `a-cuisiner.json` (facultatif) au contenu des sauvegardes dans `app/src/main/kotlin/app/toque/backup/BackupWriter.kt` et à la restauration dans `BackupRestorer.kt` (spec 007, research D7)

**Checkpoint**: W6 et W7 passent.

---

## Phase 5: User Story 3 — Liste de courses et envoi dans Notion (P2)

**Goal**: ingrédients regroupés, décochage, envoi dans Notion, repli en texte.

**Independent Test**: quickstart W8 et §C 4 à 6.

- [ ] T027 [P] [US3] Écrire `core/src/main/kotlin/app/toque/core/shopping/Quantity.kt` : lecture des quantités (entiers, décimaux, fractions, nombres mixtes ; fourchettes et textes non additionnables), unités canoniques, conversions métriques exactes seulement, affichage (research D8)
- [ ] T028 [US3] Écrire `core/src/main/kotlin/app/toque/core/shopping/ShoppingList.kt` : regroupement par nom normalisé (`TextNorm` + pluriel), facteur de portions, articles sans quantité, articles libres, cocher, régénération qui garde libres et décochés, tri, texte à copier, lecture/écriture de `courses.json` (data-model.md › Liste de courses)
- [ ] T029 [P] [US3] Écrire le jeu d'essai `core/src/test/resources/fixtures/courses-20-recettes.json` : 20 recettes écrites pour le test, au format et avec les unités des recettes Lidl (aucune recette de Lidl copiée)
- [ ] T030 [US3] Tests `core/src/test/kotlin/app/toque/core/ShoppingListTest.kt` : scénarios US3 1 et 2, cas limites (sel sans quantité, oignon/oignons/oignon jaune), portions ×1,5, et SC-004 sur les 20 recettes (toutes les sommes justes, aucune conversion inventée)
- [ ] T031 [P] [US3] Écrire `core/src/main/kotlin/app/toque/core/shopping/Notion.kt` : corps de création de page (titre daté, `to_do` par article coché, paragraphe des recettes, 100 blocs au plus), recherche de pages, lecture des réponses et codes d'erreur `notion-refus|notion-page|notion-reseau|notion-trop-long` (research D9)
- [ ] T032 [P] [US3] Tests `core/src/test/kotlin/app/toque/core/NotionTest.kt` : corps attendu, articles décochés absents, limite de 100, réponses 200/401/403/404, aucun jeton dans les messages d'erreur
- [ ] T033 [US3] Ajouter `post` et `delete` à `app/src/main/kotlin/app/toque/Http.kt` et écrire `app/src/main/kotlin/app/toque/web/NotionClient.kt` (`https://api.notion.com` seulement, délai 15 s, remplacement de la page du jour)
- [ ] T034 [US3] Routes `/api/courses*` dans `Routes.kt` et `AndroidWebBackend.kt`
- [ ] T035 [US3] Écran « Courses » dans `app/src/main/assets/web/app.js` : générer, cocher/décocher, articles libres, envoyer, message d'échec avec « Réessayer » et « Copier la liste »

**Checkpoint**: W8 passe ; §C 4 à 6 passent avec un vrai compte Notion.

---

## Phase 6: User Story 4 — Associer le téléphone et relier Notion (P2)

**Goal**: mise en route faite une fois, depuis le robot et le téléphone.

**Independent Test**: quickstart W1 à W3, W9, §C 1 à 3.

- [ ] T036 [US4] Section « Téléphone » dans `app/src/main/kotlin/app/toque/ui/screens/SettingsScreen.kt` : interrupteur, adresse à ouvrir, « Associer un téléphone » (code, temps restant, fermeture après association), liste des téléphones avec « Retirer », état Notion et « Oublier la connexion Notion » (contracts/api.md › Côté robot)
- [ ] T037 [US4] Ajouter `notionToken`, `notionPage`, `notionPageTitre` dans `Prefs.kt` ; vérifier qu'ils restent hors des sauvegardes (`Secrets.SAFE_KEYS`) et hors du journal
- [ ] T038 [US4] Routes `/api/notion*` dans `Routes.kt` et `AndroidWebBackend.kt` (vérification du jeton, liste des pages, choix, envoi d'essai, oubli) ; le jeton n'apparaît dans aucune réponse
- [ ] T039 [US4] Écran « Réglages » de l'appli web dans `app/src/main/assets/web/app.js` : coller le jeton, choisir la page, envoi d'essai, se dissocier, aide « ajouter à l'écran d'accueil »

**Checkpoint**: W1 à W3, W9, W10 passent ; §C 1 à 3 passent.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T040 [P] Traductions anglais et espagnol de tous les nouveaux textes (robot et appli web) dans `app/src/main/assets/lang/en.json` et `es.json`
- [ ] T041 [P] Guide `docs/TELEPHONE.md` et `docs/TELEPHONE.en.md` (activer, associer, ajouter à l'écran d'accueil, créer l'intégration Notion, risque du HTTP sur le Wi-Fi de la maison) ; liens depuis `README.md`, `README.en.md`, `README.es.md`
- [ ] T042 [P] Entrée dans `CHANGELOG.md` (section 1.5.0, français et anglais)
- [ ] T043 Passer les scénarios W1 à W11 sur l'émulateur (quickstart §B) et noter le résultat dans ce fichier
- [ ] T044 Mesure SC-006 sur l'émulateur : cuisson démo avec 2 clients qui interrogent `/api/etat` chaque seconde, journal de la boucle machine sans écart
- [ ] T045 `./gradlew :core:test :app:assembleRelease` et CI au vert

---

## Dependencies & Execution Order

- **Setup (1)** → **Foundational (2)** → US1, US2, US3, US4.
- **US1** et **US2** sont indépendants une fois la phase 2 faite.
- **US3** dépend de US2 (la liste de courses part de la liste « à cuisiner ») ; T027–T032
  (`core/`) peuvent commencer dès la phase 2.
- **US4** : T036 (écran d'association sur le robot) est utile dès la phase 2 pour les essais ; la
  partie Notion (T037–T039) précède l'envoi réel de US3 (T033–T035 se testent avec un faux
  client avant).
- **Polish** en dernier.

## Parallel Example

```text
Phase 2 : T008, T009, T010 en parallèle après T004–T007.
Phase 5 : T027, T029, T031, T032 en parallèle (fichiers différents de core/).
```

## Implementation Strategy

1. **MVP** : phases 1, 2, 3 et T036 (association depuis le robot) → suivre le robot sur
   l'iPhone. Une PR.
2. US2 → une PR ; US3 + US4 → une PR (Notion de bout en bout) ; Polish → une PR.
3. Chaque PR : `:core:test` en local, CI au vert, puis fusion (fork uniquement).
