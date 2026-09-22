# Audit — ressources
> Cf. procedure_audit.md (section correspondante). Remplis ce volet, puis synthétise dans rapport_audit.md.

## 1. Mesures psutil sur le modèle audité (RandomForest, `dms_predictor_v1.joblib`)
 
**Résultats (notebook `mesures.ipynb`) :**
 
| Indicateur | Valeur |
|---|---|
| RSS avant chargement | 256.5 Mo |
| RSS après chargement | 261.6 Mo |
| Delta chargement modèle | **5.1 Mo** |
| Taille du modèle sur disque | **4.96 Mo** |
| Temps d'inférence — 100 lignes | 30.68 ms total (0.3068 ms/ligne) |
| Temps d'inférence — 1 000 lignes | 5.97 ms total (0.0060 ms/ligne) |
| Temps d'inférence — 10 000 lignes | 31.82 ms total (0.0032 ms/ligne) |
 
**Lecture honnête (chiffrer, ne pas juger) :**
- Le coût **compute** du modèle audité est faible dans l'absolu : 5.1 Mo de mémoire pour charger le modèle, ~5 Mo sur disque, et un temps d'inférence de l'ordre de quelques millisecondes par lot
- **Anomalie à signaler, pas à masquer** : le temps par ligne **diminue** entre 100 lignes (0.307 ms/ligne) et 1 000/10 000 lignes (0.006 puis 0.003 ms/ligne), au lieu de rester stable ou d'augmenter. La mesure à 100 lignes inclut vraisemblablement un **coût de démarrage** qui n'est pas représentatif du coût marginal réel. Les mesures à 1 000 et 10 000 lignes, plus stables et cohérentes entre elles (0.006 et 0.003 ms/ligne), sont plus fiables pour estimer le coût réel par prédiction
- **Conclusion :** le coût d'inférence par appel individuel est négligeable une fois le modèle chargé — le vrai poids opérationnel est ailleurs (cf. section 3)

## 2. Comparaison à des alternatives plus sobres
 
**Alternatives retenues (conformes aux exemples du mini-cours) :**
- `LogisticRegression` (max_iter=1000)
- `HistGradientBoostingClassifier` (random_state=0)
Entraînées sur les mêmes features que le legacy (`age`, `nb_comorbidites`, `imc`, `sexe_bin`), mesurées avec le même protocole (RSS, temps d'inférence à 10 000 lignes, taille sur disque).
 
**Limite méthodologique assumée explicitement :** `legacy/train.py` n'effectue aucun split train/test — le F1 ci-dessous est mesuré sur les données d'entraînement pour les 3 modèles, à iso-protocole. C'est un indicateur **indicatif de comparaison relative entre les 3 modèles**, pas une mesure de performance réelle en généralisation, et un RandomForest profond peut afficher un bon F1 d'entraînement simplement parce qu'il sur-apprend, sans que cela ne prouve une meilleure performance en production.
 
**Résultats (notebook `mesures.ipynb`) :**
 
| Modèle | F1 (indicatif) | Temps inférence 10k (ms) | Taille disque (Mo) | Delta RSS (Mo) |
|---|---|---|---|---|
| RandomForest (legacy) | 0.683 | 34.28 | 4.956 | 0.000 |
| LogisticRegression | 0.577 | 7.85 | 0.001 | 0.057 |
| HistGradientBoosting | 0.640 | 70.52 | 0.366 | 0.000 |
 
**Lecture honnête (chiffrer, ne pas juger) :**
- **Taille sur disque** : LogisticRegression est environ **5 000 fois plus légère** que le RandomForest (0.001 Mo vs 4.956 Mo) ; HistGradientBoosting environ **13,5 fois plus léger** (0.366 Mo)
- **F1 indicatif** : l'écart avec le legacy est plus faible pour HistGradientBoosting (0.640, soit −0.043 point) que pour LogisticRegression (0.577, soit −0.106 point) — au prix indicatif de perte de performance, HistGradientBoosting semble un compromis plus proche du modèle actuel
- **Temps d'inférence à 10k lignes** : ici HistGradientBoosting (70.52 ms) est **plus lent** que le legacy (34.28 ms) — résultat probablement affecté par le même effet de démarrage/premier appel identifié en section 1, donc **à ne pas trancher sur cette seule mesure**
- **Un changement d'algorithme seul ne corrige pas le biais documenté dans le volet éthique** : les deux alternatives sont entraînées sur les **mêmes étiquettes historiques**, potentiellement biaisées (cf. `audit/01_ethique.md`, section 2) — un gain de sobriété ne remplace pas un traitement du biais, ce sont deux sujets distincts
**Argument de sobriété :** il existe au moins une alternative (HistGradientBoosting) sensiblement plus légère sur disque (~13,5×) pour une perte de F1 indicatif limitée (−0.043), et une alternative (LogisticRegression) extrêmement légère et rapide pour une perte de F1 indicatif plus marquée (−0.106).
 
## 3. Lecture sobriété — coût compute vs coût opérationnel
 
- Le coût **compute** pur de l'inférence (RSS, temps, taille du fichier) est mesuré ci-dessus et sera faible dans l'absolu
- Le vrai coût, mis en évidence dans le volet technique, est **opérationnel** : le modèle est rechargé intégralement à chaque appel SSH (pas de service persistant), ce qui multiplie le coût de chargement par le nombre d'appels — un chiffre de RSS/temps isolé sous-estime donc le coût réel en production si le volume d'appels est élevé (question ouverte du volet technique : volume actuel de prédictions/jour, non communiqué)

## 4. Questions ouvertes (volet ressources)
 
- Quel est le volume réel d'appels au script de prédiction par jour, pour resituer le coût mesuré ici (isolé) dans un contexte de charge réelle ?
- Un changement de modèle (vers une alternative plus légère) est-il envisagé indépendamment de la question de biais déjà relevée dans le volet éthique, ou les deux sujets seraient-ils traités ensemble en M7-B2 ?