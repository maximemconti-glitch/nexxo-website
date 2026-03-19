# Nexxo Website — Guide de construction

Site en production : **https://nexxo.systems**
Repo GitHub Pages : `maximemconti-glitch/nexxo-website` — déploiement depuis `main`.
Fichier source : `nexxo-website-standalone/index.html` (tout-en-un — CSS + JS inline)

---

## Architecture

**Single-file HTML** — pas de framework, pas de build step. Tout dans `index.html` :
- CSS dans `<style>` en tête de fichier
- JS inline en bas de `<body>`, organisé en IIFEs
- Fonts : Inter, Archivo, JetBrains Mono (Google Fonts CDN)

**Assets externes** (`macbook-assets/`) :
- `three.module.min.js` — Three.js ES module
- `GLTFLoader.js` + `BufferGeometryUtils.js` — chargement du modèle 3D
- `gsap.min.js` — animations MacBook (chargé en `<script>` classique AVANT le module)
- `mac-noUv.glb` — modèle 3D MacBook (627 Ko)
- `keyboard-overlay.png`, `macbook-screen-texture.png`

**Workflow de déploiement** :
```
1. Modifier nexxo-website-standalone/index.html
2. sed 's|../macbook-assets/|./macbook-assets/|g' index.html > /tmp/nexxo-deploy/index.html
3. git commit + git push → nexxo-website repo → GitHub Pages déploie automatiquement
```

---

## Design System

### Couleurs
```css
--accent:     #0071e3   /* bleu Apple */
--bg:         #000000
--text:       #ffffff
--accent-rgb: 0, 113, 227
```

### Typographie
- Titres : **Inter** (700)
- Corps : **Archivo** (300/400)
- Mono : **JetBrains Mono**

---

## Stack Scroll System

Le système signature : les sections montent physiquement sur les précédentes via `position: fixed` + JS.

### Setup initial
```js
const PANEL_IDS = ['process', 'realisations', 'team', 'join'];

panels.forEach((el, i) => {
  el.classList.remove('stack-slide'); // retirer position:sticky du CSS
  Object.assign(el.style, {
    position: 'fixed', top: '0', left: '0',
    width: '100%', height: 'auto', minHeight: '100vh',
    overflow: 'hidden',
  });
  el.style.setProperty('z-index', String(4 + i), 'important'); // passe le !important CSS
});
```

### Calcul des espaces (recalc)
```js
overflowH  = Math.max(0, el.offsetHeight - vh)               // contenu > 100vh
enterSize  = ENTER_DUR * vh                                   // animation d'entrée
dwellSize  = parseFloat(el.dataset.dwell || DWELL_DUR) * vh  // temps de lecture
totalSpace = enterSize + overflowH + dwellSize
spacer.style.height = (cumulOffset + vh) + 'px'
```

### Paramètres actuels
```js
const ENTER_DUR  = 0.55;  // animation d'entrée (fraction de vh)
const DWELL_DUR  = 0.45;  // pause de lecture avant que la section suivante monte
const HERO_DWELL = 0.45;  // pause sur le hero
```

### Attributs HTML
- `data-pinned="true"` → contenu ne scrolle pas, expose `progress` 0→1 via event `pinscroll`
- `data-dwell="1.5"` → override le temps de lecture

### Hero fade-out
```js
// hp = 0→1 pendant que process monte
hero.style.opacity = Math.max(0, 1 - Math.pow(hpe, 3.5)).toFixed(3);
// Exposant 3.5 = hero reste visible longtemps, fade seulement sur le dernier tiers
hero.style.transform = `scale(${1 - hpe * 0.08}) translateY(${-hpe * 3}%)`;
```

---

## Auto-Snap Bidirectionnel

Quand l'utilisateur atteint la fin d'une section, la suivante monte automatiquement.

### Triggers
```js
// Vers le bas : franchir fin du contenu + dwell
down.push({
  trigger: baseScroll + d.offset + d.enterSize + d.overflowH + d.dwellSize,
  target:  baseScroll + nextD.offset + nextD.enterSize,
});

// Vers le haut : franchir le début d'entrée d'une section
up.push({
  trigger: baseScroll + d.offset + d.enterSize,
  target:  prev ? baseScroll + prev.offset + prev.enterSize + prev.overflowH + prev.dwellSize : 0,
});
```

### Animation snap
```js
const SNAP_SPEED = 1300; // ms pour 1×vh — vitesse de référence

// Durée proportionnelle à la distance → vitesse perçue constante
const speedRef = dir > 0 ? SNAP_SPEED * 2 : SNAP_SPEED; // descente délibérément plus lente
const snapMs = Math.min(2200, Math.max(400, Math.abs(dist) / window.innerHeight * speedRef));
const ease = -(Math.cos(Math.PI * t) - 1) / 2; // easeInOutSine — même dans les deux sens
```

### Points critiques
- Le trigger down doit inclure `dwellSize` — sinon le snap se déclenche dès que la section finit d'entrer (avant que l'utilisateur ait le temps de lire)
- Cooldown **directionnel** : `if (Date.now() < _snapUntil && dir === _lastSnapDir) return` — ne bloque pas le sens inverse
- `prev <= p.trigger` (pas strict `<`) — évite de rater le trigger quand `_prevSY === trigger` exactement (cas post-snap)
- Bloquer wheel/touchmove pendant le snap : `addEventListener('wheel', ev => { if (_snapping) ev.preventDefault(); }, { passive: false })`

