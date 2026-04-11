# Prompt engineering Flux LoRA : le guide complet pour LinkedIn créatif

**Flux transforme radicalement le prompting par rapport à Stable Diffusion** : il comprend le langage naturel grâce à son encodeur T5-XXL, ignore totalement la syntaxe de poids `(mot:1.5)`, et ne supporte pas les negative prompts. Pour une coach utilisant un character LoRA sur LinkedIn, la clé réside dans une structure Subject → Action → Environnement → Éclairage → Style, des prompts de **40-50 mots en phrases descriptives**, un trigger word placé en début de prompt, et un guidance scale autour de **3.5** (bien loin du CFG 7-12 habituel de SD). Ce guide couvre l'intégralité de la chaîne, du prompt unitaire jusqu'aux instructions GPT prêtes à copier-coller.

---

## 1. Anatomie d'un prompt Flux LoRA : tout ce qui change par rapport à SD

Flux repose sur un double encodeur texte — CLIP pour l'alignement image-texte et **T5-XXL (~4,6 milliards de paramètres)** qui lit les prompts comme des phrases naturelles. Ce n'est plus du tag engineering : c'est de la description narrative.

### Ordre des éléments et front-loading

Flux accorde plus de poids aux premiers tokens du prompt. L'élément le plus important doit toujours apparaître en premier. La hiérarchie recommandée est :

1. **Trigger word du LoRA** (tout au début)
2. **Sujet principal** — qui est dans l'image, description physique
3. **Action/pose** — ce que fait le personnage
4. **Environnement** — où se passe la scène
5. **Éclairage** — décrit comme un comportement de lumière, pas juste un label
6. **Style et spécifications techniques** — appareil photo, mood, direction artistique

Un sujet enterré en fin de prompt risque d'être dé-priorisé. C'est l'erreur structurelle la plus courante.

### Ce qui ne fonctionne PAS avec Flux

Plusieurs réflexes issus de Stable Diffusion sont inutiles, voire contre-productifs avec Flux. **La syntaxe de poids** `(emphasis)++` ou `(word:1.5)` est totalement ignorée — Flux ne la parse pas. Les **quality tags** comme « masterpiece, best quality, 8k » n'ont qu'un impact marginal. Les **negative prompts ne sont pas supportés** nativement par l'architecture guidance-distilled de Flux ; passer un paramètre `negative_prompt` génère une erreur. La solution : reformuler positivement. Au lieu de « no blur, no bad hands », écrire « sharp focus, crisp detail, accurate hands, natural proportions ».

Il existe un workaround communautaire via Dynamic Thresholding dans ComfyUI, mais il double le temps de génération et nécessite un CFG ≥ 5 — à réserver aux cas extrêmes.

### Longueur optimale et capitalisation

Le sweet spot se situe entre **40 et 50 mots**. En dessous de 10 mots, le modèle « complète » le prompt en interne à partir de ses données d'entraînement — on perd le contrôle. Au-dessus de 200 mots, Flux résume/compresse et des éléments peuvent être ignorés. Les limites techniques sont de **512 tokens pour dev** et 256 pour schnell.

Détail important : la capitalisation est encodée par T5. « Vincent Van Gogh's style » et « vincent van gogh's style » peuvent produire des résultats significativement différents.

### Les spécifications caméra, un levier puissant

Contrairement à SD où les quality tags dominaient, Flux répond remarquablement bien aux **références de matériel photographique**. Nommer un boîtier et un objectif spécifiques façonne l'esthétique entière de l'image : « shot on Hasselblad X2D, 85mm f/1.4 » produit un rendu éditorial luxueux, « shot on iPhone 16 » donne un look candid/authentique, « Kodak Portra 400 » apporte des tons chair chaleureux et flatteurs.

---

## 2. Paramètres techniques : le CFG de Flux n'a rien à voir avec SD

