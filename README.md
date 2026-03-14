# Nexxo Website — Guide de construction

Site en production : **https://nexxo.systems**
Repo : `maximemconti-glitch/nexxo-website` — déploiement GitHub Pages depuis `main`.

---

## Architecture

**Single-file HTML** — pas de framework, pas de build step. Tout dans `index.html` :
- CSS dans `<style>` en tête de fichier
- JS inline en bas de `<body>`, organisé en IIFEs
- Fonts Google Fonts (Space Grotesk, Archivo, JetBrains Mono, Cormorant Garamond)

Workflow : modifier `index.html` → commit sur `dev` → merge sur `main` → déployé automatiquement.

---

## Design System

### Couleurs
```css
--accent:       #0071e3   /* bleu Apple */
--accent-light: #34aadc
--bg:           #000000
--text:         #ffffff
```

Sections alternent noir/blanc pour créer du rythme :
- Hero → blanc
- Ce qu'on construit → noir (fond sombre)
- Les chiffres qui parlent → **noir** `#06060c`
- Automatisation sans limites → **blanc** `#f5f5f7`
- Prêt à automatiser (CTA) → **noir** `#06060c`
- Notre équipe → noir
- Rejoindre Nexxo → noir

### Typographie
- Titres : **Space Grotesk** (700, tracking serré `-0.04em`)
- Corps : **Archivo** (300/400)
- Mono : **JetBrains Mono**
- Accent italique hero : **Cormorant Garamond** italic — donne un contraste éditorial fort

### Hero title pattern
```html
<h1 class="hero-title">
  L'IA qui automatise<br>
  <em>votre entreprise.</em>
</h1>
```
Le `<em>` prend Cormorant Garamond italic + gradient bleu via `-webkit-background-clip: text`.

---

## Stack Scroll

Le système signature du site : les sections montent physiquement sur les précédentes.

**Principe** : `position: fixed` sur chaque panneau + JS qui contrôle `translateY`.

```js
// Chaque section est fixée en haut
el.style.position = 'fixed';
el.style.top = '0';

// Un spacer div crée l'espace de scroll
// translateY va de 100vh (hors écran bas) à 0px (visible)
```

**Paramètres clés** :
- `ENTER_DUR = 0.8` : fraction de vh pour l'animation d'entrée
- `baseScroll = 0` : les panneaux commencent à entrer dès le 1er pixel de scroll
- `spacer height = cumulOffset + vh` : +vh pour que le dernier panneau puisse finir son entrée
- Sections trop hautes : phase `viewProgress` scrolle le contenu avant de laisser passer la suivante

**Gotchas résolus** :
- Z-index CSS `!important` → utiliser `el.style.setProperty('z-index', '4', 'important')`
- `height: 100vh; overflow: hidden` clippait les sections hautes → `height: auto; min-height: 100vh`
- Opacité sur la phase "cover" → visuellement transparent, supprimé (opacity toujours 1)
- Section `#join` transparente → avait besoin d'un `background` explicite

---

## Hero Layout

```
┌─────────────────────────────────────────────┐
│  [TEXTE GAUCHE]          [4 WIDGETS DROITE] │
│                                             │
│  Eyebrow pill badge      ┌────┬────┐        │
│  H1 grand (Space Grotesk)│ D  │ T  │        │
│    + em (Cormorant)      ├────┼────┤        │
│  Subtitle léger          │SAV │ G  │        │
│  [CTA] [Secondaire]      └────┴────┘        │
└─────────────────────────────────────────────┘
```

- Texte gauche : `max-width: calc(50vw - max(64px, 8vw) - 32px)` → ne déborde jamais sur les widgets
- Widgets : `position: absolute; right: 32px; width: calc(50% - 32px)` — grille 2×2
- Wrapper widgets : `background: rgba(0,0,0,0.04); border-radius: 26px; padding: 8px` → bloc unifié

### 4 Widgets
| Card | Contenu |
|------|---------|
| **Dashboard** | 3 stats (count-up) + activité récente + pulse live |
| **Terminal** | Feed d'events automatisés en boucle (10s) |
| **SAV IA Chat** | Conversation animée avec typing indicator, boucle |
| **Gestion interne** | 3 onglets : Planning / Facture / Stocks, auto-rotate 5.5s |

---

## Curseur Custom

**Single element, `mix-blend-mode: difference`** — le cercle blanc inverse les couleurs sous lui.

```css
#cursor {
  width: 18px; height: 18px;
  background: #ffffff;
  mix-blend-mode: difference;
}
body.cursor-hover #cursor { width: 64px; height: 64px; }
```

Éléments interactifs qui déclenchent l'agrandissement :
```js
'a:not(.dash-tab), button:not(.dash-tab), .spec-item, .team-card, .nav-cta, .btn-primary, .btn-secondary, .btn-dark'
```

**Ne pas inclure** : `.bento-card`, `.browser-card`, `.dash-tab` → curseur reste petit.

---

## Effets hover

### Particules sur boutons
Au hover : burst de points bleu/blanc qui se dispersent.
- Couleurs : `#0071e3`, `#34aadc`, `#ffffff` (majorité bleu)
- 3 particules à chaque `mousemove`, 10 au `mouseenter`
- Toutes les 40ms en `setInterval` pendant le survol
- Boutons concernés : `.btn-primary`, `.btn-secondary`, `.nav-cta`, `.btn-dark`

### Tilt 3D
```js
applyTilt('.bento-card', 8, 12);      // Fort
applyTilt('.spec-item.revealed', 5, 6); // Subtil
applyTilt('.browser-card', 11, 18);   // Accentué + ombres profondes
applyTilt('.team-card', 4, 6);        // Subtil
```

### Browser cards (portfolio)
- Ombre de base : `0 8px 28px rgba(0,0,0,0.10)` — flottent par défaut
- Ombre hover : `0 48px 90px rgba(0,0,0,0.22), 0 16px 32px...` — détachement dramatique

---

## Animations JS notables

### Count-up
```js
function countUp(id, target) {
  let v = 0; const step = target / 40;
  const t = setInterval(() => {
    v = Math.min(v + step, target);
    el.textContent = Math.round(v);
    if (v >= target) clearInterval(t);
  }, 28);
}
```

### Terminal feed en boucle
```js
function launchFeed() {
  EVENTS.forEach(ev => {
    setTimeout(() => { /* créer et append entry */ }, ev.delay);
  });
  setTimeout(() => { feed.innerHTML = ''; launchFeed(); }, 10000);
}
```

### Stagger d'apparition des cards
```css
#hero-widgets .widget-card:nth-child(n) { animation-delay: 0.15s * n; }
@keyframes cardIn { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
```

---

## Ce qui a bien marché

- **Mix-blend-mode cursor** : beaucoup plus premium qu'un dot+ring classique
- **Sections position:fixed** : l'effet "mont" est bien plus satisfaisant que sticky
- **Thème blanc sur features** : les cards dark sur fond blanc = effet "screenshot produit Apple"
- **Particules sur boutons** : simple à implémenter, très impactant visuellement
- **Wrapper teinté sur widgets** : soude les 4 cards en un seul bloc visuel sans border

## Ce qui est à améliorer la prochaine fois

- **Responsive** : les widgets disparaissent sous 1100px mais pas de layout alternatif
- **Performance** : plusieurs `setInterval` actifs en même temps — envisager IntersectionObserver pour pauser hors viewport
- **Accessibilité** : curseur custom masque le curseur natif sur tous les éléments, à repenser
