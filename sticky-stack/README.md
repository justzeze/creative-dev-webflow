# 02 — Sticky Stack projets

## Nom technique

Sticky stacking, ou empilement collant. Chaque projet fait 100vh et se fige en haut de l'écran via `position: sticky; top: 0`. Le projet suivant, plus bas dans le flux, remonte au scroll et passe par-dessus le précédent qui reste figé dessous. À la fin du dernier projet, le sticky se relâche et le footer apparaît. Effet type Thomas Monavon sur desktop.

## Faisabilité Webflow

Natif à 100% côté CSS, sur une Collection List CMS. Le sticky se pose sur le Collection Item. GSAP et Lenis viennent seulement lisser le scroll, caler le snap et ajouter du parallax.

## HTML / classes requises

Structure CMS standard :

```
project-section            (min-height 100vh)
└── project-collection-list-wrapper   (height auto)
    └── project-collection-list        (height auto)
        └── project-collection-item    (sticky, 100vh, fond opaque)  [×N via CMS]
            └── project-grid            (grid 2 colonnes)
                ├── side-content        (le texte)
                └── side-thumbnail      (l'image)
```

Point essentiel : les items doivent être opaques pour se recouvrir proprement. Fond posé sur `project-collection-item`, à la couleur exacte du contenu, sinon l'image du projet précédent bave à travers.

## CSS

Sur `project-collection-item` :

```css
.project-collection-item {
  position: sticky;
  top: 0;
  height: 100vh;              /* hauteur explicite par projet, pas 100% */
  background-color: #FFFEFA;  /* opaque, couleur exacte du contenu */
  display: flex;
  align-items: center;
  justify-content: center;
}
```

Sur les conteneurs au-dessus, pour que la section grandisse au lieu de bloquer le footer :

```css
.project-section { min-height: 100vh; }     /* pas height: 100vh figé */
.project-collection-list-wrapper { height: auto; }
.project-collection-list { height: auto; }
```

Sur le grid, colonnes stables anti-débordement :

```css
.project-grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);  /* jamais 1fr seul */
  grid-template-rows: minmax(0, 1fr);
  width: 100%;
  height: 100%;
}
.side-content, .side-thumbnail { min-width: 0; min-height: 0; }
.side-content { overflow: hidden; }
```

Alternance gauche/droite, desktop uniquement, via order sur les enfants du grid :

```css
@media (min-width: 992px) {
  .project-collection-item:nth-child(even) .side-content { order: 2; }
  .project-collection-item:nth-child(even) .side-thumbnail { order: 1; }
}
```

## JS (Lenis + snap + parallax)

```javascript
gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis({ lerp: 0.08, wheelMultiplier: 0.9, smoothWheel: true });
lenis.on("scroll", ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);

const items = document.querySelectorAll(".project-collection-item");
let snapTimeout, isSnapping = false;

lenis.on("scroll", ({ scroll }) => {
  if (isSnapping || !items.length) return;
  if (window.innerWidth <= 991) return; // pas de snap hors desktop
  clearTimeout(snapTimeout);
  snapTimeout = setTimeout(() => {
    const h = window.innerHeight;
    const maxSnap = (items.length - 1) * h;
    const nearest = Math.round(scroll / h) * h;
    if (nearest > maxSnap) return; // libre vers le footer après le dernier projet
    if (Math.abs(scroll - nearest) > 2) {
      isSnapping = true;
      lenis.scrollTo(nearest, {
        duration: 0.5,
        easing: (t) => 1 - Math.pow(1 - t, 3),
        onComplete: () => { isSnapping = false; },
      });
    }
  }, 120);
});

// parallax sur les images, recalé pour le sticky (fin à "top top")
items.forEach((item) => {
  const img = item.querySelector(".img-block");
  if (!img) return;
  gsap.fromTo(img,
    { yPercent: -8, scale: 1.2 },
    { yPercent: 8, scale: 1.2, ease: "none",
      scrollTrigger: { trigger: item, start: "top bottom", end: "top top", scrub: true } }
  );
});

ScrollTrigger.refresh();
```

## Pièges rencontrés et résolutions

- Footer qui chevauche les projets : la section était `height: 100vh` figée alors que les items débordaient. La section ne mesurait qu'un écran, le footer se plaçait par-dessus le reste. Solution : `min-height: 100vh` sur la section, `height: auto` sur wrapper et liste, `height: 100vh` explicite sur l'item.
- Grid qui chevauche au resize : `1fr` a un minimum implicite égal au contenu, un gros titre pousse la colonne au-delà de 50%. Solution : `minmax(0, 1fr)`.
- Décalage horizontal au resize : `width: 100vw` inclut la scrollbar. Solution : `width: 100%`.
- Image qui bave à travers la partie texte pendant le recouvrement : items transparents. Solution : fond opaque `#FFFEFA` sur l'item.
- Parallax qui dérive : avec le sticky, la borne de fin doit être `top top`, pas `bottom top`, sinon l'image continue d'animer une fois le projet figé.

## Réglages

- Vitesse du snap : `duration: 0.5`.
- Amplitude du parallax : `yPercent` et `scale`.
- Ratio des colonnes : passer `minmax(0, 1fr) minmax(0, 1fr)` en `minmax(0, 1fr) minmax(0, 1.2fr)` pour déséquilibrer.

## Comportement mobile

Sur tablette et mobile, passer le grid en une colonne, contenu en haut, image en bas, via placement explicite pour neutraliser l'alternance order :

```css
/* au breakpoint tablette */
.project-grid { grid-template-columns: 1fr; grid-template-rows: minmax(0, 1fr) minmax(0, 1fr); }
.side-content { grid-row: 1; grid-column: 1; }
.side-thumbnail { grid-row: 2; grid-column: 1; }
```

Le placement grid explicite prime sur order, donc l'alternance desktop ne perturbe pas l'empilement vertical.