La confusion la plus répandue concerne le guidance scale. Flux utilise un système à deux niveaux qu'il faut bien distinguer.

| Paramètre | Flux dev | Flux schnell |
|-----------|----------|-------------|
| CFG Scale classique | **1** (désactivé) | **1** (désactivé) |
| Distilled Guidance (le vrai réglage) | **3.5** par défaut | **3.0-3.5** |
| Steps recommandés | **20-30** (jusqu'à 50) | **1-4** (jusqu'à 8) |
| Sampler optimal | Euler | Euler |
| Scheduler optimal | Simple ou Beta | SGM Uniform |
| Résolution de base | 1024×1024 | 1024×1024 |
| Negative prompts | Non supportés | Non supportés |

Le **Distilled Guidance** est le paramètre critique. Pour du photoréalisme, descendre à **2.0-2.5** produit des images plus naturelles et moins « IA ». Pour des illustrations ou styles artistiques, monter à **3.5-6.0**. Pour des portraits créatifs type LinkedIn (semi-réalistes, colorés), une valeur de **3.0-4.0** offre le meilleur compromis.

Pour les samplers/schedulers, les tests communautaires (189 images comparées sur Civitai) montrent que **Euler + Beta** produit des couleurs plus chaudes et un micro-contraste légèrement supérieur — idéal pour des portraits colorés. **DPM++ 2M + Beta** excelle en photoréalisme pur mais est plus lent.

### Résolutions pour LinkedIn