---

## MacBook 3D (Three.js)

### Chargement correct des modules
```html
<!-- GSAP : script classique, AVANT le module -->
<script src="./macbook-assets/gsap.min.js"></script>

<!-- Three.js : ES module avec ./ obligatoire -->
<script type="module">
import * as THREE from "./macbook-assets/three.module.min.js";
import { GLTFLoader } from "./macbook-assets/GLTFLoader.js";
```

**⚠️ Imports ES module : toujours préfixer avec `./`**
- `"./macbook-assets/..."` ✅ valide
- `"macbook-assets/..."` ❌ bare specifier — Chrome rejette silencieusement

### Slide 0 — Live Canvas animé
```js
const liveCanvas0 = document.createElement('canvas');
liveTexture0 = new THREE.CanvasTexture(liveCanvas0);
liveTexture0.flipY = false;

function render() {
  if (!document.hidden && window._processVisible !== false) {
    if (currentSlideIdx === 0) drawSlide0(Date.now());
    renderer.render(scene, camera);
  }
  requestAnimationFrame(render); // toujours planifier, jamais bloquer
}
```

### Optimisation performance
Le renderer Three.js est pausé hors de la section process :
```js
// Dans update() du stack scroll :
if (el.dataset.pinned) {
  window._processVisible = coverP < 1; // false quand realisations couvre process
}
if (enterP === 0 && el.dataset.pinned) window._processVisible = false;
if (enterP < 1  && el.dataset.pinned) window._processVisible = true; // actif dès la montée
```

### Ouverture MacBook — Bug double trigger
**Problème** : à progress ~0.02, `_mbEnter()` (> 0.01) ET `_mbClose()` (< 0.07) se déclenchent sur le même tick → ouvre, ferme, réouvre.

**Fix** :
```js
let _prevProgress = 0;
// Close < 0.07 uniquement si l'utilisateur REMONTE (pas en entrant)
const goingBack = progress < _prevProgress;
if (mbEntered && progress > 0.93) window._mbClose();
if (mbEntered && progress < 0.07 && goingBack) window._mbClose();
if (mbEntered && progress > 0.12 && progress < 0.88) window._mbReopen();
_prevProgress = progress;
```

---

## Déploiement GitHub Pages

### Structure des repos
- `agence-ia` (privé) — monorepo contenant tout le code Nexxo
- `nexxo-website` (public) — repo GitHub Pages dédié, servi depuis `main`

### Procédure
```bash
# Cloner le repo pages si pas déjà fait
git clone https://github.com/maximemconti-glitch/nexxo-website.git /tmp/nexxo-deploy

# Générer la version prod (corriger les chemins)
sed 's|../macbook-assets/|./macbook-assets/|g' \
  nexxo-website-standalone/index.html > /tmp/nexxo-deploy/index.html

# Copier les assets si modifiés
cp -r macbook-assets/ /tmp/nexxo-deploy/
cp -r images/team/ /tmp/nexxo-deploy/images/team/

# Déployer
cd /tmp/nexxo-deploy
git add index.html macbook-assets/ images/
git commit -m "deploy: ..."
git push origin main
# → Site live sur nexxo.systems en 1-3 min
```

---

## Sections

| Section | ID | Particularité |
|---|---|---|
| Hero | `#hero` | Fade-out sur entrée de process (exposant 3.5) |
| Notre process | `#process` | `data-pinned="true"`, 540vh, MacBook 3D intégré |
| Ce qu'on construit | `#realisations` | `data-dwell="1.5"`, 6 widgets animés |
| Notre équipe | `#team` | Cartes avec tilt 3D |
| Rejoindre Nexxo | `#join` | Dernier panneau, contient le footer |

**IDs des widgets `#hero-widgets`** : caché (`display:none`), contient des doublons d'IDs — ne jamais y mettre des IDs fonctionnels.

---

## Bugs résolus — référence

| Bug | Cause | Fix |
|---|---|---|
| MacBook s'ouvre deux fois | `_mbEnter` et `_mbClose` sur le même tick (progress 0.01–0.07) | Conditionner `_mbClose < 0.07` à `goingBack` |
| Snap UP revient en haut de process | Target UP manquait `overflowH + dwellSize` | Inclure les deux dans la target |
| `#team` sans animation de scroll | `position: relative` dans la règle CSS `#team` overridait `.stack-slide` | Supprimer `position: relative` de `#team` |
| MacBook ne charge pas sur HTTPS | Imports sans `./` → bare specifier rejeté par Chrome | Préfixer : `"./macbook-assets/..."` |
| Snap manqué après un snap | `prev < trigger` strict raté quand `_prevSY === trigger` exactement | `prev <= p.trigger` |
| Cooldown bloque le sens inverse | Cooldown global sans vérification de direction | `dir === _lastSnapDir` dans la condition |
| Renderer Three.js tourne en permanence | Aucun check de visibilité | Flag `_processVisible` + `document.hidden` |
| Hero disparaît trop vite | Exposant 0.8 dans la courbe opacity (chute rapide) | `Math.pow(hpe, 3.5)` |
