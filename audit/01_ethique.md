# Audit — ethique (À COMPLÉTER)
> Cf. procedure_audit.md (section correspondante). Remplis ce volet, puis synthétise dans rapport_audit.md.

## 1. Variables sensibles
 
**Directe :**
- `sexe` — utilisée **explicitement en feature** dans `legacy/train.py` (`X["sexe_bin"] = (df["sexe"] == "M").astype(int)`), sans justification clinique documentée. Variable sensible en feature non justifiée.
**Indirectes potentielles (hypothèses à vérifier, pas des conclusions) :**
- `departement` (ex. 44, 85 dans l'extrait fourni) — proxy géographique possible
- `age` — possible facteur clinique légitime (l'âge influence la DMS pour des raisons médicales réelles)

## 2. Disparate impact — chiffré puis investigué
 
**Résultats (notebook `mesures.ipynb`, dataset complet) :**
 
| Indicateur | Valeur |
|---|---|
| DI F/M — prédictions | **0.291** |
| DI F/M — étiquettes | **0.653** |
| Seuil `dms_jours` retenu comme référence (identique F/M) | 5.6 jours |
| FNR / FPR étiquette vs référence — F | 0.363 / 0.000 |
| FNR / FPR étiquette vs référence — M | 0.000 / 0.000 |
| FNR / FPR modèle vs référence — F | 0.737 / 0.018 |
| FNR / FPR modèle vs référence — M | 0.242 / 0.223 |
| Proba moyenne prédite vs taux réel — F | 0.321 vs 0.504 |
| Proba moyenne prédite vs taux réel — M | 0.491 vs 0.491 |
 
**Investigation:**
 
- **Les deux DI sont sous le repère conventionnel de 0.80.** Mais surtout, le DI du modèle (0.291) est **plus éloigné de 1** que celui des étiquettes (0.653) : `|0.291−1| = 0.709 > |0.653−1| = 0.347`. **Le modèle amplifie un écart déjà présent dans les données d'entraînement**.
- **Le seuil de `dms_jours` associé à un séjour « prolongé » est identique pour les deux groupes (5.6 jours)** : pas de biais de méthode — la comparaison F/M est sur une base commune.
- **Le taux réel de séjours prolongés (référence) est quasi identique entre les groupes : 50.4 % chez les femmes, 49.1 % chez les hommes.** la disparité observée n'est pas explicable par une différence de réalité clinique. Le DI bas pointe ici vers un problème de détection.
- **L'étiquette historique sous-détecte déjà les séjours longs chez les femmes** : FNR étiquette = 0.363 chez les femmes contre 0.000 chez les hommes (FPR nul des deux côtés). Autrement dit, **avant même le modèle**, l'étiquetage manque plus d'un tiers des séjours réellement prolongés chez les femmes, alors qu'il est exact chez les hommes. C'est un biais d'étiquetage.
- **Le modèle hérite de ce biais et l'aggrave** : FNR modèle = 0.737 chez les femmes (73.7 % des séjours réellement prolongés ne sont pas signalés) contre 0.242 chez les hommes. À l'inverse, le FPR est plus élevé chez les hommes (0.223 vs 0.018) : le modèle sur-signale les hommes et sous-signale massivement les femmes.
- **La calibration confirme un problème spécifique aux femmes** : la probabilité moyenne prédite est bien calée sur le taux réel chez les hommes (0.491 vs 0.491) mais très en-dessous chez les femmes (0.321 prédit vs 0.504 réel).

**Qui est lésé, par quelle erreur, sous quelle hypothèse d'usage :**
- **Constat robuste, indépendant de l'hypothèse d'usage :** à réalité clinique quasi identique (taux réels 50.4 % vs 49.1 %), le modèle repère 3 fois moins souvent les séjours à risque de prolongation chez les femmes que chez les hommes (FNR 73.7 % vs 24.2 %).
- Le **sens du préjudice reste conditionné à l'usage réel** :
  - Si le signal déclenche un bénéfice (anticipation, coordination de sortie, réservation de lit) → **les patientes en sont privées 3 fois plus souvent que les patients**.
  - Si le signal expose à un désavantage → ce sont alors davantage les **hommes** qui en subissent le poids.
- Dans les deux lectures, il y a un **traitement différencié significatif et non expliqué par la réalité clinique mesurée**, ce qui suffit à documenter le constat même sans trancher l'usage.

## 3. RGPD santé
 
**Art. 9**
- S'applique de facto : le dataset contient des données de santé (`nb_comorbidites`, `imc`, `dms_jours`, `sejour_prolonge`)
- Minimisation (art. 5) : `sexe` utilisée en feature sans justification clinique visible → à questionner conjointement avec le constat de la section 1. Ce point est renforcé par la section 2 : à taux réel de séjours prolongés quasi identique entre sexes (50.4 % / 49.1 %), l'usage de `sexe` en feature ne s'accompagne d'aucun bénéfice prédictif visible qui justifierait de conserver une variable sensible sans justification clinique documentée.
**Art. 22 — les 2 conditions examinées :**
- *Décision exclusivement automatisée ?* Le commentaire du code dans `predict.py` est explicite : « décision automatique sans supervision humaine, sans log, sans traçabilité ». 
- *Effet significatif ?* Une prédiction influençant la prise en charge, l'allocation de lit ou la sortie a un effet potentiellement significatif sur le patient
- **Conclusion :** l'art. 22 ne peut pas être écarté à ce stade — la question du rôle déterminant du score reste ouverte.

## 4. Usage réel du score
 
- **Qui lit le score :** indéterminé — `predict.py` ne produit qu'un `print()` console après appel SSH, aucune trace de destinataire
- **Quand :** non précisé dans le code fourni (à l'admission ? en cours de séjour ?)
- **Qu'est-ce que ça déclenche :** non documenté — aucune écriture en base, aucun appel en aval visible dans le code

## 5. AI Act
Rien dans le code ne suggère un marquage CE dispositif médical
**Conclusion documentée :** qualification **indéterminée, faute d'information sur l'usage réel**. Probablement pas haut risque au sens strict en l'état des informations disponibles (pas d'autorité publique, pas de triage urgence confirmé), mais à confirmer par la question ouverte sur l'usage réel.
 
## 6. Questions ouvertes (volet éthique)
 
- Le score `RISQUE_SEJOUR_PROLONGE` est-il consulté par un soignant qui décide ensuite, ou déclenche-t-il une action automatique (allocation de lit, alerte) ?
- Quel est le taux de désaccord entre le score et la décision clinique finale ? Est-il tracé quelque part ?
- L'usage de `sexe` comme variable prédictive repose-t-il sur une justification clinique documentée ?
- Le score conditionne-t-il, même indirectement, l'éligibilité à des soins ou un triage aux urgences ?
- Le modèle sous-détecte les séjours réellement prolongés chez les femmes 3 fois plus souvent que chez les hommes (FNR 73.7 % vs 24.2 %), à taux réel quasi identique entre groupes : cette disparité a-t-elle déjà été identifiée par MediVox ?
- L'étiquetage historique (`sejour_prolonge`) sous-compte déjà les séjours longs chez les femmes (FNR 0.363 vs 0.000 chez les hommes) avant même le modèle : une explication opérationnelle connue ? Un biais non identifié dans le processus de documentation ?
 