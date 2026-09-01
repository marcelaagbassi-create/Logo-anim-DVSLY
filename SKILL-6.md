---
name: dvsly-live-logo
description: Système de design du logo animé DVSLY pour l'écran du mode Live (conversation vocale avec l'IA). Utilise cette skill dès que l'utilisateur travaille sur l'identité visuelle du mode live de DVSLY, veut ajuster l'animation du logo (état repos ou état parole), demande une spec technique/brief créatif pour ce composant, ou mentionne "logo vivant", "onde vocale du logo", "état repos/parole", ou le fichier dvsly_logo_repos.html / dvsly_logo_parole.html. Contient la palette de marque exacte, les deux specs d'animation, et les prototypes HTML de référence.
---

# DVSLY — Logo animé du mode Live

Système de design pour l'écran d'accueil et d'écoute du mode Live de l'application DVSLY (conversation vocale avec l'IA, façon Gemini Live). Le logo de marque (le "X" en ruban croisé) a deux états, et la règle d'or est : **c'est toujours le vrai logo qui bouge — jamais une image de remplacement.** On distord, colore et anime la géométrie réelle du logo, on ne la remplace pas par un asset généré à part.

## Palette de marque (extraite du logo officiel)

| Zone | Couleur | Hex |
|---|---|---|
| Bras gauche, profond | Navy | `#003c50` |
| Bras gauche, milieu | Teal | `#0090a8` |
| Bras gauche, clair | Cyan | `#00b4c8` |
| Centre | Blanc chaud | `#eaf6f4` |
| Bras droit, profond | Rouge-orange | `#dc4028` |
| Bras droit, milieu | Orange | `#f06428` |
| Bras droit, clair | Ambre/Or | `#f8b400` |

Ne jamais utiliser un arc-en-ciel générique (multi-teintes non liées à la marque) — ça casse la reconnaissabilité. Toute variation de couleur doit rester dans cette gamme navy→cyan / rouge→or.

## État 1 — Repos (personne ne parle)

**Comportement attendu** : le logo est **vivant** — il se déforme en continu, de façon organique et clairement visible (pas un simple frémissement), comme s'il respirait de l'énergie en attendant que la conversation commence.

**Technique** : filtre SVG combinant `feTurbulence` (bruit organique, 3 octaves, qui dérive en continu) + `feDisplacementMap` appliqué directement sur l'`<image>` du vrai logo — jamais un asset séparé. Au repos :
- L'amplitude de déformation oscille en continu entre ~0.34 et ~0.80 (superposition de deux sinusoïdes à des fréquences différentes pour éviter tout effet mécanique/répétitif).
- `dispScale = amp × 90` — c'est ce multiplicateur qui donne une torsion nettement visible du ruban, pas juste un tremblement.
- `feTurbulence.baseFrequency` et `seed` dérivent lentement en continu (indépendamment de l'amplitude) pour que le mouvement ne se répète jamais identiquement.
- Un halo lumineux (`feGaussianBlur` + `feColorMatrix`, fusionnés via `feMerge`) respire avec la même amplitude.

Référence : `dvsly_logo_repos.html` (version masque/dégradé, alternative plus simple — voir note ci-dessous) et `dvsly_logo_parole.html` (version turbulence, qui contient aussi cet état repos).

> Note : deux techniques coexistent pour l'état repos selon le fichier — le masque CSS + dégradé qui défile (`dvsly_logo_repos.html`, mouvement de couleur uniquement, silhouette fixe) et la turbulence SVG (mouvement de forme). Trancher avec l'équipe design laquelle part en prod, ou les combiner (silhouette qui se déforme ET dégradé qui coule dedans).

## État 2 — Parole (l'utilisateur OU l'IA parle) — le logo se fige

**Comportement attendu** : c'est l'inverse du repos. Dès que quelqu'un parle, le logo **se fige immédiatement** sur sa forme d'origine exacte, sans aucune distorsion ni mouvement — comme s'il se concentrait, immobile, pour écouter/délivrer la parole.

**Technique** :
- Quand `speaking = true` : `amp` relâche rapidement vers 0 (facteur de lissage ~0.4/frame, snap final à 0 sous 0.004 pour éviter tout résidu visuel) → `dispScale` retombe à 0 → la forme affichée redevient un tracé exact et immobile du logo d'origine.
- La dérive de `feTurbulence` (baseFrequency/seed) s'arrête aussi pendant qu'on parle — rien ne bouge, pas même le bruit de fond du filtre.
- Le halo redescend à une opacité basse et fixe (0.2) pendant la parole, pour renforcer l'impression d'immobilité.

Référence : `dvsly_logo_parole.html`

### Intégration audio réelle (à faire côté app)

Le fichier de démonstration fait alterner automatiquement `speaking` entre `true`/`false` via un minuteur (`updateSpeakingCycle`), uniquement pour visualiser les deux états sans interaction. **En production, remplacer ce minuteur par une vraie détection vocale** :

```js
// Web Audio API — amplitude RMS en direct
const analyser = audioContext.createAnalyser();
analyser.fftSize = 256;
const data = new Uint8Array(analyser.frequencyBinCount);

function getAmplitude(){
  analyser.getByteTimeDomainData(data);
  let sum = 0;
  for(let i=0;i<data.length;i++){
    const v = (data[i]-128)/128;
    sum += v*v;
  }
  return Math.sqrt(sum/data.length); // ~0 à ~1
}

// speaking = true dès que l'amplitude dépasse un seuil bas (~0.06),
// que ce soit le flux micro (utilisateur) ou le flux TTS (IA) pendant qu'il joue.
const SILENCE_THRESHOLD = 0.06;
speaking = getAmplitude() > SILENCE_THRESHOLD;
```

- **Toi (utilisateur)** : brancher sur le flux micro.
- **IA** : brancher sur le flux audio du TTS pendant la lecture de la réponse.
- Garder l'asymétrie de lissage (relâchement rapide vers l'immobilité / reprise progressive du mouvement au retour au repos) — c'est ce qui donne la sensation de réactivité "vivante" sans que le logo tremble au moindre bruit de fond.

## Différenciation Toi / IA (optionnel, à trancher avec l'utilisateur)

Les deux prototypes actuels utilisent la même palette pour les deux locuteurs. Si on veut distinguer visuellement "tu parles" de "l'IA parle" sans texte, options à discuter :
- Légère bascule de dominante (plus cyan quand IA, plus orange quand utilisateur, ou l'inverse)
- Vitesse de dérive du bruit différente
- Position/forme du halo

Ne pas trancher seul — demander la préférence avant d'implémenter.

## Fichiers de référence

- `dvsly_logo_repos.html` — état repos, standalone, sans UI.
- `dvsly_logo_parole.html` — état parole, standalone, sans UI, avec simulation de voix + retour instantané au silence.
- Logo source : PNG officiel DVSLY (fond transparent), utilisé tel quel comme masque et comme image source du filtre — ne jamais retracer/reconstruire la forme à la main.

## Ce qu'il ne faut pas faire

- Ne pas utiliser une image statique différente (ex: une capture d'écran ou un asset "onde sonore" générique) pour représenter un état — la forme animée doit toujours dériver du vrai fichier logo.
- Ne pas inverser la logique : c'est le **repos** qui est vivant/déformé, et la **parole** qui fige le logo sur sa forme d'origine — pas l'inverse.
- Ne pas laisser une transition lente entre les deux états — le figement au début de la parole doit être quasi immédiat (relâchement rapide, pas une décroissance molle).
- Ne pas utiliser une palette hors de la gamme de marque (pas d'arc-en-ciel générique).
- Ne pas garder de minuteur de démonstration en production — `speaking` doit venir d'une vraie détection audio (micro ou flux TTS), pas d'un cycle automatique.
