# 09 — Grille infinie draggable (playground) branchée au CMS

## Nom technique

Grille de cartes infinie dans les deux axes, navigable à la molette et au glisser, avec parallax de colonnes, réduction d'échelle pendant le mouvement et enroulement modulo. Base : composant Osmo Supply "Infinite Cards Grid", adapté ici pour être alimenté par une Collection List Webflow au lieu d'items statiques.

## Faisabilité Webflow

Structure Webflow classique plus GSAP et son plugin Observer. Le script clone les items sources pour remplir la grille : que ces items viennent du CMS ou du statique lui est indifférent, Webflow rend le HTML côté serveur avant l'exécution du JS.

**Arbitrage CMS ou statique** : CMS justifié ici, car le contenu se répète et grandit en continu, et l'ajout doit pouvoir se faire depuis l'Editor sans ouvrir le Designer. Le critère général reste le même : ce qui se répète va en collection, ce qui est unique et figé reste statique.

## Structure CMS

Collection **Playground** : `name`, `media-type` (Option Image/Video), `image`, `video-url`, `poster`, `ordre`.

Le champ `poster` sert au `preload="none"` des vidéos, décisif pour les performances (voir vigilance).

## Correspondance Osmo vers Webflow CMS

C'est le point délicat, une Collection List générant trois niveaux de DOM.

| Élément Osmo | Élément Webflow | Classe | Attribut |
|---|---|---|---|
| wrapper | Section | `infinite-grid` | `data-infinite-grid-init` |
| collection | **Collection List Wrapper** | `infinite-grid__collection` | `data-infinite-grid-collection` |
| list | **Collection List** | `infinite-grid__list` | `data-infinite-grid-list` |
| item | **Collection Item** | `infinite-grid__item` | `data-infinite-grid-item` |
| card | div interne | `infinite-grid__card` | `data-infinite-grid-card` |
| image | Image + Code Embed | `infinite-grid__card-img` | — |

Toutes les valeurs d'attributs restent **vides** : le script utilise des sélecteurs de présence, il ne lit jamais le contenu. Seul `data-infinite-grid-status` est écrit par le script (`loading`, `idle`, `dragging`, `scrolling`).

## Dépendances

Dans le footer du site, avant le loader Slater :

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.15/dist/Observer.min.js"></script>
```

Sans ce plugin, `gsap.registerPlugin(Observer)` échoue et tue tout le fichier Slater, y compris les autres effets.

## CSS

Styles de structure, portés par les classes Webflow :

```css
.infinite-grid { position: relative; overflow: clip; width: 100%; height: 100svh; touch-action: none;
                 position: sticky; top: 0; }
.infinite-grid__collection { position: absolute; width: 100%; height: 100%; will-change: transform; }
.infinite-grid__list { position: absolute; left: 0; top: 0; width: 100%; height: 100%; }
.infinite-grid__item {
  position: absolute; left: 0; top: 0;
  display: flex; width: 13em; padding: 2.25em;
  justify-content: center; align-items: center;
  aspect-ratio: 1/1;
  font-size: clamp(1.5em, 2vw, 3em);
  will-change: transform; backface-visibility: hidden;
}
.infinite-grid__card { position: relative; width: 100%; height: 100%; border-radius: 0.125em;
                       user-select: none; will-change: transform; backface-visibility: hidden; }
.infinite-grid__card-img { position: absolute; width: 100%; height: 100%;
                           pointer-events: none; object-fit: cover; border-radius: inherit; }

/* Conteneur de defilement, donne la duree avant le footer */
.playground-scroll { position: relative; width: 100%; height: 400vh; }
```

États et confort, à mettre dans le custom code head de la page :

```html
<style>
.infinite-grid[data-infinite-grid-status] { cursor: grab; transition: opacity 0.5s ease; opacity: 1; }
.infinite-grid[data-infinite-grid-status="loading"] { opacity: 0; }
.infinite-grid[data-infinite-grid-status="dragging"] { cursor: grabbing; }
:is(.wf-design-mode, .wf-editor) .infinite-grid[data-infinite-grid-status] { opacity: 1; }
:is(.wf-design-mode, .wf-editor) .infinite-grid .infinite-grid__list {
  display: flex; flex-wrap: wrap; justify-content: center; align-items: center; }
:is(.wf-design-mode, .wf-editor) .infinite-grid .infinite-grid__item {
  position: relative; top: unset; left: unset; }
