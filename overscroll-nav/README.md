# 06 — Navigation over-scroll entre projets + compteur voyageur

## Nom technique

Navigation entre pages CMS déclenchée par over-scroll aux extrémités, avec indicateur de progression qui se déplace physiquement le long du bord. Simule le scroll continu entre projets du modèle Monavon, en architecture multi-pages.

## Faisabilité Webflow

Custom code léger plus deux champs CMS auto-référencés. Le choix d'architecture est structurant :

- **Modèle A, multi-pages CMS (retenu)** : chaque projet garde sa page template. Le scroll infini est simulé par navigation à l'over-scroll. SEO propre, CMS simple, sticky natif suffisant pour la colonne info.
- **Modèle B, page unique** : tous les projets empilés, colonne fixed échangée en JS. Refonte complète, perte des avantages CMS. Non retenu.

Le JS ne connaît pas les URLs voisines, seul le CMS les a. On les expose au DOM via des liens cachés liés à des champs Reference.

## Structure CMS

Sur la collection projet, deux champs **Reference** pointant vers la même collection :

- `next-project`
- `previous-project`

À remplir sur chaque item. Pour une boucle, le dernier pointe vers le premier. Un champ vide désactive la navigation dans ce sens, le script le gère.

## HTML / classes

Sur la page template :

- `scroll-progress` : Text Block "00%", fixe en bas à droite.
- `close-top` : mention "Projet précédent", placée **dans** la colonne média, en absolu à `top: -3rem`, centrée, donc hors contenu.
- `close-bottom` : mention "Projet suivant", même principe à `bottom: -3rem`.
- Deux **LinkBlock cachés** (classe `nav-ref`, display none) avec `data-nav="next"` et `data-nav="prev"`.

Liaison des liens cachés dans le Designer : réglage Link, panneau Connect, **déplier la référence** (chevron) puis choisir le **URL Path à l'intérieur**.

## CSS

```css
.close-top, .close-bottom {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  opacity: 0;
  z-index: 50;
  pointer-events: none;
  transition: opacity 300ms;
}
.close-top { top: -3rem; }
.close-bottom { bottom: -3rem; }

.scroll-progress {
  position: fixed;
  right: 1.5rem;
  bottom: 1.5rem;
  z-index: 50;
  pointer-events: none;
}

.nav-ref { display: none; }
```

Le conteneur média doit être en `position: relative` et **sans** `overflow: hidden`, sinon les mentions en offset négatif sont coupées.

## JS

Deux blocs **séparés**. Règle : un seul endroit écrit dans un élément donné. Le bloc navigation ne touche jamais au pourcentage.

```javascript
/* ===== Navigation projet : mentions + bascule entre projets ===== */
(() => {
  const closeTop = document.querySelector(".close-top");
  const closeBottom = document.querySelector(".close-bottom");
  if (!closeTop && !closeBottom) return; // page projet seulement

  const nextURL = document.querySelector('[data-nav="next"]')?.getAttribute("href") || "";
  const prevURL = document.querySelector('[data-nav="prev"]')?.getAttribute("href") || "";

  const THRESHOLD = 200; // effort d'over-scroll avant de changer de projet
  let overscroll = 0;

  const getMax = () => Math.max(1, document.documentElement.scrollHeight - window.innerHeight);
  const atTop = () => window.scrollY <= 2;
  const atBottom = () => window.scrollY >= getMax() - 2;

  function update() {
    if (closeTop) closeTop.style.opacity = (atTop() && prevURL) ? "1" : "0";
    if (closeBottom) closeBottom.style.opacity = (atBottom() && nextURL) ? "1" : "0";
  }
  window.addEventListener("scroll", update, { passive: true });
  update();

  function tryNav(delta) {
    if (atTop() && delta < 0 && prevURL) overscroll += -delta;
    else if (atBottom() && delta > 0 && nextURL) overscroll += delta;
    else { overscroll = 0; return; }
    if (overscroll >= THRESHOLD) {
      overscroll = 0;
      window.location.href = delta > 0 ? nextURL : prevURL;
    }
  }

  window.addEventListener("wheel", (e) => tryNav(e.deltaY), { passive: true });

  let lastY = null;
  window.addEventListener("touchstart", (e) => {
    lastY = e.touches[0]?.clientY ?? null;
    overscroll = 0;
  }, { passive: true });
  window.addEventListener("touchmove", (e) => {
    if (lastY == null) return;
    const y = e.touches[0].clientY;
    tryNav(lastY - y);
    lastY = y;
  }, { passive: true });
})();

/* ===== Pourcentage reel + trajet du compteur (bas vers milieu) ===== */
(() => {
  const progressEl = document.querySelector(".scroll-progress");
  if (!progressEl) return;

  progressEl.style.willChange = "transform";

  function update() {
    const maxScroll = Math.max(1, document.documentElement.scrollHeight - window.innerHeight);
    const ratio = Math.min(1, window.scrollY / maxScroll);   // 0 en haut, 1 en bas
    const pct = Math.round(ratio * 100);

    progressEl.textContent = String(pct).padStart(2, "0") + "%";

    const travel = window.innerHeight * 0.5;                 // course du bas vers le milieu
    progressEl.style.transform = `translateY(${ -ratio * travel }px)`;
  }
  window.addEventListener("scroll", update, { passive: true });
  window.addEventListener("resize", update, { passive: true });
  update();
})();
```

## Pièges rencontrés et résolutions

- **Lien branché sur le mauvais URL Path** : le panneau Connect propose un URL Path à la racine (le projet courant) et un URL Path dans chaque référence dépliable. Prendre celui de la racine fait naviguer vers la page courante. Symptôme : "ça revient sur le même projet". Toujours déplier la référence et choisir le URL Path indenté.
- **Champs Reference non cliquables** dans Connect : ce sont des dossiers, cliquer la flèche et non le nom.
- **Nouveau champ invisible dans le Designer** après création par API : recharger le Designer (Cmd+R).
- **"[Changes in draft]"** devant un item dans le sélecteur de référence : simple mention de modifications non publiées, l'item reste sélectionnable.
- **L'item en cours d'édition n'apparaît pas dans son propre sélecteur** : protection anti auto-référence de Webflow, normal.
- **Deux blocs qui écrivent dans le même élément** : la première version calculait le pourcentage dans le bloc navigation ET dans le bloc pourcentage. Les deux écouteurs de scroll s'écrasaient, le chiffre clignotait. Un élément, un seul écrivain.
- **Sens du pourcentage inversé** : `maxScroll - scrollY` donne 100% au chargement. Le calcul honnête est `scrollY / maxScroll`.
- **Vérification console** : `document.querySelector('[data-nav="next"]')?.getAttribute("href")` doit renvoyer l'URL du voisin. `undefined` signifie souvent qu'on teste sur la mauvaise page.

## Réglages

- `THRESHOLD` : insistance de l'over-scroll. 200 par défaut.
- `travel` : course du compteur, `innerHeight * 0.5` pour un trajet bas vers milieu. Ajuster entre 0.49 et 0.51 selon la hauteur du texte.
- Les mentions n'apparaissent que si un voisin existe.

## Extension possible

Transition de sortie avant navigation : remplacer `window.location.href = url` par une timeline GSAP qui fond le body, puis navigue dans son `onComplete`.
