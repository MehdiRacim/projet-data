# Le massif landais devient-il méditerranéen ?

Projet de dataviz (Ynov M2 IA/Data), données ouvertes Météo-France / data.gouv.fr.

## Dashboard

```bash
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
.venv\Scripts\python app/app.py      # puis ouvrir http://127.0.0.1:8050
```

Dash + Plotly, **écran unique** (tient en 1920 × 1080 comme en 1366 × 768, sans défilement).
Deux sélecteurs comparent une station landaise à une station méditerranéenne ; les
4 panneaux se mettent à jour, chacun avec son verdict (« Méditerranéen ? Oui / En partie / Non »),
un titre qui énonce la conclusion et le graphique qui la prouve :

1. **Köppen** : pluie du mois d'été le plus sec (normales glissantes de 30 ans) face au seuil de 40 mm.
2. **Gaussen** : diagramme ombrothermique 1996-2025 des deux stations.
3. **Évolution** : mois secs par an depuis 1950 (moyenne mobile 11 ans).
4. **Feux** : surface brûlée par an dans les départements des deux stations (échelle log).

Bleu = Landes, orange = Méditerranée (paire validée pour les daltoniens).
Le bouton « Sources & méthode » rappelle les données, les critères et les limites.

| Fichier | Rôle |
|---|---|
| `app/app.py` | mise en page et interactions |
| `app/figures.py` | graphiques et calcul des verdicts |
| `app/donnees.py` | chargement de `data/processed/` |
| `app/assets/style.css` | mise en page écran unique |

## Pipeline climat

```bash
pip install -r requirements.txt
python pipeline/telecharger.py        # climat mensuel Météo-France → data/raw/
python pipeline/indicateurs.py        # indicateurs climat → data/processed/
python pipeline/telecharger_feux.py   # feux de forêt BDIFF → data/raw/bdiff/
python pipeline/indicateurs_feux.py   # indicateurs feux → data/processed/
```

Source : [Données climatologiques de base – mensuelles](https://www.data.gouv.fr/datasets/donnees-climatologiques-de-base-mensuelles)
(départements 40, 33, 66, 34, 13 ; 1950 → 2026).

### Stations retenues (séries complètes 1950-2026)

| Groupe | Stations |
|---|---|
| Massif landais | Mont-de-Marsan, Cazaux, Sauternes, Bordeaux-Mérignac |
| Témoins méditerranéens | Perpignan, Montpellier-Aéroport, Marignane |

Dax et Soorts-Hossegor ont été écartées : leurs séries ont des trous.

### Fichiers produits (`data/processed/`)

| Fichier | Grain | Contenu |
|---|---|---|
| `stations.csv` | station | nom, groupe, lat/lon, altitude |
| `mensuel.csv` | station × mois | RR (mm), TM (°C), `mois_sec` (Gaussen) |
| `annuel.csv` | station × année | Gaussen (`n_mois_secs`, `mois_secs`, `deficit_gaussen`), critères Köppen de l'année |
| `normales_30.csv` | station × fenêtre de 30 ans (`debut`–`fin`) | Köppen et Gaussen sur la climatologie moyenne |

### Définitions

- **Gaussen** : un mois est sec si P ≤ 2T. `deficit_gaussen` = Σ(2T − P) sur les mois secs.
- **Köppen** (Peel et al., 2007) : été = avril-septembre. **Cs** (méditerranéen) si
  le mois d'été le plus sec reçoit < 40 mm **et** moins d'1/3 du mois d'hiver le plus humide.
  `p_ete_min`, `p_hiver_max` et `ratio_ete_hiver` donnent la distance au seuil.

### ⚠️ Limite méthodologique

Köppen se calcule sur une **normale de 30 ans**. Calculé sur une seule année
(`koppen_annuel`, `cs_annuel`), il classe les Landes en « Cs » 80 à 100 % des années,
autant que Perpignan : le critère ne discrimine plus rien. Pour le dashboard, utiliser
`normales_30.csv` (une normale glissante par année de fin) pour la classe, et
`annuel.csv` pour les indicateurs continus (mois secs, `p_ete_min`).

## Pipeline feux de forêt

Source : [Base de Données sur les Incendies de Forêts en France (BDIFF)](https://www.data.gouv.fr/datasets/base-de-donnees-sur-les-incendies-de-forets-en-france-bdiff)
(IGN / ministère de l'Agriculture, Licence Ouverte v2).

La fiche data.gouv.fr ne propose pas de fichier : sa seule ressource renvoie vers le
[portail BDIFF](https://bdiff.agriculture.gouv.fr/incendies). `telecharger_feux.py`
reproduit la recherche du portail puis son export « CSV ». Un export est limité à
30 000 lignes : on découpe par zone (Aire méditerranéenne, Aquitaine/GIP ATGeRi) et
par période (2006-2015, 2016-2025). Total : 42 230 feux, 27 départements.

| Groupe | Départements |
|---|---|
| Massif landais | Gironde (33), Landes (40), Lot-et-Garonne (47) |
| Témoins méditerranéens | Pyrénées-Orientales (66), Hérault (34), Bouches-du-Rhône (13) |

Les autres départements des deux zones sont gardés avec `groupe = Autre`.

### Fichiers produits (`data/processed/`)

| Fichier | Grain | Contenu |
|---|---|---|
| `feux.csv` | un feu | date d'alerte, commune, surface (ha), surface forêt (ha), cause |
| `feux_annuel.csv` | département × année | nombre de feux, surface brûlée, feux ≥ 1 ha, grands feux (≥ 100 ha), plus grand feu, part estivale, saison des feux |
| `feux_mensuel.csv` | département × année × mois | nombre de feux, surface brûlée |
| `feux_lacunes.csv` | département × année | trous signalés par la BDIFF (Pyrénées-Atlantiques 2010-2011) |

`saison_longueur_jours` = écart entre le 10e et le 90e centile des dates de départ
(fenêtre qui contient 80 % des feux).

### ⚠️ Limites

- Une seule année domine tout : 2022 en Gironde (Landiras, La Teste), environ 28 700 ha,
  contre quelques centaines d'hectares les autres années. Utiliser des médianes ou une
  échelle log, et le montrer comme un événement extrême.
- Les pratiques de saisie diffèrent entre zones : l'aire méditerranéenne recense davantage
  de petits feux. Comparer de préférence les feux ≥ 1 ha.
- Surfaces brutes : pas encore rapportées à la surface forestière de chaque département.
