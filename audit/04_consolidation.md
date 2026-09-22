# Audit — consolidation

> Cf. `procedure_audit.md` (section 5). Synthèse des 3 volets (`01_ethique.md`, `02_technique.md`, `03_ressources.md`) en un tableau unique, hiérarchisé, destiné à alimenter `rapport_audit.md`.

## Méthode

Chaque ligne reprend un constat déjà chiffré dans un des 3 volets d'audit — rien n'est ajouté ici, ce tableau **synthétise et hiérarchise**, il n'introduit pas de nouveau constat.

**Légende :** 🔴 risque majeur (action jugée prioritaire par l'auditeur) · 🟠 risque significatif (à traiter à moyen terme) · 🟡 à surveiller (non urgent en l'état des informations disponibles)

**Répartition :** 5 🔴 · 5 🟠 · 5 🟡 (15 indicateurs)

## Tableau consolidé

| # | Indicateur | Volet | Sévérité | Conséquence client |
|---|---|---|---|---|
| 1 | Le modèle sous-détecte les séjours réellement prolongés chez les femmes 3× plus souvent que chez les hommes (FNR 73.7 % vs 24.2 %), alors que le taux réel est quasi identique entre groupes (50.4 % vs 49.1 %) — le modèle amplifie un biais déjà présent dans les étiquettes (DI 0.291 vs 0.653) | Éthique | 🔴 | Traitement différencié par sexe non justifié par les données cliniques — exposition à une contestation d'équité de soins |
| 2 | Base légale RGPD art. 9 (données de santé) non documentée dans le code fourni | Éthique | 🔴 | Traitement de données de santé potentiellement sans fondement juridique vérifié |
| 3 | Secret de connexion en clair dans le code source versionné (`DB_PASSWORD` dans `train.py`) | Technique | 🔴 | Accès non autorisé à la base de données possible pour quiconque consulte le dépôt |
| 4 | Le modèle et le service ne vivent que sur un serveur unique, déployé manuellement par `scp`, sans sauvegarde documentée (SPOF) | Technique | 🔴 | Arrêt total et immédiat du service de prédiction en cas de panne serveur, sans plan de reprise connu |
| 5 | Aucune journalisation des appels ni des résultats (`predict.py` n'affiche qu'un `print()` console) | Technique | 🔴 | Impossible de détecter un incident silencieux, et impossible de vérifier la conformité art. 22 |
| 6 | L'étiquetage historique (`sejour_prolonge`) sous-compte déjà les séjours longs chez les femmes avant même le modèle (FNR étiquette 36.3 % vs 0 %) | Éthique | 🟠 | Cause racine antérieure au modèle — un changement d'algorithme seul ne suffira pas à corriger le biais |
| 7 | Variable sensible `sexe` utilisée directement en feature, sans justification clinique documentée | Éthique | 🟠 | Écart de traitement difficile à défendre devant un contrôle de conformité |
| 8 | RGPD art. 22 : rôle déterminant du score indéterminé — impossible de mesurer le taux de désaccord score/décision clinique faute de logs | Éthique | 🟠 | Qualification juridique du traitement automatisé impossible à trancher en l'état |
| 9 | Aucune validation des entrées dans `predict.py` (types castés sans contrôle, pas de gestion d'erreur) | Technique | 🟠 | Plantage non explicite du script en production sur une entrée mal formée |
| 10 | Aucun pipeline CI/CD — déploiement historique entièrement manuel | Technique | 🟠 | Reprise après incident lente, sans garantie que le code en prod corresponde à une version tracée |
| 11 | Qualification AI Act (art. 6) indéterminée, faute d'information sur l'usage réel du score en aval | Éthique | 🟡 | Obligations réglementaires (si haut risque) non identifiables tant que l'usage n'est pas clarifié |
| 12 | Absence de versionning/metadata sur le modèle sérialisé (`.joblib`) | Technique | 🟡 | Impossible de savoir avec certitude quelle version du modèle tourne en production à un instant donné |
| 13 | Architecture non modulaire — feature engineering dupliqué entre `train.py` et `predict.py`, sans pipeline scikit-learn | Technique | 🟡 | Risque de désynchronisation silencieuse entre entraînement et inférence en cas de modification future |
| 14 | Rechargement complet du modèle à chaque appel, traitement strictement séquentiel — impact actuel non mesurable faute de connaître le volume réel de requêtes | Technique/Ressources | 🟡 | Coût actuellement négligeable dans l'absolu (5.1 Mo, quelques ms), mais non représentatif d'une charge de production inconnue |
| 15 | Une alternative plus légère existe (HistGradientBoosting, ~13,5× plus léger sur disque, perte de F1 indicatif limitée) mais un changement d'algorithme seul ne traiterait pas le biais du #1 (mêmes étiquettes historiques) | Ressources | 🟡 | Risque de "fausse solution" si un changement de modèle est décidé sans traiter la question du biais séparément |

## Constats transverses (reliant plusieurs volets)

- **#5 ↔ #8** : l'absence de journalisation (technique) est la cause directe de l'impossibilité de trancher la question juridique de l'art. 22 (éthique) — un seul chantier technique (logs) débloquerait une clarification de conformité
- **#1 ↔ #6** : le biais du modèle (🔴) trouve son origine dans un biais d'étiquetage antérieur (🟠) — utile pour éviter que le client ne conclue que corriger le modèle seul suffirait
- **#15 ↔ #1** : anticipe une question probable côté client (« on change juste de modèle ? ») en la rattachant explicitement au constat de biais, sans y répondre (hors périmètre)

## Ce que ce tableau ne fait pas

Conformément au mandat d'audit : aucune ligne ne propose de correction ni d'architecture cible. Chaque sévérité reflète un jugement d'auditeur sur l'urgence perçue à partir des chiffres disponibles — elle reste discutable et pourra être réévaluée une fois les questions ouvertes (listées dans chaque volet) tranchées par MediVox.