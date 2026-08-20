# Reprise de session — Roul'trouv (chasse-au-tresor.html)

Dernière mise à jour : 2026-08-18

## Contexte du projet

Application web (`chasse-au-tresor.html`) pour créer et imprimer une fiche de
« chasse au trésor de la route » (jeu d'observation pour les trajets en
voiture). L'utilisateur choisit des éléments à repérer par la fenêtre
(véhicules, panneaux, animaux, etc.), chacun ayant une rareté (1/3/5 pts),
plus optionnellement des défis bonus, puis génère une fiche imprimable.

⚠️ L'app n'est plus un fichier 100% autonome depuis l'externalisation du
catalogue (voir plus bas) : elle a besoin d'un serveur HTTP pour charger
`data/items.json`/`data/bonus.json` via `fetch()` (marche sur GitHub Pages ou
`http-server` en local ; **pas** en ouvrant le fichier en double-clic —
`fetch()` échoue en `file://`, mais l'app reste utilisable grâce au repli
intégré, voir plus bas).

Le dépôt contient aussi :
- `data/items.json`, `data/bonus.json` : catalogue externalisé, éditable
  depuis l'app (voir section dédiée plus bas)
- `icons/` : images PNG personnalisées, poussées via l'API GitHub depuis l'app
- `maquette-demipage-validee.html` : maquette de référence figée du mode demi-page
- `.git`

⚠️ **Deux copies locales existent** selon le PC utilisé :
- `C:\Users\antony.durand\ONEPOINT\Orga Perso - General\99.Perso\Projets\Roule & Trouve\Roule-Trouve`
- `D:\Documents\Créations\Roule-Trouve`

**Faire un `git pull` avant de commencer** pour éviter de diverger entre les deux.

---

## ⭐ Point d'architecture le plus important

**La géométrie de la fiche est définie UNE SEULE FOIS, en millimètres, hors
`@media print`.** Chercher le bloc commenté :

```
/* ============================================================
   GEOMETRIE PARTAGEE ECRAN + IMPRESSION
   ...
   >>> C'EST ICI qu'on modifie la fiche.
   ============================================================ */
```

Conséquence : **l'aperçu écran et le PDF sont rigoureusement identiques**.
Toute modification de taille/police/grille se fait dans ce bloc et s'applique
aux deux. Le bloc `@media print` ne contient plus que la mécanique de page
(sauts de page, suppression des ombres, `print-color-adjust`).

**Ne jamais redupliquer des valeurs de géométrie dans `@media print`** — c'était
le bug historique (voir plus bas).

---

## Historique du dernier gros chantier : refonte de l'impression

### Le bug de fond (corrigé)

Les styles de fiche existaient **en double**, avec des valeurs **différentes** :
section 11 (aperçu écran) et section 12 (`@media print`).

| | Écran (avant) | Print (avant) |
|---|---|---|
| Image | 34px | 26px |
| Libellé | 9.5px | 8px |
| Case à cocher | 17px | 14px |
| Largeur fiche | 790px (≈209mm) | 194mm |

D'où les deux symptômes : le PDF paraissait « pilé » (tout plus petit), et
**modifier l'aperçu ne changeait rien au PDF**.

Trois bugs aggravants supprimés au passage :
- `padding:8mm 6mm !important` sur `.sheet` en print → écrasait le padding demi-page
- `max-width:none !important` → annulait la largeur A4
- `color:#000 !important` sur titre/pastilles → forçait le noir à l'impression

Ajout de `print-color-adjust:exact` pour que les fonds pastel, le badge de score
et les liserés de rareté s'impriment réellement (sinon le navigateur les supprime).

### État final des deux formats

