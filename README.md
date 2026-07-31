# Atelier Human Design

Application web autonome d'analyse Human Design. **Un seul fichier, aucune dépendance, aucun réseau.**

Le calcul astronomique s'exécute intégralement dans le navigateur : ni bibliothèque externe, ni API, ni requête sortante. Ouvrez `index.html` en double-cliquant — y compris hors connexion.

---

## Démarrer

```bash
open index.html
```

Modes disponibles par paramètre d'URL :

| Paramètre | Effet |
|---|---|
| `?test=1` | Exécute la suite de vérification (35 tests) et affiche le rapport |
| `?demo=1` | Pré-remplit le formulaire avec un jeu d'essai |
| `?debug=1` | Ouvre le journal de diagnostic (sinon silencieux, bufferisé) |
| `?source=remote` | Bascule sur un moteur de calcul distant (voir *Intégration*) |

Le panneau de diagnostic se bascule aussi au clavier : `Ctrl/Cmd + Maj + D`.

---

## Ce que fait l'application

### Moteur de calcul

Le Human Design se déduit de la position des astres à deux instants : la naissance (thème conscient) et le moment où le Soleil se trouvait 88° d'arc plus tôt (thème inconscient). Rien n'est simulé.

| Corps | Méthode | Précision |
|---|---|---|
| Soleil | Meeus ch. 25 | ~0,01° |
| Lune | Meeus ch. 47, série ELP tronquée (59 termes) | ~0,01° |
| Mercure → Pluton | Éléments képlériens JPL 1800–2050, Kepler par Newton-Raphson | 0,001° à 0,05° |
| Nœuds lunaires | Nœud vrai (osculateur), nœud moyen en option | — |

À comparer à la largeur d'une ligne du schéma : **0,9375°**. ΔT, nutation et précession sont appliqués.

L'instant du design est obtenu par résolution de Newton sur l'arc solaire exact, pas par une approximation à 88 jours.

**Le lieu de naissance ne sert qu'au fuseau horaire.** Les longitudes retenues sont géocentriques : le Human Design n'utilise ni maisons ni ascendant, et vos coordonnées précises ne changent rien au résultat. Les règles d'heure d'été historiques viennent de la base tz du navigateur, via `Intl` — aucune table à maintenir.

### Validation

Sur 600 naissances aléatoires entre 1900 et 2025, la répartition obtenue est :

| Type | Obtenu | Publié |
|---|---|---|
| Générateur | 34 % | ~35 % |
| Générateur Manifesteur | 33 % | ~33 % |
| Projecteur | 21 % | ~20 % |
| Manifesteur | 10 % | ~9 % |
| Réflecteur | 1,2 % | ~1 % |

Les 12 profils tombent exactement — aucune combinaison impossible (pas de 1/1, pas de 2/1), ce qui valide la roue Rave et l'arc de 88°.

### Interface

Six sections : Accueil, Fonctionnement, Analyse, Rapport, Historique, Export. Routage par `#hash`, bouton retour du navigateur fonctionnel.

- **Analyse** — parcours guidé en 5 étapes, sauvegarde locale, validation de champs, plus de 200 villes référencées et repli sur ~420 fuseaux IANA.
- **Rapport** — quatre vues : Synthèse (6 points en langage clair), Rapport complet (15 chapitres avec sommaire collant), Explorer (4 modules), Questions.
- **Historique** — recherche, filtre par type, 5 tris, réouverture, suppression, téléchargement par entrée.
- **Export** — HTML autonome, PDF (impression A4), JSON, TXT.

Un **indice de stabilité** recalcule le schéma aux bornes de l'incertitude horaire déclarée et indique lesquels du type, de l'autorité, du profil et de la définition résistent.

### Les quatre modules d'exploration

| Module | Contenu |
|---|---|
| **Calcul avancé** | Croix, définition, centres, canaux, 64 portes, longitudes écliptiques, logique de dérivation, résumé technique |
| **Lecture complète** | Interprétation développée : type, stratégie, autorité, profil, centres, canaux, non-soi, signature |
| **Intégration** | Architecture API, contrat de données, configuration éditable en direct, test de source |
| **Version multi-profil** | Lecture de couple, duo et équipe, calculée depuis l'historique |

Le module multi-profil **calcule** la théorie de connexion — électromagnétique, compagnonnage, dominance, compromis — puis les centres que le groupe définit ensemble et les angles morts partagés.

---

## Architecture

