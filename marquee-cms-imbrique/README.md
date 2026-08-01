# 08 — Marquee CMS imbriqué avec parallax masqué

## Nom technique

Bande horizontale à défilement continu et sans couture, alimentée par une Collection List **imbriquée**, chaque tuile masquant un média qui dérive à l'intérieur selon sa position dans le cadre et selon le scroll de la page.

## Faisabilité Webflow

Entièrement natif côté données. Webflow autorise une Collection List dans un Collection Item **à une seule condition** : qu'elle soit branchée sur un champ **Multi-Reference** de l'item courant. Limite : 5 éléments maximum, sans tri ni filtre. Pour un marquee ce n'est pas gênant, le script duplique le contenu pour boucler.

## Structure CMS

Deux collections :

- **Projets** : reçoit un champ `index-media` de type **Multi-Reference** vers la collection médias. C'est une sélection éditoriale, indépendante de la galerie complète de la page projet.
- **Médias** : `media-type` (Option Image/Video), `image`, `video-url`, `order`.

## HTML / classes

```
index-projet-item                  ← Collection Item du projet
  index-projet-content-grid
    index-projet-title             ← numero, description, lien
  index-projet-media-wrapper       ← le cadre, overflow hidden, position relative
    index-track                    ← Collection List imbriquee, source Index Media, en flex
      index-media                  ← Collection Item, la tuile, overflow hidden, ratio 16/10
        Image                      ← index-media-inner, conditionnelle Media Type = Image
        Code Embed                 ← index-media-inner, conditionnelle Media Type = Video
```

Balise du Code Embed, `src` via + Add Field sur Video URL :

```html
<video autoplay loop muted playsinline preload="metadata"
       style="width:100%;height:100%;object-fit:cover;display:block;">
  <source src="[video-url]" type="video/mp4">
</video>
```

## CSS

```css
.index-projet-media-wrapper { position: relative; width: 100%; overflow: hidden; }
.index-track { display: flex; flex-wrap: nowrap; column-gap: 1rem; will-change: transform; }
.index-media {
  position: relative; flex-shrink: 0;
  width: 24vw; min-width: 240px;
  aspect-ratio: 16 / 10;
  overflow: hidden; background-color: #000;
}
.index-media-inner {
  position: absolute; inset: 0;
  width: 100%; height: 100%;
  object-fit: cover; will-change: transform;
}
```

## JS

```javascript
/* ===== Index : marquee continu avec parallax masque ===== */
(() => {
  const frames = document.querySelectorAll(".index-projet-media-wrapper");
  if (!frames.length) return;

  const SPEED      = 0.5;   // vitesse du defilement, px par frame
  const PARALLAX_X = 20;    // parallax horizontal, dans le sens du marquee
  const PARALLAX_Y = 8;     // derive verticale au scroll, en %
  const SCALE      = 1.15;  // marge du media dans son masque

  frames.forEach((frame) => {
    const track = frame.querySelector(".index-track");
    if (!track || !track.children.length) return;

    // on duplique jusqu'a couvrir deux fois le cadre
    const originals = [...track.children];
    let guard = 0;
    while (track.scrollWidth < frame.clientWidth * 2 && guard < 8) {
      originals.forEach((el) => track.appendChild(el.cloneNode(true)));
      guard++;
    }
    // puis on double l'ensemble : c'est ce qui rend la boucle sans couture
    const cs = getComputedStyle(track);
    const gap = parseFloat(cs.columnGap || cs.gap) || 0;
    const setWidth = track.scrollWidth + gap;
    [...track.children].forEach((el) => track.appendChild(el.cloneNode(true)));

    let tiles = [];
    const cache = () => {
      tiles = [...track.children].map((el) => ({
        left: el.offsetLeft,
        w: el.offsetWidth,
        inner: el.querySelector(".index-media-inner"),
      })).filter((t) => t.inner);
    };
    cache();
    window.addEventListener("resize", cache, { passive: true });

    let x = 0;

    function tick() {
      x += SPEED;
      if (x >= setWidth) x -= setWidth;   // la boucle
      track.style.transform = `translate3d(${-x}px,0,0)`;

      const r = frame.getBoundingClientRect();
      const vh = window.innerHeight;
      const py = (((vh - r.top) / (vh + r.height)) - 0.5) * 2 * PARALLAX_Y;
      const fw = frame.clientWidth;

      for (const t of tiles) {
        const pos = t.left - x;
        if (pos < -t.w || pos > fw) continue;          // hors cadre, on saute
        const c = (pos + t.w / 2 - fw / 2) / fw;       // ecart au centre du cadre
        t.inner.style.transform =
          `translate3d(${-c * PARALLAX_X}px, ${py}%, 0) scale(${SCALE})`;
      }
      requestAnimationFrame(tick);
    }
    requestAnimationFrame(tick);
  });
})();
```

## Pièges rencontrés et résolutions

- **Croire qu'on ne peut pas imbriquer une Collection List** : c'est possible via un champ Multi-Reference, et seulement ainsi. L'API refuse de créer cette liste imbriquée, il faut passer par le Designer.
- **Marquee piloté par la position du curseur** : première version où la souris déplaçait directement la bande. Rendu brusque et pas de défilement au repos. La bonne approche est de faire défiler en continu et, si l'on veut une réaction, de moduler la **vitesse** et non la position. Ici la réaction au curseur a été retirée volontairement, le mouvement constant suffit.
- **Boucle visible** : dupliquer une fois ne suffit pas. Il faut remplir deux fois le cadre **puis** doubler l'ensemble, et faire revenir la position tous les `setWidth`.
- **Le `gap` fausse la largeur** : `scrollWidth` n'inclut pas le dernier écart, d'où le `+ gap` dans `setWidth`. Sans ça, un saut apparaît à chaque boucle.
- **Conflit de transform** : le marquee écrit dans `.index-track`, le parallax dans `.index-media-inner`. Deux éléments distincts, aucun écrasement. Ne jamais animer les deux sur le même noeud.
- **Ordre des blocs si le parallax est séparé** : les clones doivent exister avant que le parallax ne s'attache, sinon seules les tuiles d'origine bougent. Ici tout est dans une seule boucle, le problème ne se pose pas.
- **Deform impossible sur cette bande** : html2canvas fige une capture et ne sait pas cloner une vidéo (`Unable to clone video as it is tainted`). Un bandeau en mouvement avec des vidéos est incompatible avec le deform de la fiche 01. Voies alternatives : filtre SVG `feTurbulence` sur le DOM vivant, ou marquee entièrement WebGL. Écarté comme non pertinent au regard de l'effort.

## Réglages

| Constante | Effet |
|---|---|
| `SPEED` | 0.3 très lent, 1 plus vif. Valeur négative pour inverser le sens |
| `PARALLAX_X` | profondeur horizontale en pixels |
| `PARALLAX_Y` | dérive verticale au scroll |
| `SCALE` | à monter si `PARALLAX_X` dépasse 30, sinon un bord vide apparaît |

## Points de vigilance

- Les vidéos sont dupliquées avec les tuiles. Peu de vidéos, ou `preload="none"` avec un poster.
- Sur mobile la bande défile mais ne réagit pas, il n'y a pas de survol. Pour un balayage au doigt, passer le cadre en `overflow-x: auto` sous 767px et désactiver le script à cette taille.
