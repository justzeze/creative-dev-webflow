# 07 — Curseurs contextuels Ouvrir / Fermer + zone cliquable

## Nom technique

Label textuel suivant le pointeur, en `mix-blend-mode: difference`, apparaissant uniquement sur une zone donnée. Couplé à une zone entièrement cliquable qui ouvre ou referme une page. Duo symétrique : "Ouvrir" sur les listes, "Fermer" sur la page projet.

## Faisabilité Webflow

Un élément texte, deux classes, et un bloc JS par zone. Aucune dépendance externe. Le point clé est le blend : le label s'inverse automatiquement selon le fond, donc il reste lisible sur une image sombre comme sur un aplat clair, sans avoir à gérer de variantes de couleur.

## HTML / classes

Un seul élément par page, au niveau du **body**, pour qu'il soit au-dessus de tout et hors de tout conteneur qui pourrait le couper.

- `close-cursor` avec le texte "Fermer", sur la page projet.
- `open-cursor` avec le texte "Ouvrir", sur la home et la page index.
- Combo `is-visible` sur chacun, état allumé.

La zone d'activation est un conteneur existant : `inner-side-wrapper` sur la page projet, `project-section` sur la home, `index-projet-media-wrapper` sur l'index. Elle reçoit `cursor: pointer`.

Pour la navigation par item, un **LinkBlock caché** (classe `nav-ref`, display none) avec l'attribut `data-project-url`, placé **dans le Collection Item**, lié à **URL Path**. Ici, contrairement à la fiche 06, c'est le URL Path **de la racine** qui est correct, puisqu'on est dans une Collection List et que l'item courant est le bon projet.

## CSS

```css
.close-cursor, .open-cursor {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 100;
  font-size: 0.7rem;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  color: #FFFFFF;
  mix-blend-mode: difference;   /* lisible sur n'importe quel fond */
  pointer-events: none;         /* ne bloque jamais le clic dessous */
  opacity: 0;
  transform: translate(-50%, -50%) scale(0.6);
  transition: opacity 200ms ease, transform 200ms ease;
}
.close-cursor.is-visible,
.open-cursor.is-visible {
  opacity: 1;
  transform: translate(-50%, -50%) scale(1);
}
```

Pas de fond, pas de pilule, pas de padding : juste le mot. Le blend fait tout le travail de lisibilité.

## JS

### Curseur "Fermer" + clic de retour (page projet)

```javascript
/* ===== Curseur "Fermer" qui suit la souris ===== */
(() => {
  const zone = document.querySelector(".inner-side-wrapper");
  const cursor = document.querySelector(".close-cursor");
  if (!zone || !cursor) return;

  const OFFSET_X = 20;  // distance au pointeur, en px
  const OFFSET_Y = 20;

  let raf = null, mx = 0, my = 0;
  const move = () => {
    cursor.style.left = (mx + OFFSET_X) + "px";
    cursor.style.top  = (my + OFFSET_Y) + "px";
    raf = null;
  };
  zone.addEventListener("mousemove", (e) => {
    mx = e.clientX; my = e.clientY;
    if (!raf) raf = requestAnimationFrame(move);
  });
  zone.addEventListener("mouseleave", () => cursor.classList.remove("is-visible"));
  zone.addEventListener("mouseover", (e) => {
    if (e.target.closest('a, button, [role="button"]')) cursor.classList.remove("is-visible");
    else cursor.classList.add("is-visible");
  });
})();

/* ===== Clic sur la zone projet = retour a l'index, meme projet ===== */
(() => {
  const zone = document.querySelector(".inner-side-wrapper");
  if (!zone) return;

  zone.addEventListener("click", (e) => {
    if (e.target.closest('a, button, [role="button"], input, textarea, select')) return;

    const sameOrigin = document.referrer && new URL(document.referrer).origin === location.origin;
    if (sameOrigin && window.history.length > 1) {
      history.back();   // restaure la position de scroll, donc le bon projet
    } else {
      window.location.href = "/";   // arrivee directe, pas d'historique
    }
  });
})();
```

### Curseur "Ouvrir" + clic vers le projet (home et index)

```javascript
/* ===== Curseur "Ouvrir" + clic vers la page projet ===== */
(() => {
  const cursor = document.querySelector(".open-cursor");
  const zones = document.querySelectorAll(".index-projet-media-wrapper"); // ou .project-section
  if (!cursor || !zones.length) return;

  const OFFSET_X = 20, OFFSET_Y = 20;
  let raf = null, mx = 0, my = 0;
  const move = () => {
    cursor.style.left = (mx + OFFSET_X) + "px";
    cursor.style.top  = (my + OFFSET_Y) + "px";
    raf = null;
  };

  zones.forEach((zone) => {
    zone.addEventListener("mousemove", (e) => {
      mx = e.clientX; my = e.clientY;
      if (!raf) raf = requestAnimationFrame(move);
    }, { passive: true });
    zone.addEventListener("mouseenter", () => cursor.classList.add("is-visible"));
    zone.addEventListener("mouseleave", () => cursor.classList.remove("is-visible"));
    zone.addEventListener("click", (e) => {
      if (e.target.closest('a, button, [role="button"]')) return;
      const item = zone.closest(".index-projet-item");   // ou .project-collection-item
      const url = item?.querySelector("[data-project-url]")?.getAttribute("href");
      if (url) window.location.href = url;
    });
  });
})();
```

## Pièges rencontrés et résolutions

- **Le JS écrit dans `transform`, le CSS aussi** : le décalage posé en CSS était écrasé à chaque mouvement de souris, "je modifie et rien ne change". Séparer les rôles : le JS ne touche qu'à `left` et `top`, le CSS garde `transform` pour le centrage et l'apparition. Le décalage passe en constantes JS.
- **Doublon d'élément** : un `<div class="close-cursor">` collé dans un Code Embed **plus** un élément créé par API donnent deux curseurs superposés. N'en garder qu'un. L'API ne peut pas éditer le contenu d'un Code Embed, c'est donc à la main dans le Designer.
- **`opacity` de base repassée à 1** : le label reste alors affiché en permanence, figé à la dernière position. La base doit être à 0, c'est le JS qui allume via `is-visible`.
- **Fond noir illisible** : une pilule sombre avec texte blanc devient invisible quand elle est petite. Supprimer le fond et laisser le `mix-blend-mode: difference` gérer le contraste.
- **Le clic ne marche qu'en publié** : Slater ne tourne pas en preview. Tester sur l'URL publiée, pas dans le Designer.
- **`history.back()` plutôt qu'une URL** : c'est ce qui restaure la position de scroll de la page d'origine, donc l'utilisateur retombe exactement sur le projet qu'il avait ouvert. Prévoir le repli si l'historique est vide.

## Réglages

- `OFFSET_X` et `OFFSET_Y` : distance du label au pointeur, en pixels.
- `font-size` : 0.7rem, discret. Le blend rend le texte plus présent qu'il n'y paraît.
- `cursor: none` sur la zone si l'on veut masquer le curseur système, façon Monavon intégral. Attention, cela masque aussi le curseur sur les liens internes à la zone.

## Points de vigilance

- **Mobile** : pas de survol, donc pas de label. Le clic, lui, fonctionne. Comportement dégradé acceptable par nature.
- **Le blend n'agit que dans le même contexte d'empilement**. Un ancêtre isolant casse l'inversion. L'élément est posé au body précisément pour éviter ça.
- **Sélection de texte** : `cursor: pointer` sur une grande zone rend la sélection moins naturelle. Acceptable sur un portfolio, à réévaluer sur un site éditorial.
