# Implementation Plan: Appli web sur le téléphone, liste « à cuisiner » et courses vers Notion

**Branch**: `002-webapp-telephone-courses` | **Date**: 2026-11-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-webapp-telephone-courses/spec.md`

## Summary

Depuis le téléphone, sur le Wi-Fi de la maison : suivre le robot en direct, parcourir les
recettes, tenir une liste « à cuisiner » partagée avec le robot, générer la liste de courses et
l'envoyer dans Notion. Le téléphone ne commande jamais la machine.

Approche ([research.md](research.md)) : un **petit serveur HTTP écrit à la main** dans `core/web/`,
démarré par l'interface (`app.toque`) quand le réglage est activé, qui ne répond qu'au réseau
local et qu'aux **téléphones associés par un code** affiché sur le robot. L'appli web est en
**HTML/CSS/JS simples** dans les assets. La liste « à cuisiner » et la liste de courses sont des
**fichiers JSON** modifiés par opérations unitaires. Le **regroupement des ingrédients** et les
**messages pour Notion** sont en Kotlin pur et testés ; l'envoi HTTPS vers l'API Notion est fait
par le robot avec une intégration interne.

## Technical Context

**Language/Version**: Kotlin (JDK 17 pour la compilation, cible Android API 23–28) ; HTML, CSS et
JavaScript ES2017 sans compilation pour l'appli web (Safari iOS 12+, Chrome Android récent).

**Primary Dependencies**: aucune nouvelle. `java.net.ServerSocket`, `HttpURLConnection`,
`core/json/Json.kt`, `TextNorm` et `SearchIndex` existants. API Notion (version `2022-06-28`).

**Storage**: `files/telephones.json`, `files/a-cuisiner.json`, `files/courses.json` (data-model) ;
réglages `notionToken`, `notionPage`, `webEnabled`, `webPort` dans les préférences existantes.

**Testing**: JUnit dans `core/src/test/kotlin/app/toque/core/` (serveur, routes, association,
courses, Notion, i18n) ; banc d'essai émulateur de la spec 007 (quickstart §B) ; Notion réel
(quickstart §C) ; robot (quickstart §D).

**Target Platform**: Monsieur Cuisine Smart (Android 8.1) ; Connect best effort (rien de propre au
Smart) ; téléphones iPhone et Android sur le même Wi-Fi.

**Project Type**: appli Android (interface) + appli web servie par le robot.

**Performance Goals**: état affiché < 3 s après ouverture, retard < 2 s (SC-001) ; réponse de
recherche < 500 ms sur le robot ; aucun effet mesurable sur une cuisson (SC-006).

**Constraints**: lecture seule vis-à-vis de la machine (FR-003) ; réseau local seulement
(FR-013) ; désactivé par défaut (FR-015) ; secrets jamais renvoyés ni journalisés (FR-014) ;
4 connexions simultanées au plus ; pas de HTTPS sur le réseau local (research D2).

**Scale/Scope**: 1 à 3 téléphones (10 au plus), ~1 700 recettes, 50 recettes « à cuisiner » au
plus, 100 articles de courses au plus.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | État |
|---|---|---|
| I. Sécurité de cuisson | Le serveur ne reçoit que des fonctions de lecture (`Snapshot`, recettes, listes) ; aucune référence à `MachineClient` ; liste fermée des routes vérifiée par un test ; l'appli web n'a aucun bouton de cuisson (US2 scénario 4) ; aucune alerte n'est masquée sur le robot. | ✅ |
| II. Pilote isolé | Rien dans le pilote ; le serveur vit dans `app.toque`. Un plantage du serveur ne touche pas la cuisson (processus séparé). | ✅ |
| III. Cœur testable | HTTP, routes, accès, association, regroupement des ingrédients, messages Notion dans `core/` avec JUnit ; seuls les fils, les fichiers et l'envoi HTTPS sont côté Android. | ✅ |
| IV. Local d'abord | Désactivé par défaut ; réseau local seulement, adresses non privées refusées ; Notion facultatif, échec propre (liste gardée, copie en texte) ; jeton Notion jamais renvoyé, jamais journalisé, hors des sauvegardes. | ✅ |
| V. Sobriété | Pas de WebView sur le robot ; fils en basse priorité ; 4 connexions ; réponses de quelques Ko ; photos servies depuis le cache. Mesure SC-006 sur le robot. | ✅ |
| VI. Accessible et réversible | Textes de l'appli web et du robot en FR/EN/ES (test i18n étendu à `app.js`) ; guide complété (activer, associer, Notion). | ✅ |
| VII. Ne jamais briquer | Aucune modification d'Android, de l'appli Lidl ou d'un module (rien à inscrire au registre, research D12) ; arrêté en mode Lidl ; fichiers dans le dossier privé de l'appli. | ✅ |

**Re-check après la phase 1** : ✅. Les routes du [contrat](contracts/api.md) ne contiennent
aucune action sur la machine ; le seul secret (jeton Notion) n'apparaît dans aucune réponse.

## Project Structure

### Documentation (this feature)

```text
specs/002-webapp-telephone-courses/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── api.md
└── tasks.md             # /speckit-tasks
```

### Source Code (repository root)

```text
core/src/main/kotlin/app/toque/core/web/
├── HttpServer.kt        # ServerSocket, lecture des requêtes, limites, adresses privées, Host
├── Http.kt              # Request, Response, cookies, JSON
├── Routes.kt            # liste fermée des routes, contrôle d'accès, en-têtes de protection
├── Pairing.kt           # code à usage unique, jetons, empreintes, telephones.json
├── LiveState.kt         # état pour /api/etat à partir de Snapshot (+ recette suivie)
└── ToCook.kt            # liste « à cuisiner » : opérations, états, a-cuisiner.json
core/src/main/kotlin/app/toque/core/shopping/
├── Quantity.kt          # lecture des quantités et unités, conversions métriques exactes
├── ShoppingList.kt      # regroupement, portions, articles libres, courses.json
└── Notion.kt            # corps des requêtes Notion, lecture des réponses, codes d'erreur
core/src/test/kotlin/app/toque/core/
├── WebServerTest.kt
├── WebRoutesTest.kt
├── PairingTest.kt
├── ToCookTest.kt
├── ShoppingListTest.kt  # dont les 20 recettes d'essai (SC-004)
└── NotionTest.kt
core/src/test/resources/fixtures/courses-20-recettes.json   # recettes écrites pour le test

