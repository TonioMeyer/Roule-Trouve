# Reprise de session — Roul'trouv (chasse-au-tresor.html)

Dernière mise à jour : 2026-08-14

## Contexte du projet

Application web autonome (un seul fichier `chasse-au-tresor.html`) pour créer et
imprimer une fiche de « chasse au trésor de la route » (jeu d'observation pour
les trajets en voiture). L'utilisateur choisit des éléments à repérer par la
fenêtre (véhicules, panneaux, animaux, etc.), chacun ayant une rareté
(1/3/5 pts), puis génère une fiche imprimable.

Le dépôt contient aussi :
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
- **`buildPreview()`** : génère les deux formats ; `perPage` vaut 15 (demi-page)
  ou 33 (pleine page)

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

## Note d'outillage

- Éditions via le connecteur **Filesystem** : `edit_file` fonctionne de manière
  fiable ; `str_replace` échoue à cause de l'encodage des accents dans le chemin.
- Le connecteur Filesystem **ne permet pas d'exécuter git** — les commits restent
  manuels. Pour que Claude puisse commiter lui-même, lancer **Claude Code** depuis
  le dossier du repo.
- Validation avant écriture : extraire le `<script>` et faire `node --check`,
  et vérifier l'équilibre des accolades du `<style>`.
