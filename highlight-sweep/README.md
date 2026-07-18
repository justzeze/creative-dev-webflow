# 05 — Highlight Sweep

## Nom technique

Balayage de surlignage lettre par lettre. Un fond coloré (noir, texte inversé) passe successivement sur chaque lettre du texte, du début à la fin, comme un marqueur qui glisse. En boucle continue pour un élément d'appel, au survol pour les liens. Vraies lettres conservées, aucun caractère parasite (à distinguer du text scramble, qui lui remplace les lettres par des glyphes aléatoires).

## Faisabilité Webflow

Custom code, vanilla JS, aucune dépendance. Le texte est découpé en spans par lettre, un balayage allume la classe de surlignage sur une fenêtre glissante.

## HTML / classes requises

- Cible spécifique via classe dédiée. La cible en boucle a besoin d'une combo dédiée si sa classe est générique. Exemple : élément scroll to explore, classe utilitaire `text-size-regular` partagée, on ajoute une combo `scroll-explore`.
- Liens : `nav-link`.
- Bouton : `let-talk-wrapper` avec texte intérieur ciblé par querySelector scopé.

## CSS

```css
.hl-char.is-on {
  background-color: #000;
  color: #FFFEFA;
}

/* un bouton déjà noir : on inverse pour que le surlignage se voie */
.let-talk-wrapper .hl-char.is-on {
  background-color: #FFFEFA;
  color: #000;
}
```

## JS

```javascript
function splitChars(el) {
  const text = el.textContent;
  el.textContent = "";
  const chars = [];
  [...text].forEach((ch) => {
    const span = document.createElement("span");
    span.textContent = ch;
    span.className = "hl-char";
    span.dataset.space = ch === " " ? "1" : "0";
    el.appendChild(span);
    chars.push(span);
  });
  return chars;
}

function sweep(chars, { speed = 90, band = 1 } = {}) {
  return new Promise((resolve) => {
    let i = 0;
    const end = chars.length + band;
    const step = () => {
      chars.forEach((c, idx) => {
        const on = idx <= i && idx > i - band && c.dataset.space === "0";
        c.classList.toggle("is-on", on);
      });
      if (i++ < end) setTimeout(step, speed);
      else { chars.forEach((c) => c.classList.remove("is-on")); resolve(); }
    };
    step();
  });
}

// 1. cible en boucle continue au load
const explEl = document.querySelector(".scroll-explore");
if (explEl) {
  const chars = splitChars(explEl);
  const loop = () => sweep(chars, { speed: 90 }).then(() => setTimeout(loop, 1500));
  loop();
}

// 2. nav links au survol
document.querySelectorAll(".nav-link").forEach((el) => {
  const chars = splitChars(el);
  let running = false;
  el.addEventListener("mouseenter", () => {
    if (running) return;
    running = true;
    sweep(chars, { speed: 70 }).then(() => (running = false));
  });
});

// 3. bouton au survol
const letTalk = document.querySelector(".let-talk-wrapper");
if (letTalk) {
  const target = letTalk.querySelector(".text-weight-normal") || letTalk;
  const chars = splitChars(target);
  let running = false;
  letTalk.addEventListener("mouseenter", () => {
    if (running) return;
    running = true;
    sweep(chars, { speed: 70 }).then(() => (running = false));
  });
}
```

## Pièges rencontrés et résolutions

- Classe générique ciblée : le scroll to explore portait `text-size-regular`, partagée par d'autres éléments. Ajout d'une combo `scroll-explore` pour ne viser que lui.
- Bouton déjà noir : un surlignage noir dessus serait invisible. Règle CSS inversée pour ce cas.
- Relance au survol pendant un balayage en cours : garde `running` pour ne pas empiler les cycles.

## Réglages

- Vitesse : `speed` en ms par lettre. Plus petit égale plus rapide. 90 pour la boucle, 70 au survol.
- Largeur du balayage : `band`. À 1, une lettre allumée à la fois. Monter à 3 pour une traînée.
- Pause entre deux boucles : le `setTimeout(loop, 1500)`.

## Variante

Pour un remplissage qui reste allumé jusqu'au bout du mot au lieu d'un balayage qui passe, augmenter `band` jusqu'à la longueur du texte, ou retirer la condition de fenêtre glissante pour laisser les lettres allumées.
