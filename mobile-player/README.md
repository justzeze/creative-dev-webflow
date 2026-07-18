# 03 — Player mobile double bande synchronisée

## Nom technique

Dual-track scroll player. Un cadre figé (pinné) contient deux cases, titre en haut, image en bas. Deux bandes distinctes défilent à l'intérieur, pilotées par une seule et même progression de scroll, donc toujours synchronisées. La bande titres glisse, la bande images fait un recouvrement en calque (l'image courante reste, la suivante monte par-dessus avec parallax). Effet type player Monavon sur mobile.

## Faisabilité Webflow

Structure custom plus GSAP. Le lien titre-image est garanti par le CMS : deux Collection Lists branchées sur la même collection, même ordre, même nombre d'items. On n'affiche pas les deux ensemble, on les sépare en deux pistes, ce qui est impossible avec une liste unique où chaque item soude titre et image (ils défileraient collés, les images arriveraient dans la case des titres).

Scopé mobile et tablette uniquement. Le sticky stack desktop (fiche 02) reste actif sur grand écran, ce player le remplace en dessous de 992px.

## HTML / classes requises

Bloc posé en frère de la section projets, masqué sur desktop :

```
mobile-player            (wrap, hauteur fixée par JS = N × 100vh)
└── mp-frame             (sticky top 0, 100vh, overflow hidden, grid 45vh / 55vh)
    ├── mp-titles        (case haut, overflow hidden, fond offwhite)
    │   └── Collection List (Projects)   [classe mp-titles-track sur la liste]
    │       └── mp-title-item            (height 45vh, flex)
    │           ├── Text lié au champ Name
    │           └── Text lié au champ Numero
    └── mp-images        (case bas, overflow hidden)
        └── Collection List (Projects)   [classe mp-images-track sur la liste]
            └── mp-image-item            (position absolute, inset 0)
                └── Image liée à Thumbnail (cover)
```

Règle de synchro : chaque item de titre égale la hauteur de sa case (45vh), chaque item d'image remplit sa case (55vh). Une seule fiche s'affiche par case.

## CSS

```css
.mobile-player { position: relative; width: 100%; display: none; }  /* caché desktop */

.mp-frame {
  position: sticky; top: 0;
  width: 100%; height: 100vh; overflow: hidden;
  display: grid; grid-template-columns: 1fr; grid-template-rows: 45vh 55vh;
}

.mp-titles { position: relative; overflow: hidden; background-color: #FFFEFA; }
.mp-images { position: relative; overflow: hidden; }

.mp-title-item { height: 45vh; }        /* égale la case titre */
.mp-image-item {                         /* empilées au même endroit */
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%;
}
```

Bascule desktop / mobile, au breakpoint tablette :

```css
/* tablette et en dessous */
.mobile-player { display: block; }   /* on montre le player */
.project-section { display: none; }  /* on cache l'empilement desktop */
```

## JS

Placé à la suite du code existant, dans un matchMedia pour ne toucher que mobile et tablette.

```javascript
const mmPlayer = gsap.matchMedia();
mmPlayer.add("(max-width: 991px)", () => {
  const wrap = document.querySelector(".mobile-player");
  const titles = document.querySelector(".mp-titles-track");
  const titlesCase = document.querySelector(".mp-titles");
  const imageItems = gsap.utils.toArray(".mp-image-item");
  if (!wrap || !titles || !imageItems.length) return;

  const count = imageItems.length;
  if (count < 2) return;

  wrap.style.height = count * 100 + "vh"; // distance de scroll
  const tH = titlesCase.offsetHeight;

  // les images récentes passent au-dessus des précédentes
  imageItems.forEach((img, i) => { img.style.zIndex = i; });

  const st = ScrollTrigger.create({
    trigger: wrap, start: "top top", end: "bottom bottom", scrub: true,
    onUpdate: (self) => {
      const p = self.progress * (count - 1); // 0 → dernier projet

      // TITRES : glissent dans leur case
      gsap.set(titles, { y: -p * tH });

      // IMAGES : la courante reste fixe, la suivante monte par-dessus + parallax
      imageItems.forEach((img, i) => {
        const cover = Math.min(Math.max(i - p, 0), 1);    // 1 = sous la case, 0 = en place
        const parallax = Math.min(Math.max(p - i, 0), 1); // 0→1 pendant qu'elle se fait recouvrir
        gsap.set(img, { yPercent: cover * 100 - parallax * 12 });
      });
    },
  });

  ScrollTrigger.refresh();
  return () => { wrap.style.height = ""; st.kill(); };
});
```

## Pièges rencontrés et résolutions

- Une seule liste ne peut pas faire l'effet : titre et image soudés dans le même item défilent collés, l'image débarque dans la case du titre. Il faut deux Collection Lists sur la même source CMS.
- Case titre écrasée à zéro sur une seule rangée définie : quand on empile en une colonne, il faut deux rangées explicites, sinon le second bloc va dans une rangée implicite auto et écrase le premier.
- Images qui défilent au lieu de se recouvrir : passer `mp-image-item` en `position: absolute`, empilées au même endroit, et animer chaque item individuellement (cover + parallax) plutôt que translater tout le ruban.
- Snap desktop qui parasite le mobile : gater le snap avec `if (window.innerWidth <= 991) return;`.

## Réglages

- Proportion des cases : les deux valeurs `45vh 55vh` du grid, à garder égales aux hauteurs d'item.
- Intensité du parallax : le `* 12` dans le onUpdate. Monter à 20 pour plus marqué, descendre à 8 si un liseré de fond apparaît en bas pendant la transition.
- Vitesse : un projet par 100vh via `count * 100`.

## Point de vigilance

Le smooth Lenis lisse la molette mais pas le tactile par défaut. Pour un rendu liquide au doigt, ajouter à la config Lenis : `syncTouch: true, syncTouchLerp: 0.08, touchMultiplier: 1.2`. Tester sur vrai téléphone, le tactile lissé peut sembler moins immédiat sur appareils faibles. Compromis à assumer.
