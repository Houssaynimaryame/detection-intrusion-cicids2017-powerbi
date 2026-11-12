# Guide Power BI

## 1. Import
*Obtenir les données → Texte/CSV* : charger les fichiers de `results/` (encodage UTF-8, séparateur virgule).

| Table | Rôle |
|---|---|
| `attack_summary` | **table de référence** des catégories (1 ligne par catégorie) |
| `daily_summary`, `port_summary`, `traffic_stats`, `label_summary` | analyses du trafic |
| `data_quality` | bilan du nettoyage |
| `model_results`, `per_class_metrics`, `confusion_matrix`, `binary_detection`, `split_summary` | performance ML |
| `feature_importance` | importance des features |
| `leakage_robustness_check`, `leakage_per_class_recall` | robustesse |

## 2. Modèle de données
Vue *Modèle* : relations **1 → plusieurs** (sens unique) depuis `attack_summary[Attack_Category]` vers `daily_summary`, `port_summary`, `traffic_stats` et `label_summary` (même colonne). Les tables ML restent indépendantes (elles n'ont que 7 classes, sans Infiltration/Heartbleed).

Réglages à faire :
- `daily_summary[Day]` → *Trier par colonne* → `Day_Order`
- `port_summary[Destination Port]` → type **Texte** (sinon Power BI l'additionne)
- `confusion_matrix[Pct_of_Actual]`, `binary_detection[*rate*]` → format pourcentage (valeurs déjà en %/fractions : vérifier l'échelle)

## 3. Mesures DAX (table `_Mesures`, créée via *Entrer des données* vide)
```DAX
Total Flux = SUM(attack_summary[Count])
Flux Attaque = CALCULATE([Total Flux], attack_summary[Attack_Category] <> "Benign")
% Attaques = DIVIDE([Flux Attaque], [Total Flux])
% Conservé = CALCULATE(MAX(data_quality[Value]), data_quality[Metric] = "% conservé")
Meilleur Macro F1 = MAX(model_results[Macro_F1])
Taux de détection = MAX(binary_detection[Detection_rate_recall])
Taux fausses alertes = MAX(binary_detection[False_alarm_rate])
```

## 4. Pages
**Page 1 — Overview** : cartes (`Total Flux`, `% Attaques`, `% Conservé`) · histogramme des catégories (**échelle logarithmique**, sinon Heartbleed/Infiltration sont invisibles) · segment `Attack_Category`.

**Page 2 — Analyse des attaques** : histogramme empilé *Jour × catégorie* (`daily_summary`) · barres *top ports* (`port_summary`, légende = catégorie) · tableau `traffic_stats` (durée, paquets, octets moyens par catégorie). Message clé : chaque type d'attaque correspond à un jour de capture.

**Page 3 — Performance ML** : cartes (`Meilleur Macro F1`, `Taux de détection`, `Taux fausses alertes`) · barres *Macro F1 par modèle* · **matrice** recall par classe × modèle (`per_class_metrics`, mise en forme conditionnelle) · **matrice de confusion** (`Actual` en lignes, `Predicted` en colonnes, valeur = `Pct_of_Actual`, dégradé de couleur).

**Page 4 — Robustesse & features** : barres *Macro F1 par scénario* (`leakage_robustness_check`) — la page qui montre ta rigueur · top 20 `feature_importance`.

## 5. Conseils
- Titres qui racontent le message (« Un split aléatoire gonfle le Macro F1 de 13 points ») plutôt que « Macro F1 par scénario ».
- Même palette partout : Benign en gris, attaques en couleurs.
- Exporter chaque page en image (*Fichier → Exporter → PDF*, ou capture d'écran) vers `images/dashboard_*.png`.