Le sélecteur **« Format »** (barre d'aperçu) propose :

- **Pleine page A4** (`.sheet.full`) : trame 3 colonnes × 11 rangées = **33 cases**
- **Demi-page A4** (`.sheet.half`) : trame 3 colonnes × 5 rangées = **15 cases**,
  deux fiches par feuille A4

**Les deux formats utilisent exactement la même carte** — c'est volontaire et
demandé. Le style de carte est donc sous le sélecteur commun `.sheet .sheet-item`
(et non `.sheet.half .sheet-item`).

Format de carte (identique partout) :
- carte horizontale, hauteur **17mm**, contour gris `1.5px #cbd3dc`
- **liseré gauche 4px** coloré selon la rareté (`.r1`/`.r2`/`.r3` → vert/orange/rouge)
- image **13mm** à gauche (largeur de colonne fixe 15mm pour aligner les colonnes)
- libellé **3mm** (clamp 2 lignes) + sous-ligne **« +N pt »** en Space Mono 2.4mm,
  colorée selon la rareté
- case à cocher **5.5mm** à droite

Spécifique demi-page :
- en-tête compact « **Roul'trouv** » + badge « Score max — N pts »
- cartouches **Départ / Destination / Date** (fonds pastel), pas de champ Copilote
- bandeau bas juste sous la trame (`margin-top:2.4mm`, pas `auto`) ; l'espace
  restant en bas de la demi-page est laissé blanc pour écrire
- pas de pied de page
- hauteur fixe **138mm** + `@page margin:8mm` → 2×138 = 276mm dans les 281mm
  imprimables. Saut de page via la classe `.break-after` sur les fiches d'index impair.

Spécifique pleine page : en-tête complet, `.sheet-fields` (Copilote/Destination/Date),
bloc score et pied de page conservés.

### Autres correctifs récents

- **Sélecteur « Par page » supprimé** du dock (9/12/16/20). Il était devenu sans
  effet depuis que les cartes ont une taille fixe : le nombre de cases découle de
  la trame. Ses 3 références JS ont été nettoyées (`saveState`, `restoreState`,
  tableau d'écouteurs `change`).
- **Options de `<select>` illisibles en thème nuit** : le rendu natif des listes
  déroulantes **ne gère pas la transparence**. `--field-bg` vaut
  `rgba(255,255,255,.06)` en thème nuit, ce qui se composait sur le blanc système
  → fond blanc + `--ink` quasi blanc = texte invisible jusqu'au survol.
  Corrigé avec des couleurs **opaques** scopées au thème nuit
  (`[data-theme="night"] select`/`select option` → `#141C2D` / `#E7EEFA`),
  plus `color-scheme:dark|light` par thème.

### Qualité des images personnalisées

`fileToPngBase64` utilise `maxDim = 256`. ⚠️ Les images uploadées **avant** ce
changement sont restées en 128px — il faut les ré-uploader pour bénéficier de la
meilleure résolution (d'autant que les images sont désormais affichées à 13mm).

---

## Catalogue externalisé + publication GitHub (le plus récent chantier)

**Constat de départ** : les modifs faites dans l'interface (changer la rareté
d'un élément via sa pastille de points, supprimer un élément du catalogue)
n'étaient sauvegardées que dans le `localStorage` du navigateur — perdues si
on change d'appareil/navigateur, jamais visibles pour un autre visiteur de la
page GitHub Pages.

**Solution** : le catalogue (`DEFAULT_ITEMS`/`DEFAULT_BONUS_ITEMS`, toujours
présents dans le `<script>` comme repli) est maintenant aussi disponible en
externe dans `data/items.json`/`data/bonus.json`. Au démarrage, `loadCatalog()`
tente un `fetch()` des deux fichiers ; en cas d'échec (fichier absent,
`file://`, hors-ligne) elle retombe silencieusement sur les tableaux intégrés
— l'app reste toujours fonctionnelle, seule la publication devient impossible.

Le flux complet, dans l'ordre :
1. `initApp()` (tout en bas du script) appelle `loadCatalog()`, construit
   `items`/`bonusItems` à partir du résultat, **puis seulement** appelle
   `restoreState()` + `render()` + `initGhFromStorage()`. Avant, ce
   `restoreState(); render();` tournait de façon synchrone au chargement du
   script ; il a fallu tout basculer en `async` pour que le catalogue soit prêt
   avant le premier rendu.
2. `RARITY_BY_LABEL` (Map label→rareté) n'est plus une constante calculée sur
   `DEFAULT_ITEMS` mais une variable reconstruite après le fetch, sur la
   **vraie** base (fetchée ou repli). Sert à ne persister en `localStorage` que
   les écarts (`rarityOverrides`) par rapport à cette base, pas tout le
   catalogue.
3. Modifier une rareté (clic sur la pastille `+N`) ou supprimer un élément du
   catalogue (croix ×, visible au survol/focus) met à jour `items` en mémoire
   + `localStorage` **immédiatement**, comme avant — ça marche même sans être
   connecté à GitHub.
4. Si connecté à GitHub (même connexion que pour les icônes, modale ⚙️
   Réglages) et qu'il y a au moins un écart en attente, un bouton
   **« 📤 Publier le catalogue (N) »** apparaît dans le dock. Il n'apparaît
   **jamais** si non connecté ou s'il n'y a rien à publier — c'est voulu (pas
   de bouton qui traîne pour rien).
5. `publishCatalogToGithub()` reconstruit le catalogue complet à partir de
   `items` (en excluant les ajouts personnels `custom`), l'encode en base64
   UTF-8-safe (`utf8ToBase64`, nécessaire pour les accents/emoji — `btoa()`
   seul plante dessus), et l'écrit dans `data/items.json` via
   `putFileToGithub()` (helper mutualisé avec `uploadIcon`, extrait de la
   logique GET-sha-puis-PUT). Au succès : `baseItemsCatalog`/`RARITY_BY_LABEL`
   sont mis à jour sur le nouveau catalogue et `removedDefaultLabels` est vidé
   — donc le compteur d'écarts en attente retombe à 0 et le bouton disparaît.

**Ce qui reste volontairement local, jamais publiable** : les ajouts perso
(section « ➕ Ajouter un élément à toi », `custom:true`), la sélection en
cours pour un trajet donné, le pseudo/destination/objectif. Seul le catalogue
de base (rareté, éléments présents) est publiable.

**Non implémenté** : suppression/republication des défis bonus par défaut
(seuls les défis bonus perso sont supprimables, comme avant). `data/bonus.json`
existe et est chargé, mais rien ne le republie pour l'instant — à faire si le
besoin se présente, sur le même modèle que les items.

---

## Session 2026-08-18 : coches multiples, cadeaux, répartition des défis

Grosse session de gameplay. Cinq chantiers, tous dans `chasse-au-tresor.html`.

### 1. Coches multiples par item (×1 / ×2 / ×3)

Un item peut être à repérer plusieurs fois (ex. « 2 voitures rouges »). Les
points se **cumulent** à chaque coche.

- Champ `mult` (1→3) sur chaque item. Helpers **source unique de vérité** :
  `itemMult(it)` (borne 1..3), `itemPoints(it)` = `POINTS[rarity] * mult`,
  `itemChecks(it)` = `mult` (un ×3 vaut 3 objets à trouver).
  ⚠️ **Toujours passer par ces helpers** pour tout calcul de points/objets
  (score max, totaux de page) — ne jamais réutiliser `POINTS[i.rarity]` seul.
- Éditeur : pastille **`×n`** sur chaque carte (`.mult-chip`), clic = cycle
  1→2→3→1 (comme la pastille de rareté). Placée **sous** le `pts-chip`
  (top:32px/left:7px) pour ne jamais chevaucher la coche de sélection à droite.
  Style ambre (`--accent2`) quand active, pour être lisible dans tous les thèmes
  (le piège : sur thème nuit, un fond sombre sur fond sombre était invisible).
- Persistance : `multOverrides` (items du catalogue, comme `rarityOverrides`) +
  champ `mult` dans les customs.
- Fiche : `boxesHtml(it)` génère autant de cases que le multiplicateur
  (conteneur `.boxes`, `data-multi` si >1) ; suffixe **« · ×n »** dans le libellé
  via `multSuffix(it)`.

### 2. Cadeaux à débloquer par paliers — TROIS modes

Toggle « 🎁 Cadeaux à débloquer » dans le panneau « L'équipage », avec un menu
**Critère** à trois valeurs (valeur interne `giftCriteria`) :

1. **`points`** — seuils manuels de points (ex. `20, 40, 60`). Pagination normale.
2. **`count` (« Objets — paliers simples »)** — seuils manuels de cases cochées,
   n'importe lesquelles (« 20 objets cochés → 🎁 »). Pagination normale.
   **C'est le comportement initial**, restauré après un test terrain : compléter
   un groupe entier (mode paquets) s'est avéré trop dur sur un vrai trajet.
   Seuils manuels → permet des paliers **progressifs** (10, 25, 45, 70…).
3. **`pack` (« Objets — paquets, 1 page par cadeau »)** — l'utilisateur saisit un
   **nombre de cadeaux**, l'appli découpe et **une page = un paquet** (voir §3).

UI (`syncGiftConfig`) : `pack` montre le champ « Nombre de cadeaux » ;
`points`/`count` montrent le champ seuils (label adapté : « Paliers de points »
vs « Paliers d'objets cochés »). Aperçu en direct des paliers (`renderGiftPreview`,
pastilles `.pill`), rafraîchi quand on change la sélection, le nb de cadeaux ou
les défis bonus.

`getGiftConfig(objTotal)` renvoie `{enabled, criteria, thresholds[]}`. En mode
`pack`, les seuils sont calculés par `giftTranches(total, n)` (paquets ~égaux,
**petits d'abord**, le reste aux derniers). Persistance : `giftEnabled`,
`giftCriteria`, `giftThresholds`, `giftCount`.

Fiche : ligne **« 🎁 Cadeaux à débloquer »** (`renderGiftLine`) sur la **dernière
page** seulement (une case à cocher + 🎁 + seuil par palier). Légende « (points) »
ou « (objets trouvés) ».

### 3. Mode paquets : pagination qui suit les paquets

Quand `criteria === 'pack'` (variable `packMode`), la pagination ne suit plus la
trame fixe mais les paquets : **une page (ou demi-page) = un paquet**, avec un
bandeau **« PAQUET N → tout cocher → 🎁 »** (`.pack-banner`) en tête.

- Découpage **par cartes** (« option cartes égales » : pages bien remplies), pas
  par objets — choix assumé vu que la plupart des cartes sont ×1. Les seuils
  affichés (`giftCfg.thresholds`) sont ensuite recalculés sur le **cumul réel des
  coches** de chaque paquet.
- Si un paquet dépasse la capacité d'une page, le nombre de cadeaux est
  **augmenté automatiquement** au minimum viable + **toast** d'info.
- Ce mode est le seul à redécouper la pagination ; `points` et `count` gardent la
  pagination normale.

### 4. Répartition des défis bonus

**Avant** : tous les défis sur la 1re page. **Maintenant** :

- **Mode normal** (`points`/`count` ou pas de cadeaux) : les défis passent sur la
  **DERNIÈRE page** (bilan de fin, avec total + cadeaux) — plus cohérent.
- **Mode paquets** : les défis sont répartis en **round-robin** sur les paquets
  (`bonusByPack`, défi i → paquet `i % nPacks`). Certains paquets peuvent rester
  sans défi si l'utilisateur n'en met pas assez — assumé.
- **Alerte de cohérence** (mode paquets, dans l'aperçu éditeur `.gift-warn`) :
  s'affiche si `nbDéfis % nbPaquets !== 0` (répartition inégale), avec le détail.
  Ex. 4 paquets / 4 défis → pas d'alerte ; 4/6 → alerte ; 4/8 → pas d'alerte.

### 5. Récap objets + correctifs d'affichage

- **Bandeau score** : la légende « 1 pt = fréquent… » est **retirée**. Le bandeau
  affiche « **Objets / N** » par page (via `pageChecks`, coches cumulées) et, sur
  la dernière page, un total « **🏆 Total : objets · points** » (cellule `.tot`).
- **Encart bonus à hauteur adaptative** : `.sheet-bonus[data-count="N"]` → grille
  2 colonnes, hauteur selon `ceil(N/2)` lignes (13/17/22/26mm). Corrige le bug
  « le 8e défi était coupé » (l'encart était figé à 17mm, ~7 défis maxi).
- **Capacité de page proportionnelle au bonus** : `lastPageCapacity` (remplace
  l'ancien `firstPageCapacity`) réserve autant de rangées de cartes que l'encart
  bonus en a besoin (`bonusCardRows`), sur la dernière page. Si trop plein, le
  surplus bascule sur une page de plus. Évite tout débordement.
- **Ligne cadeaux** : `justify-content:flex-start` (au lieu de `space-around`)
  pour éviter un 🎁 orphelin centré tout seul quand ça passe à la ligne.

### Catalogue enrichi (trajet Bordeaux → Chenonceau → Beauval, été)

Ajoutés dans **`data/items.json`** (la source de vérité en ligne — le
`DEFAULT_ITEMS` du HTML n'est qu'un repli, cf. section catalogue externalisé) :
- « Botte de foin ronde » (Nature & animaux, rareté 2)
- « Panneau direction Beauval » (Panneaux & signalisation, rareté 3)

⚠️ Pour les voir **en ligne**, il faut committer/pousser `data/items.json`.
Emojis approximatifs (🌾🔵 / 🟤🐼) — mieux vaut uploader une vraie image via le
crayon ✏️. Rappel : le logo Beauval est une marque déposée, non recréable ;
l'utilisateur met sa propre capture.

## Emplacements clés dans le code

Le CSS est découpé en sections numérotées et commentées (`1. TOKENS COMMUNS`,
`2. THÈMES`, … `12. IMPRESSION`).

- **Géométrie de la fiche** : bloc « GEOMETRIE PARTAGEE ECRAN + IMPRESSION »,
  juste après la section 11, **hors** `@media print`
- **Impression** : section 12 — mécanique de page uniquement
- **Thèmes** : section 2, quatre thèmes `paper` / `retro` / `modern` / `night`
  sur `<html data-theme="...">`
- **Réglages GitHub** : modale ouverte par la roue crantée ⚙️ (bouton flottant
  en haut à droite) ; section JS « D. CONNEXION GITHUB »
- **`buildPreview()`** : génère les deux formats. Capacités : `fullCapacity` = 15
  (demi-page) ou 33 (pleine page) ; `lastPageCapacity` réduit la dernière page
  selon la taille de l'encart bonus. Trois helpers cadeaux : `getGiftConfig`,
  `giftTranches`, `renderGiftLine` ; défis : `renderBonusBlock` + `bonusByPack`.

## Détails techniques utiles

- Rareté → points : `POINTS = {1:1, 2:3, 3:5}`. Classes CSS `r1`/`r2`/`r3`.
- Le visuel d'un item vient de `visual(item)` : image perso (slug) > SVG (`ICONS`)
  > emoji.
- Le titre imprimé diffère selon le mode : « Roul'trouv » (demi-page) vs
  « Chasse au trésor de la route » (pleine page).
- Token GitHub : stocké en `localStorage` **uniquement si** la case « se souvenir »
  est cochée ; bouton « Oublier le token » pour l'effacer.

## Pistes / points ouverts

- Tester l'impression réelle (Ctrl+P) dans les deux formats, avec
  **« Graphiques d'arrière-plan » activé** dans Chrome.
- Le champ « Départ » des cartouches est toujours vide par défaut : pas de source
  de données dans l'éditeur. À ajouter si utile.
- Le badge « Score max » et les libellés de cartouches utilisent `--accent`, qui
  change selon le thème. Avec un thème à accent très clair, le texte blanc du
  badge serait peu lisible à l'impression — envisager de figer ces couleurs pour
  le print.
- Ré-uploader les icônes créées avant le passage à 256px.
- **À tester** cette session : impression avec 8 défis bonus (vérifier qu'ils
  apparaissent tous), 5 seuils de cadeaux (pas d'orphelin), et surtout que la
  dernière page ne déborde pas quand beaucoup de défis + trame pleine en
  demi-page (le calcul `bonusCardRows` est prudent mais empirique).
- Le mapping `bonusCardRows` (rangées de cartes sacrifiées selon le nb de défis)
  est approximatif ; à ajuster si un test réel montre un débordement ou trop de
  vide en demi-page.
- `data/items.json` enrichi (botte de foin, panneau Beauval) — **penser à
  committer/pousser** pour la version en ligne.

## Note d'outillage

- Éditions via le connecteur **Filesystem** : `edit_file` fonctionne de manière
  fiable ; `str_replace` échoue à cause de l'encodage des accents dans le chemin.
- Le connecteur Filesystem **ne permet pas d'exécuter git** — les commits restent
  manuels. Pour que Claude puisse commiter lui-même, lancer **Claude Code** depuis
  le dossier du repo.
- Validation avant écriture : extraire le `<script>` et faire `node --check`,
  et vérifier l'équilibre des accolades du `<style>`.
