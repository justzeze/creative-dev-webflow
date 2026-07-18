# 01 — WebGL Deform (distorsion par vélocité + aberration chromatique)

## Nom technique

Displacement distortion piloté par la vélocité de la souris, avec RGB split (aberration chromatique), rendu WebGL sur texture. Effet type CodeGrid / Zajno. Une grille de données stocke un champ de déplacement qui se relâche dans le temps, la souris y injecte de la force selon sa vitesse, un shader échantillonne la texture décalée par canal R, G, B.

## Faisabilité Webflow

Custom code obligatoire, via Three.js. Rien de natif. Le point clé d'adaptation : un shader ne travaille que sur une texture. Une vidéo en est une nativement. Le texte et le SVG ne le sont pas, il faut donc les rasteriser d'abord.

- Texte : rasterisé via html2canvas.
- SVG inline : sérialisé en image (plus net qu'une capture).
- Le shader est rendu conscient de l'alpha pour détourer texte et logo au lieu de les plaquer sur un fond noir.

## HTML / classes requises

L'effet cible une combo dédiée `fx-deform`, posée sur chaque élément à animer. Ne jamais cibler une classe utilitaire générique comme `heading-style-h1` directement, sinon l'effet touche tous les H1 du site.

- Sur le titre : combo `heading-style-h1` + `fx-deform`.
- Sur le logo SVG footer : combo `footer-logo-content` + `fx-deform`.

La classe `fx-deform` peut rester vide de styles, elle sert uniquement de cible JS.

## Dépendances

Chargées en modules ES, tout en haut du fichier Slater, une seule fois :

```javascript
import * as THREE from "https://esm.sh/three@0.160.0";
import html2canvas from "https://esm.sh/html2canvas@1.4.1";
```

Ne jamais utiliser de specifier nu (`from "three"`) dans le navigateur, il ne se résout pas hors bundler.

## JS

À placer à la fin du fichier Slater. Enfermé dans une IIFE pour éviter tout conflit de variables avec le reste du code.

```javascript
/* ================= FX DEFORM ================= */
(() => {
  const GRID_SIZE = 25, MOUSE_RADIUS = 0.25, STRENGTH = 0.1;
  const RELAXATION = 0.925, DISPLACEMENT = 0.015, ABERRATION = 0.12;

  const isSvg = (el) => el.tagName.toLowerCase() === "svg";

  // --- rasterisation : texte via html2canvas, SVG via sérialisation ---
  async function elementToTexture(el) {
    if (isSvg(el)) {
      const rect = el.getBoundingClientRect();
      const dpr = Math.min(window.devicePixelRatio, 2);
      const clone = el.cloneNode(true);
      clone.setAttribute("width", rect.width);
      clone.setAttribute("height", rect.height);
      clone.style.color = getComputedStyle(el).color; // résout les currentColor
      const xml = new XMLSerializer().serializeToString(clone);
      const url = "data:image/svg+xml;base64," + btoa(unescape(encodeURIComponent(xml)));
      const img = await new Promise((res, rej) => {
        const im = new Image(); im.onload = () => res(im); im.onerror = rej; im.src = url;
      });
      const c = document.createElement("canvas");
      c.width = rect.width * dpr; c.height = rect.height * dpr;
      c.getContext("2d").drawImage(img, 0, 0, c.width, c.height);
      const t = new THREE.CanvasTexture(c);
      t.minFilter = t.magFilter = THREE.LinearFilter;
      return t;
    }
    if (document.fonts?.ready) await document.fonts.ready; // texte net
    const c = await html2canvas(el, { backgroundColor: null, scale: Math.min(window.devicePixelRatio, 2) });
    const t = new THREE.CanvasTexture(c);
    t.minFilter = t.magFilter = THREE.LinearFilter;
    return t;
  }

  // --- montage du canvas sans casser le flux ---
  function mountCanvas(el, canvas) {
    Object.assign(canvas.style, {
      position: "absolute", top: "0", left: "0",
      width: "100%", height: "100%", pointerEvents: "auto",
    });
    if (isSvg(el)) {
      // on enveloppe le SVG dans un wrapper relatif qui garde sa place dans le flux.
      // NE PAS positionner le canvas en absolu sur offsetLeft/Top, ça déraille si le SVG fait 100% de large.
      const wrapper = document.createElement("div");
      wrapper.style.position = "relative";
      wrapper.style.display = "block";
      wrapper.style.width = "100%";
      el.parentNode.insertBefore(wrapper, el);
      wrapper.appendChild(el);
      wrapper.appendChild(canvas);
      el.style.visibility = "hidden";
    } else {
      if (getComputedStyle(el).position === "static") el.style.position = "relative";
      el.appendChild(canvas);
      [...el.children].forEach((ch) => { if (ch !== canvas) ch.style.visibility = "hidden"; });
      el.style.color = "transparent";
    }
  }

  async function createDeformEffect(el) {
    let width = Math.ceil(el.getBoundingClientRect().width);
    let height = Math.ceil(el.getBoundingClientRect().height);
    if (!width || !height) { console.warn("[fx] taille nulle, ignoré", el); return; }
    let gridX, gridY;
    const mouse = { x: 0.5, y: 0.5, prevX: 0.5, prevY: 0.5, vX: 0, vY: 0 };

    const scene = new THREE.Scene();
    const camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0.1, 10);
    camera.position.z = 1;

    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setClearColor(0x000000, 0);
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    const canvas = renderer.domElement;

    const texture = await elementToTexture(el);
    mountCanvas(el, canvas);

    function createDataTexture() {
      const aspect = width / height;
      gridX = aspect >= 1 ? Math.round(GRID_SIZE * aspect) : GRID_SIZE;
      gridY = aspect >= 1 ? GRID_SIZE : Math.round(GRID_SIZE / aspect);
      const data = new Float32Array(gridX * gridY * 4);
      const t = new THREE.DataTexture(data, gridX, gridY, THREE.RGBAFormat, THREE.FloatType);
      t.magFilter = t.minFilter = THREE.NearestFilter; t.needsUpdate = true;
      return t;
    }
    let dataTexture = createDataTexture();

    const material = new THREE.ShaderMaterial({
      transparent: true,
      uniforms: { uTexture: { value: texture }, uDataTexture: { value: dataTexture } },
      vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0); }`,
      fragmentShader: `
        uniform sampler2D uTexture, uDataTexture; varying vec2 vUv;
        void main(){
          vec4 offset = texture2D(uDataTexture, vUv);
          vec2 shift = ${DISPLACEMENT} * offset.rg;
          vec2 split = shift * ${ABERRATION};
          vec4 cr = texture2D(uTexture, vUv - shift + split);
          vec4 cg = texture2D(uTexture, vUv - shift);
          vec4 cb = texture2D(uTexture, vUv - shift - split);
          gl_FragColor = vec4(cr.r, cg.g, cb.b, cg.a); // alpha détouré
        }`,
    });

    scene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material));

    canvas.addEventListener("mousemove", (e) => {
      const rect = canvas.getBoundingClientRect();
      const x = (e.clientX - rect.left) / rect.width;
      const y = (e.clientY - rect.top) / rect.height;
      mouse.vX = x - mouse.prevX; mouse.vY = y - mouse.prevY;
      mouse.prevX = mouse.x; mouse.prevY = mouse.y; mouse.x = x; mouse.y = y;
    });

    // --- tactile (mobile) : équivalent du hover, passif pour ne pas bloquer le scroll ---
    const setFromTouch = (t, reset) => {
      const rect = canvas.getBoundingClientRect();
      const x = (t.clientX - rect.left) / rect.width;
      const y = (t.clientY - rect.top) / rect.height;
      if (reset) { mouse.x = mouse.prevX = x; mouse.y = mouse.prevY = y; return; }
      mouse.vX = x - mouse.prevX; mouse.vY = y - mouse.prevY;
      mouse.prevX = mouse.x; mouse.prevY = mouse.y; mouse.x = x; mouse.y = y;
    };
    canvas.addEventListener("touchstart", (e) => { if (e.touches[0]) setFromTouch(e.touches[0], true); }, { passive: true });
    canvas.addEventListener("touchmove", (e) => { if (e.touches[0]) setFromTouch(e.touches[0], false); }, { passive: true });

    function updateDataTexture() {
      const data = dataTexture.image.data;
      for (let i = 0; i < data.length; i += 4) { data[i] *= RELAXATION; data[i + 1] *= RELAXATION; }
      const gmx = gridX * mouse.x, gmy = gridY * (1 - mouse.y), maxD = GRID_SIZE * MOUSE_RADIUS;
      for (let i = 0; i < gridX; i++) for (let j = 0; j < gridY; j++) {
        const dSq = (gmx - i) ** 2 + (gmy - j) ** 2;
        if (dSq >= maxD * maxD) continue;
        const idx = 4 * (i + gridX * j);
        const power = Math.min(10, maxD / Math.sqrt(dSq));
        data[idx] += STRENGTH * 100 * mouse.vX * power;
        data[idx + 1] -= STRENGTH * 100 * mouse.vY * power;
      }
      mouse.vX *= 0.9; mouse.vY *= 0.9; dataTexture.needsUpdate = true;
    }

    renderer.setAnimationLoop(() => { updateDataTexture(); renderer.render(scene, camera); });
  }

  function initDeform() {
    document.querySelectorAll(".fx-deform").forEach((el) =>
      createDeformEffect(el).catch((err) => console.error("[fx] échec sur", el, err))
    );
  }

  // se lance même si la page est déjà chargée (Slater injecte le script tardivement)
  if (document.readyState === "complete") initDeform();
  else window.addEventListener("load", initDeform);
})();
/* ============================================= */
```

## Pièges rencontrés et résolutions

- Import nu `from "three"` dans Slater : ne se résout pas dans le navigateur. Utiliser esm.sh. Et une seule erreur d'import tue tout le module, donc le reste du JS (scramble, player) meurt avec. Symptôme trompeur : "mon scramble ne marche plus non plus".
- SVG monté en absolu sur offsetLeft/Top : le logo débordait et chevauchait le footer car le SVG fait width 100% d'un grand conteneur. Solution : envelopper le SVG dans un wrapper relatif et poser le canvas dedans, la place dans le flux est conservée.
- Fond noir plein au lieu du détourage : shader non conscient de l'alpha. Corrigé par `gl_FragColor.a = cg.a`, `material.transparent = true`, `alpha: true` sur le renderer et `setClearColor(0, 0)`.
- `window.load` déjà passé : Slater injecte le script tard, le listener ne se déclenche jamais. Garde `document.readyState === "complete"`.
- Classe générique ciblée : impose la combo `fx-deform`.

## Réglages

- `ABERRATION` : intensité du décalage RGB. 0.12 par défaut pour du texte fin, descendre vers 0.08 si trop violent.
- `STRENGTH` : force de la distorsion injectée.
- `RELAXATION` : vitesse de retour au repos, plus proche de 1 égale traînée plus longue.
- `MOUSE_RADIUS` : rayon d'influence de la souris.
- `GRID_SIZE` : finesse de la grille de déplacement.

## Points de vigilance

- Halo sombre sur les bords : alpha non prémultiplié, ajustable sur la texture si visible.
- Perf : chaque élément ouvre un contexte WebGL. Sur mobile, si ça chauffe, ne pas appliquer l'effet en dessous d'une largeur donnée.
- Accessibilité : le texte et le SVG restent dans le DOM, juste masqués visuellement, donc lisibles par les lecteurs d'écran et le SEO.
- Mobile : l'effet est piloté par mousemove, absent au tactile. Ajouter les listeners touchstart et touchmove en passif (voir JS) pour l'activer au doigt sans bloquer le scroll. Sans ça, le rendu s'affiche mais reste inerte sur mobile.