Trois couches strictement séparées : chacune ignore l'implémentation des deux autres.

```
Logique          Affichage              Stockage
─────────        ─────────              ────────
Astro            Render                 Store
Engine           Report                   ├── LocalDriver  (localStorage)
Synthesis        Menus                    └── RemoteDriver (HTTP)
Composite
```

### Brancher un moteur de calcul externe

Toute source doit répondre à un contrat unique, `CHART_SHAPE`. Deux points à toucher, pas un de plus :

```js
CONFIG.source     = 'remote';
CONFIG.apiBaseUrl = 'https://votre-api.example';
```

puis adapter `mapChartResponse(raw, input)`. Les champs absents de la réponse distante sont complétés par le moteur local — l'application ne peut pas se retrouver sans données. Le repli est automatique si la source distante échoue.

L'onglet **Intégration** permet de tester tout cela sans écrire une ligne de code.

### Brancher une base de données

`Store` expose une API asynchrone (`list`, `get`, `put`, `remove`, `clear`). Remplacer `LocalDriver` par `RemoteDriver` ne demande aucune modification ailleurs.

Chaque entrée d'historique conserve **les données d'entrée du moteur** en plus de la projection d'affichage. Le schéma étant déterministe, rouvrir un rapport le recalcule à l'identique : ~3 Ko par entrée au lieu de ~20 Ko, et déduplication automatique si la même naissance est ré-analysée.

### API publique

Exposée sur `window.HDAtelier` :

```js
renderReport(chart, opts)        saveAnalysisToHistory(chart)   prepareExport(format, chart)
renderHistory()                  loadHistory()                  downloadReport(format, chart)
toggleMenuSection(id)            deleteHistoryItem(id)          openHistoryItem(id)
```

Ainsi que les modules : `CONFIG`, `HD`, `Astro`, `Engine`, `Provider`, `Render`, `Report`, `Synthesis`, `Composite`, `Menus`, `Store`, `History`, `Export`, `Router`, `Tests`.

---

## Tests

```
open "index.html?test=1"
```

35 tests couvrant :

- **Intégrité des données** — 64 portes uniques, 36 canaux sans doublon, chaque porte rattachée à un centre et à un ancrage graphique, 12 profils
- **Géométrie de la roue** — porte 41 à 302°, découpage en 6 lignes, continuité sur 360°
- **Repères astronomiques** — Meeus 25.b (Soleil, 1992-10-13) et 47.a (Lune, 1992-04-12), équation du centre, vitesse diurne, mouvement planétaire sur un an
- **Fuseaux** — heure d'été française, aller-retour heure murale ⇄ UTC, validité des 200+ villes
- **Moteur** — dérivation du type et de l'autorité, hiérarchie des autorités, 100 naissances aléatoires
- **Rapport** — 15 sections, rendu sans valeur non résolue, synthèse, points d'attention
- **Composite** — exclusivité des 4 natures de connexion, conservation des canaux
- **Historique** — recalcul à l'identique depuis l'enregistrement, filtres et tris
- **Export** — 4 formats, document HTML sans ressource externe
- **Routeur et modules** — 6 sections déclarées, 4 modules rendus

---

## Limites connues

Elles sont réelles et assumées :

- **Croix d'incarnation** — le registre des 192 croix nommées (`HD.CROSS_REGISTRY`) est interrogé par le calcul mais vide. En attendant, la croix s'affiche par son angle et son quaternaire de portes : exact, jamais inventé.
- **Pluton** — précision d'environ 0,05°, ce qui peut occasionnellement faire basculer une ligne près d'une frontière.
- **PDF** — passe par la boîte d'impression du navigateur (« Enregistrer au format PDF »). Sans bibliothèque externe, il n'existe pas d'autre voie hors ligne. La feuille d'impression est optimisée : A4, marges 16/14/18 mm, fond clair, sections insécables.
- **Fuseaux antérieurs à 1900** — approximatifs, comme dans toutes les bases tz.

---

## Confidentialité

Aucune requête réseau n'est émise, à aucun moment. Les données de naissance et l'historique restent dans le stockage local du navigateur. Le fichier peut être enregistré et utilisé entièrement hors connexion.

---

## Publication web

Le fichier étant nommé `index.html`, activer GitHub Pages suffit à le servir tel quel :

```bash
gh api -X POST repos/auralinktherapeute/human-design-agent/pages \
  -f 'source[branch]=main' -f 'source[path]=/'
```