</style>
```

Les blocs `wf-design-mode` rendent la grille lisible dans le Designer, sinon on ne voit qu'un empilement superposé.

## JS

Le script d'Osmo est repris intégralement. **Une seule modification**, l'initialisation :

```javascript
// Osmo termine par :
// document.addEventListener('DOMContentLoaded', () => { initInfiniteCardsGrid(); });
// Slater injecte le script APRES cet evenement, le listener ne se declencherait jamais.
if (document.readyState === "loading") {
  document.addEventListener("DOMContentLoaded", initInfiniteCardsGrid);
} else {
  initInfiniteCardsGrid();
}
```

Réglages exposés en tête de fonction :

```javascript
const wheelSpeed = 0.6;              // vitesse molette et trackpad
const dragSpeed = 1.2;               // vitesse au glisser
const gridOverscan = 1;              // rangees et colonnes generees hors ecran
const startOffsetY = 0.33;           // decalage vertical entre colonnes
const positionLerp = 0.05;           // lissage du deplacement
const xToYInfluence = 0.2;           // influence horizontale sur le vertical
const columnSpeedPattern = [1, 1, 0.9]; // parallax par colonne
const minCardScale = 0.5;            // echelle minimale pendant le mouvement
const scaleLerp = 0.02;              // lissage de l'echelle
```

## Réglages visuels

| Ce que tu veux changer | Où |
|---|---|
| Taille des cartes | `infinite-grid__item`, `width` |
| Espace entre cartes | `infinite-grid__item`, `padding` (il n'y a pas de gap, les cartes sont en absolu) |
| Format des cartes | `infinite-grid__item`, `aspect-ratio` |
| Échelle globale | `infinite-grid__item`, `font-size` — tout étant en `em`, ce seul levier redimensionne l'ensemble |
| Durée avant le footer | `playground-scroll`, `height` |

## Pièges rencontrés et résolutions

- **`DOMContentLoaded` déjà passé** : le piège Slater récurrent. Page vide, aucune erreur en console. Voir la correction ci-dessus.
- **Avertissement Slater "There is a load event in the file"** : analyse statique, sans conséquence si le code teste `document.readyState` avant. Ne pas supprimer les gardes.
- **Augmenter la hauteur de la section pour allonger l'expérience** : erreur. `rows` est calculé sur `wrapper.clientHeight`, donc 500vh multiplie par cinq le nombre de cartes clonées, soit plus de deux cents éléments animés à chaque frame, dont on ne voit qu'un cinquième. La grille est déjà infinie, sa hauteur n'est que la fenêtre d'observation.
- **Bonne solution pour la durée** : envelopper la section dans un conteneur `playground-scroll` haut (400vh) et passer la grille en `position: sticky; top: 0; height: 100svh`. La grille reste épinglée pendant plusieurs écrans, le footer n'arrive qu'ensuite, et le nombre de cartes ne change pas.
- **Sticky qui ne prend pas** : un ancêtre en `overflow: hidden` tue le sticky. Vérifier `main-wrapper` et `page-wrapper`.
- **Cartes de tailles différentes impossibles** : le script mesure `originalItems[0]` seulement et applique cette dimension à toute la grille. Pour un effet mosaïque, garder l'item à taille fixe et faire varier le contenu **à l'intérieur** de la carte.
- **`preventDefault: true` sur l'Observer** : la molette pilote la grille et non la page. Tout contenu placé sous la grille peut devenir inatteignable. À vérifier au cas par cas.
- **Nav recouverte** : `nav-component` n'avait aucun `z-index` et passait sous les curseurs (100) et overlays (50). Lui donner `z-index: 1000`.

## Points de vigilance

- **Vidéos** : le script clone chaque item 6 à 12 fois. Une vidéo devient donc autant de balises jouant simultanément. Garder la majorité en images, deux ou trois vidéos au maximum, avec `preload="none"` et un `poster`.
- **Format vidéo** : `.mov` n'est lu ni par Chrome ni par Firefox. Convertir en MP4 H.264.
- **Lien de page courante** : ne jamais mettre `#`. Webflow ajoute automatiquement `w--current` au lien de la page affichée, styler cet état et, si souhaité, y poser `pointer-events: none`. Ajouter `aria-current="page"` en JS pour les lecteurs d'écran.
