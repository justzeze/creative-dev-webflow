# 04 — Nav Blend auto-inversé

## Nom technique

Auto-inverting text via mix-blend-mode difference. Un texte qui s'inverse automatiquement selon le fond derrière lui, noir sur les zones claires, blanc sur les zones foncées, en continu au scroll, sans JS. Effet type logo T-M de Monavon.

## Faisabilité Webflow

Natif. Webflow gère mix-blend-mode via la section Effects du panneau Style, sous le nom Blending. Aucun code requis pour le cas simple.

## Principe

Le mode Difference calcule la valeur absolue de (fond moins texte), pixel par pixel.

- Texte blanc sur fond clair : donne noir.
- Texte blanc sur fond foncé : donne blanc.

Donc on écrit le texte en blanc, et il apparaît noir sur les zones claires, blanc sur les zones foncées. Contre-intuitif mais c'est la clé : la couleur source doit être blanche.

Variante Exclusion pour un rendu plus doux tirant vers le gris.

## Cas simple, toute la nav s'inverse

- Sélectionner l'élément, panneau Style, Effects, Blending, Difference.
- Couleur du texte en blanc.

Le point critique, cause d'échec numéro un : le contexte d'empilement. mix-blend-mode ne se mélange qu'avec ce qui est peint dans le même contexte, derrière l'élément. Si un ancêtre crée un contexte isolé, le blend ne voit plus les sections de fond.

Un ancêtre casse le blend s'il a : `position: fixed`, `isolation: isolate`, `opacity` < 1, un `transform`, `filter` ou `will-change`.

Cas typique nav fixed : le blend doit être posé sur l'élément fixed lui-même, pas sur un enfant. Un enfant d'un élément fixed est enfermé dans le contexte du fixed, il ne se mélange qu'avec le fond de la nav (transparent), jamais avec les sections derrière.

```css
.nav-component {
  position: fixed;
  mix-blend-mode: difference;  /* sur l'élément fixed, pas sur les liens enfants */
}
.nav-link { color: #fff; }
```

## Cas avancé, inverser seulement certains liens

Si logo et bouton doivent rester intacts et que seuls les liens s'inversent, on ne peut pas blender un enfant enfermé dans le fixed. Il faut sortir les liens dans leur propre couche fixed qui porte le blend.

```
page-wrapper (sans transform)
├── nav-component        (fixed, SANS blend)
│   ├── logo             ← inchangé
│   └── let-talk-wrapper ← inchangé
└── nav-links-layer      (fixed, mix-blend-mode: difference, texte blanc)
    └── nav-link ×N      ← seuls ceux-là s'inversent
```

Coût : recaler horizontalement la couche des liens, puisqu'ils quittent le flex de la nav.

## Alternative JS, basculement discret

Si on veut un changement net section par section plutôt qu'un mélange continu, IntersectionObserver sur les sections foncées, toggle d'une classe sur les liens. Moins fin que le blend (bascule d'un bloc, pas au pixel), mais contrôlable. Le blend reste supérieur pour un fond au contraste morcelé.

```javascript
const links = document.querySelectorAll(".nav-link");
const dark = document.querySelectorAll(".section.is-dark");
const io = new IntersectionObserver((entries) => {
  entries.forEach((e) => links.forEach((l) => l.classList.toggle("is-light", e.isIntersecting)));
}, { rootMargin: "-50% 0px -50% 0px" });
dark.forEach((s) => io.observe(s));
```

## Pièges rencontrés et résolutions

- Blend appliqué sur les liens enfants d'une nav fixed : ne se mélange qu'avec la nav, jamais avec la page. Solution : blend sur l'élément fixed, ou couche séparée si on ne veut que certains liens.
- Texte noir au lieu de blanc : avec Difference, la couleur source doit être blanche pour donner l'inversion attendue.
- Un smooth scroll qui pose un transform sur un wrapper global recrée un contexte isolé et casse le blend. Garder la nav hors de tout ancêtre transformé.

## Réglages

- Difference pour un noir/blanc franc, Exclusion pour un rendu gris plus doux.
- Les fonds derrière doivent avoir une couleur ou image réellement peinte, un fond transparent n'a rien à mélanger.