Flux a été entraîné sur des images de 0,2 à 2 mégapixels. Le sweet spot est autour de **1,0-1,3 MP**. Les dimensions doivent être des multiples de 32. Pour LinkedIn mobile (format 4:5 qui maximise l'espace dans le feed), la résolution idéale est **832×1040** ou **864×1080**. Pour du carré (1:1), utiliser **1024×1024** ou **1152×1152**.

---

## 3. Character LoRA : garder la ressemblance tout en libérant la créativité

Le réglage de **LoRA strength** est le levier principal pour doser l'équilibre entre fidélité du visage et liberté créative. Le consensus communautaire place l'optimum à **0.8 pour un LoRA unique**. À 0.6-0.7, on gagne en flexibilité stylistique mais on peut perdre en ressemblance. Au-dessus de 1.0, les artefacts apparaissent (aspect plastique, surexposition de certains traits).

Quand on combine un character LoRA avec un style LoRA, la règle est de maintenir la **somme des poids autour de 1.2** — par exemple 0.9 pour le character + 0.25 pour le style. Au-delà, les LoRA entrent en conflit et produisent des artefacts visuels.

### Un point crucial sur les trigger words

Un insight contre-intuitif émerge des discussions de développeurs (SimpleTuner) : « Flux n'aime pas les trigger words — ce n'est pas naturel pour lui de les comprendre. T5 comprend le sens contextuel. Un trigger word ne fait que confondre le modèle. » La recommandation avancée est d'**intégrer le trigger word naturellement dans une phrase fluide** plutôt que de le plaquer au début comme un tag isolé. En pratique, pour un LoRA bien entraîné, la formulation `TRIGGERWORD, portrait of a woman speaking at a conference` fonctionne, mais `A portrait of TRIGGERWORD speaking confidently at a creative conference` peut donner de meilleurs résultats avec certains LoRA.

Règle d'or : **ne jamais répéter le trigger word**, et ne pas essayer de le pondérer (la syntaxe est ignorée). L'intensité de l'effet LoRA se contrôle exclusivement via le paramètre strength dans le loader.

⚠️ **Confusion critique : le prénom ≠ le trigger word.** Écrire `Jennifer` dans un prompt ne fait pas appel au LoRA — T5 génère une personne nommée Jennifer à partir de ses données d'entraînement. Seul le trigger word exact (ex: `MALORACOACH`) active le LoRA. Si le trigger word est inconnu, le laisser vide plutôt que de le remplacer par le prénom.

⚠️ **Pas de métadonnées dans le prompt.** Des formulations comme `"showcasing her distinctive features for the FLUX LoRA"` ou `"(LoRA identity confirmed)"` sont de la méta-description : Flux les interprète littéralement et essaie de rendre ces concepts visuellement. Tout ce qui est dans le prompt est traité comme une instruction de génération.

### Pose, expression et action

Avec un character LoRA, décrire explicitement la pose et l'expression en langage naturel : « looking over her shoulder with a confident smile », « leaning against a wall, arms crossed, thoughtful expression », « mid-laugh with hands in motion ». La qualité des résultats dépend directement de la **diversité du dataset d'entraînement** : si le LoRA a été entraîné avec des angles variés (face, profil, 3/4), des expressions variées, et des distances variées (headshot, buste, plein pied), les prompts auront beaucoup plus de latitude créative. **25-30 images de haute qualité suffisent** pour Flux (contre 70-200 pour SD 1.5).

---

## 4. Vocabulaire de style pour des portraits LinkedIn créatifs et colorés

L'objectif est un rendu **vibrant, éditorial, semi-réaliste** — ni corporate froid, ni cartoon. Voici les registres de vocabulaire que Flux interprète le mieux, organisés par dimension.

### Styles artistiques que Flux maîtrise

Pour le contexte LinkedIn créatif, les combinaisons les plus efficaces sont : « vibrant editorial photography » (rendu magazine haut de gamme avec couleurs saturées), « mixed media artwork blended with photorealistic skin texture » (effet illustration-photo hybride très distinctif), « colorful lifestyle photography with artistic flair » (entre la photo et l'art), et « semi-realistic creative portrait with bold color palette » (le sweet spot pour du LinkedIn non-corporate).

Les mouvements artistiques que Flux rend particulièrement bien incluent l'Art Nouveau (idéal pour des éléments décoratifs en arrière-plan), le Pop Art (couleurs vives, impact visuel fort), et l'Expressionnisme (formes expressives, couleurs vibrantes). Pour un style plus photographique mais distinctif, les références « cross-processed Ektachrome » ou « Kodak Portra 400 with boosted saturation » ajoutent du caractère sans basculer dans l'illustration.

### Éclairage : décrire le comportement, pas le label

Flux réagit beaucoup mieux à une description de ce que fait la lumière qu'à un simple nom de technique. « Warm golden light streaming from the left, creating soft shadows across her face and gentle rim light on her hair » surpasse largement « golden hour lighting ». Pour les portraits LinkedIn créatifs, trois setups fonctionnent particulièrement bien : la lumière naturelle de fenêtre latérale (chaleur, authenticité), le split lighting dramatique avec couleurs (impact, disruption), et le backlight golden hour (aspiration, storytelling).

### Palettes de couleurs à intégrer dans les prompts

Pour une marque de coaching neurodiversité, la palette recommandée s'articule autour de **violet/purple** (créativité, sagesse, unicité), **teal/turquoise** (clarté, énergie calme), **or/jaune chaud** (optimisme, confiance), et **coral/rose chaud** (empathie, connexion). Flux comprend les descriptions de palette : « rich purple and warm gold color palette with teal accents » fonctionne directement. Flux 2 supporte même les codes HEX (`hex #C4725A warm terracotta`), mais pour Flux 1 dev, les descriptions naturelles sont préférables.

### Fonds adaptés au professionnel-créatif

- Workspaces créatifs : « modern creative studio with colorful artwork on walls »
- Environnements naturels : « sunlit botanical garden with lush green plants »
- Abstraits : « soft gradient background transitioning from deep purple to warm gold »
- Urbains-artistiques : « weathered brick wall with traces of colorful street art »
- Minimalistes-vivants : « clean background with floating geometric shapes in vibrant colors »

L'erreur à éviter : ne pas spécifier « white background » avec Flux dev, car cela provoque un bug de flou documenté.

---

## 5. Mapper le type de post LinkedIn à l'angle visuel

Chaque catégorie de post LinkedIn appelle un traitement visuel distinct. Voici la matrice complète pour les quatre types principaux.

**Post d'expertise/autorité** : composition confiante, regard caméra ou geste vers l'audience, éclairage structuré, couleurs profondes (navy, purple, teal) avec accents vibrants. Arrière-plan : espace de travail, scène de conférence, bibliothèque moderne. Métaphores : phare, boussole, sommet de montagne, échiquier.

**Post d'humanisation/authenticité** : pose détendue, sourire naturel, cadrage rapproché, lumière chaude et douce. Couleurs : coral, jaune doré, rose chaud. Arrière-plan : café cosy, jardin, bureau avec plantes. Métaphores : porte ouverte, lumière à travers une fenêtre, jardin en croissance.

**Post de disruption/différenciation** : composition dynamique, angles audacieux, contraste fort. Couleurs : combinaisons complémentaires à fort contraste (purple + jaune, teal + orange). Arrière-plan : formes géométriques abstraites, éléments surréalistes, pop-art. Métaphores : chaînes brisées, nager à contre-courant, kaléidoscope.

**Post storytelling/transformation** : rendu cinématique, éclairage atmosphérique, sens du mouvement et du parcours. Couleurs : dégradés chauds (ambre → purple). Arrière-plan : chemins, ponts, portes s'ouvrant sur la lumière. Métaphores : routes divergentes, phoenix, saisons changeantes.

### Métaphores visuelles pour la neurodiversité

Pour une audience ADHD/HPI, privilégier des métaphores d'**empowerment** (jamais de déficit) : le prisme qui décompose la lumière en arc-en-ciel (cerveau qui voit plus de couleurs), les constellations (connexions uniques), le moteur Ferrari (puissance brute à canaliser), le cerveau avec des éclairs colorés (énergie créative), les spirales et chemins non-linéaires (pensée divergente comme force).

---

## 6. Instructions GPT complètes : le système prêt à intégrer

Voici le bloc d'instructions à copier dans la section « Instructions » d'un GPT custom. Il analyse automatiquement chaque post LinkedIn et génère 2-3 variantes de prompts Flux LoRA adaptés.

```markdown
# RÔLE : Générateur de prompts visuels Flux LoRA pour LinkedIn

Tu es un directeur artistique spécialisé en création d'images IA pour LinkedIn. 
Tu analyses chaque post LinkedIn qu'on te soumet et tu génères 2-3 prompts 
optimisés pour Flux LoRA (character LoRA portrait).

## CONTEXTE MARQUE
- Trigger word du LoRA : [À REMPLACER PAR TON TRIGGER WORD]
- Description du sujet : [À REMPLACER : ex. "a woman in her 40s with curly 
  dark hair and warm brown eyes"]
- Palette de marque : purple (créativité), teal (clarté), gold (optimisme), 
  coral (empathie)
- Style : créatif, coloré, chaleureux, professionnel mais non-corporate
- Audience : entrepreneurs neuroatypiques (HPI/ADHD)
- Format cible : 4:5 portrait (1080×1350 px) pour mobile LinkedIn

## RÈGLES FLUX IMPÉRATIVES
1. Écrire TOUS les prompts en anglais — zéro mot français, même isolé ("Autodérision", "Cinématique", etc.)
2. Utiliser du langage naturel descriptif, PAS des listes de tags
3. Front-loader : trigger word exact du LoRA en PREMIER — jamais le prénom de la personne
4. Longueur : **40-70 mots par prompt** — compter les mots avant livraison ; si >70, couper (sacrifier les qualificatifs redondants en priorité)
5. JAMAIS de negative prompt ni de syntaxe de poids (word:1.5)
6. JAMAIS « white background » (bug Flux dev)
7. Décrire la lumière comme un COMPORTEMENT, pas un label
8. Inclure des specs caméra RÉELLES pour ancrer le rendu (85mm, f/1.8, Hasselblad X2D, Fujifilm X-T5…) — éviter les termes inventés comme "neon-autobiographical lighting"
9. Utiliser des formulations POSITIVES uniquement
   ("sharp focus" pas "no blur")
10. JAMAIS de métadonnées dans le prompt : "for the LoRA", "LoRA identity confirmed", "showcasing her features" → Flux interprète tout littéralement

## PROCESSUS D'ANALYSE

### Étape 1 : Lecture du post
Identifier : message central, ton émotionnel, émotion cible pour le lecteur, 
métaphores/concepts clés, thèmes neurodiversité éventuels.

### Étape 2 : Classification du post
Assigner UN type principal :
- EXPERTISE → frameworks, méthodes, données, conseils, how-to
- HUMANISATION → histoire personnelle, vulnérabilité, coulisses, leçon
- DISRUPTION → remise en question, prise de position contraire, recadrage
- STORYTELLING → arc narratif, transformation, avant/après, succès client
- NEURO_IDENTITÉ → directement sur ADHD, HPI, neurodiversité

### Étape 3 : Sélection des paramètres visuels

**EXPERTISE :**
- Cadrage : medium shot ou portrait 3/4
- Action : pose confiante (parler, montrer, écrire au tableau)
- Style : "vibrant editorial photography"
- Lumière : "bright natural light with dramatic accents"
- Couleurs : deep purple + teal + gold
- Fond : creative workspace, scène de conf, bibliothèque moderne
- Specs : "85mm f/2.8, professional photography, sharp focus"

**HUMANISATION :**
- Cadrage : close-up ou medium shot candide
- Action : sourire chaleureux, café en main, rire, contemplation
- Style : "warm lifestyle photography"  
- Lumière : "soft golden hour light" ou "warm window light"
- Couleurs : coral + warm yellow + soft purple
- Fond : café cosy, jardin, bureau avec plantes
- Specs : "50mm f/1.4, shallow depth of field, natural photography"

**DISRUPTION :**
- Cadrage : angle dynamique ou portrait bold
- Action : geste audacieux, bras croisés avec sourire complice
- Style : "high-contrast creative photography" ou "pop-art inspired"
- Lumière : "dramatic split lighting" ou "bold colored lighting"
- Couleurs : high-contrast purple + electric yellow, teal + coral
- Fond : formes géométriques abstraites, blocs de couleur
- Specs : "wide angle, dramatic composition, vivid saturated colors"

**STORYTELLING :**
- Cadrage : plan cinématique large ou portrait atmosphérique  
- Action : marcher vers l'avant, regarder l'horizon, en mouvement
- Style : "cinematic photography"
- Lumière : "golden hour dramatic lighting"
- Couleurs : warm gradient amber → purple
- Fond : chemin, pont, porte ouverte sur la lumière
- Specs : "35mm lens, cinematic composition, atmospheric depth"

**NEURO_IDENTITÉ :**
- Cadrage : portrait créatif ou medium shot conceptuel
- Action : entourée de métaphore visuelle, mains créant/construisant
- Style : "colorful creative photography with surreal elements"
- Lumière : "bright, multi-colored creative lighting"
- Couleurs : spectre prismatique, purple + tous les accents
- Fond : patterns neuronaux abstraits, spirales colorées, prisme
- Specs : "creative composition, vibrant colors, sharp detail"

### Étape 4 : Génération des variantes

Produire EXACTEMENT 3 variantes nommées **A, B, C** (ne pas renommer — "L'Architecte", "La Pilote" ne sont pas des noms de variantes valides) :

**Variante A — Portrait réaliste :**
Prompt centré sur le sujet dans une mise en scène cohérente avec le post.
Structure : trigger_word + cadrage + description sujet + action + style +
lumière + couleurs + fond + specs

**Variante B — Métaphore conceptuelle :**
Le sujet interagit avec une métaphore visuelle tirée du contenu du post.
Structure : trigger_word + scène conceptuelle décrivant la métaphore +
style + lumière + couleurs + mood

**Variante C — Artistique/abstraite :**
Traduction purement visuelle/émotionnelle du message, style illustration.
Structure : description de scène abstraite + style artistique + palette +
composition + ton émotionnel (trigger_word optionnel)

### Étape 5 : Format de sortie

⚠️ **Ce format est OBLIGATOIRE et non-négociable.** Toute variante livrée sans les 4 paramètres (Format, Guidance, LoRA strength, Pourquoi) est incomplète.

Pour chaque variante :
🎨 **[Variante A/B/C] — [description courte]**
> [Le prompt Flux complet, 100% en anglais, 40-70 mots — compter les mots]
- Format : 4:5 portrait (ou 1:1 si carousel)
- Guidance : [2.5-4.0 selon le style]
- LoRA strength : [0.7-0.9]
- Pourquoi : [1 phrase expliquant le choix VISUEL — ne pas référencer le guide de style Jennifer Perrault ici, expliquer pourquoi cette image sert le post]
```

---

## 7. Exemples de prompts complets prêts à l'emploi

Voici cinq exemples finaux couvrant les cas d'usage principaux, avec le placeholder `MALORACOACH` comme trigger word.

### Post d'expertise : « Les 3 piliers d'une routine ADHD-friendly »

> **Variante A — Portrait réaliste :**
> `MALORACOACH, three-quarter portrait of a confident woman with curly dark hair gesturing with her hands while explaining a concept, vibrant editorial photography, bright natural side light with warm golden accents creating soft shadows, rich purple and teal color palette, modern creative workspace with colorful diagrams on a glass board behind her, conveying knowledge and warmth, shot on Canon EOS R5 85mm f/2.8 sharp focus`

> **Variante B — Métaphore conceptuelle :**
> `MALORACOACH, medium shot of a woman with curly dark hair standing beside three luminous pillars of light in purple gold and teal, each pillar radiating gentle warmth, colorful creative photography with surreal architectural elements, soft dramatic lighting from within the pillars casting prismatic reflections, feeling of structure and clarity emerging from creative energy`

### Post d'humanisation : « Le jour où j'ai failli tout abandonner »

> **Variante A — Portrait réaliste :**
> `MALORACOACH, intimate close-up portrait of a woman with curly dark hair sitting in a cozy café holding a warm cup of coffee with both hands, gentle contemplative smile, warm lifestyle photography, soft golden window light from the left creating a warm glow on her face, coral and warm amber tones, blurred café background with soft bokeh, shot on Fujifilm X-T5 50mm f/1.4 natural light`

> **Variante B — Métaphore conceptuelle :**
> `MALORACOACH, cinematic shot of a woman with curly dark hair standing at a crossroads in a sunlit forest path, one path dark and overgrown the other bathed in warm golden light, atmospheric photography, golden hour rays filtering through tall trees, purple shadows transitioning to warm amber light ahead, feeling of choosing courage over fear, 35mm cinematic lens`

### Post de disruption : « Arrêtez de copier les stratégies des neurotypiques »

> **Variante A — Portrait bold :**
> `MALORACOACH, dynamic portrait of a woman with curly dark hair looking directly at camera with bold knowing smirk and arms confidently crossed, high-contrast creative photography, dramatic split lighting with purple light from one side and electric gold from the other, vibrant teal and coral abstract geometric shapes in background, feeling of playful defiance and confidence, wide angle dramatic composition vivid colors`

### Post neurodiversité : « Mon cerveau ADHD voit des connexions invisibles »

> **Variante C — Artistique :**
> `A luminous constellation map made of interconnected glowing nodes in purple gold and teal, some connections taking unexpected spiral paths that form beautiful complex patterns, bright prismatic energy radiating from key nodes, mixed media digital illustration blending astronomical charts with neural network imagery, dark indigo background with vibrant bioluminescent elements, conveying the beauty of a differently-wired mind seeing hidden patterns`

---

## 8. Paramètres de génération recommandés : récapitulatif

Pour chaque image générée, voici les réglages à appliquer (Flux dev avec character LoRA) :

| Paramètre | Réglage recommandé | Notes |
|-----------|-------------------|-------|
| **Distilled Guidance** | 3.0-3.5 | Baisser à 2.5 pour plus de réalisme, monter à 4.0 pour plus de stylisation |
| **CFG Scale** | 1 | Toujours 1 avec Flux, c'est le distilled guidance qui compte |
| **Steps** | 25-30 | 28 est un bon défaut ; monter à 35 pour plus de détail |
| **Sampler** | Euler | Le plus stable et rapide pour Flux |
| **Scheduler** | Beta | Couleurs plus chaudes que Simple — idéal pour portraits colorés |
| **LoRA Strength** | 0.8 | Baisser à 0.7 pour plus de liberté créative, monter à 0.9 pour plus de ressemblance |
| **Résolution LinkedIn 4:5** | 864×1080 ou 832×1040 | Multiples de 32, ratio 4:5 |
| **Résolution LinkedIn 1:1** | 1024×1024 | Pour les carrousels |

---

## 9. Ressources essentielles pour aller plus loin

La documentation officielle de Black Forest Labs se trouve sur **docs.bfl.ml/guides/prompting_summary** — c'est la source de référence qui couvre le framework Subject + Action + Style + Context. Le repo GitHub **black-forest-labs/skills** offre des guidelines installables, y compris comme plugin Claude Code.

Sur **Civitai**, trois articles font autorité : le « Flux Complete LoRA Settings and Dataset Guide » (post-mortem de 16+ LoRA), le guide « Photorealistic Characters using Dual Prompt Split Sigma Workflow » (technique avancée ComfyUI), et le « Quickstart Guide to Flux.1 » dans la section éducation. Pour l'exploration de styles, le **Flux Style Test Gallery** (enragedantelope.github.io) permet de visualiser comment Flux interprète différents mouvements artistiques.

Côté outils, le **FLUX-Prompt-Generator** de gokaygokay sur HuggingFace Spaces est l'outil le plus complet — il intègre des centaines de catégories et peut utiliser GPT/Claude/Groq pour enrichir les prompts. Le **Flux-Prompt-Enhance** (même auteur) est un modèle T5 qui transforme un prompt court en prompt détaillé optimisé pour Flux, intégrable dans ComfyUI via le node ComfyUI-Fluxpromptenhancer.

Les communautés les plus actives sont **r/FluxAI** et **r/StableDiffusion** sur Reddit, le serveur Discord **SECourses** (Dr. Furkan Gözükara) pour les tutoriels approfondis, et le blog **apatero.com** pour des guides détaillés en français et anglais.

---

## Conclusion : trois principes à retenir

Ce guide révèle un changement de paradigme par rapport à l'ère Stable Diffusion. **Premier principe** : penser en narrateur, pas en tagueur. Flux comprend « une femme confiante qui explique un concept en gesticulant, baignée dans une lumière dorée latérale » infiniment mieux qu'une liste de mots-clés. **Deuxième principe** : le contrôle se fait par le langage et les paramètres techniques (guidance, LoRA strength), jamais par la syntaxe de prompt — les poids, emphases et negative prompts appartiennent au passé. **Troisième principe** : la cohérence visuelle sur LinkedIn vient de la constance du template et de la palette, pas de la rigidité du prompt. En utilisant les instructions GPT ci-dessus avec des variables stables (trigger word, palette de marque, specs caméra préférées) et un système de classification des posts, chaque image sera à la fois unique et reconnaissable — exactement ce qu'il faut pour construire une identité visuelle forte sur LinkedIn.