# Creative Dev / Webflow — Bibliothèque d'effets

Bibliothèque personnelle d'effets et interactions web haut de gamme, pensés pour Webflow et réutilisables d'un projet à l'autre. Chaque effet est documenté de bout en bout : nom technique, faisabilité, structure HTML, CSS, JS commenté, pièges rencontrés et réglages.

Maintenue par Charles NGBO-ZEZE.

## Stack de référence

Webflow, Slater, GSAP, ScrollTrigger, Lenis, Three.js et WebGL, CMS Webflow, custom code vanilla.

## Effets

| Effet | Dossier | Résumé |
|-------|---------|--------|
| WebGL Deform | [webgl-deform](./webgl-deform) | Distorsion d'une texture par vélocité souris et tactile, avec aberration chromatique. Sur texte et SVG. |
| Sticky Stack | [sticky-stack](./sticky-stack) | Empilement de projets plein écran qui se recouvrent au scroll, alternance gauche droite. |
| Player mobile double bande | [mobile-player](./mobile-player) | Deux bandes CMS synchronisées, titres qui glissent, images en recouvrement, sur cadre figé. |
| Nav Blend auto-inversé | [nav-blend](./nav-blend) | Texte de nav qui s'inverse automatiquement selon le fond via mix-blend-mode. |
| Highlight Sweep | [highlight-sweep](./highlight-sweep) | Balayage de surlignage lettre par lettre, en boucle ou au survol. |
| Navigation over-scroll | [overscroll-nav](./overscroll-nav) | Passage au projet suivant/précédent par over-scroll entre pages CMS, avec pourcentage de progression. |
| Curseurs contextuels | [curseurs-contextuels](./curseurs-contextuels) | Labels Ouvrir / Fermer suivant le pointeur en blend difference, zone cliquable. |
| Marquee CMS imbriqué | [marquee-cms-imbrique](./marquee-cms-imbrique) | Bande horizontale sans couture via Collection List imbriquée, parallax masqué. |
| Grille infinie playground | [infinite-grid-playground](./infinite-grid-playground) | Grille draggable infinie branchée au CMS, épinglée en sticky. |

## Dépendances globales

À charger une seule fois dans le custom code footer du site Webflow, avant le loader Slater :

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/ScrollTrigger.min.js"></script>
<script src="https://unpkg.com/lenis@1.1.13/dist/lenis.min.js"></script>
```

GSAP et Lenis sont des globals, disponibles partout sans import. Three.js et html2canvas se chargent en modules ES via esm.sh, directement en haut du fichier Slater (voir la fiche webgl-deform).

## Mode d'emploi

Chaque dossier contient un README autonome. Le workflow général :

1. Poser la structure HTML et les classes dans Webflow, en imposant une combo dédiée dès qu'une classe utilitaire est partagée.
2. Coller le CSS et le JS dans Slater.
3. Republier le site. Slater lit le DOM du site publié, pas le Designer.
4. Tester sur desktop, tablette et mobile.

## Rappels transverses

- Grid qui déborde ou chevauche, utiliser minmax(0, 1fr), jamais 1fr seul.
- 100vw inclut la scrollbar, préférer 100% pour éviter le décalage horizontal.
- position sticky mort, un ancêtre a overflow hidden ou un transform.
- mix-blend-mode piégé, l'élément est enfermé dans un parent fixed ou transformé.
- Classe utilitaire partagée, toujours une combo pour cibler proprement.
- Imports ES dans le navigateur, esm.sh, jamais de specifier nu. Une seule erreur d'import ou de syntaxe tue tout le module.
- WebGL ne travaille que sur des textures, rasteriser le texte et sérialiser le SVG.

## Ajouter un effet

Convention : un dossier par effet, nommé en kebab-case, contenant un `README.md` qui suit la structure standard.

```
mon-effet/
└── README.md   (nom technique, faisabilité, HTML, CSS, JS, pièges, réglages)
```

Optionnel, une démo isolée par effet peut vivre dans un sous-dossier `demo/` avec son propre index.html, styles.css et script.js.

Quand un effet évolue, on remplace son README, on n'empile pas de version. Le déclencheur de production d'une fiche, côté assistant, est la phrase "sauvegarde de l'effet".

## Licence

Usage personnel et projets clients. Réutilisation libre pour Jules Studio.