app/src/main/kotlin/app/toque/web/
├── WebService.kt        # démarre/arrête le serveur selon les réglages et le mode Lidl
├── AndroidWebBackend.kt # branche les routes sur RecipeRepo, Snapshot, fichiers, Notion
└── NotionClient.kt      # envoi HTTPS (Http.post/delete)
app/src/main/kotlin/app/toque/Http.kt                      # + post, delete
app/src/main/kotlin/app/toque/Prefs.kt                     # webEnabled, webPort, notion*
app/src/main/kotlin/app/toque/ToqueApp.kt                  # démarre WebService
app/src/main/kotlin/app/toque/ui/screens/SettingsScreen.kt # section Téléphone
app/src/main/kotlin/app/toque/ui/screens/HomeScreen.kt     # rangée « À cuisiner »
app/src/main/kotlin/app/toque/ui/screens/RecipeScreen.kt   # bouton « À cuisiner », portions
app/src/main/kotlin/app/toque/ui/screens/CookScreen.kt     # fin de recette : retirer de la liste ; recette suivie
app/src/main/kotlin/app/toque/backup/BackupWriter.kt       # + a-cuisiner.json (spec 007)
app/src/main/assets/web/{index.html,app.js,app.css,manifest.webmanifest,icone.png}
app/src/main/assets/lang/{en,es}.json
core/src/test/kotlin/app/toque/core/I18nTest.kt            # lit aussi assets/web/*.js
docs/TELEPHONE.md, docs/TELEPHONE.en.md                    # activer, associer, Notion
```

**Structure Decision**: pas de nouveau module Gradle. La logique va dans `core/` (nouveaux
paquets `web` et `shopping`), le branchement Android dans `app/…/web/`, l'appli web dans les
assets de l'interface. Le code et les commentaires nouveaux sont en anglais ; les textes
visibles, les noms de fichiers et de champs JSON (le contrat) restent en français.

## Complexity Tracking

Aucune violation de la constitution à justifier.
