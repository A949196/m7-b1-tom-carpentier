# Rapport d'audit — prédicteur de séjour prolongé v1 (MediVox)
 
> 2 lectorats : 👩‍💻 Hélène (directrice technique) · ⚖️ Marc (DPO).
 
## 1. Synthèse exécutive 
⚖️ Le risque le plus sérieux est un **biais actif et mesuré, pas seulement théorique** : le modèle rate 3 fois plus souvent les femmes qui font réellement un séjour prolongé que les hommes (73,7 % de cas manqués contre 24,2 %), alors que les deux groupes font en réalité un séjour long presque aussi souvent l'un que l'autre (50,4 % vs 49,1 %). Il s'agit d'une différence de détection. Ce biais existait déjà dans l'historique des données avant le modèle, et le modèle l'aggrave.
 
⚖️ À ce biais s'ajoute une **incertitude de conformité** : on ne sait pas quelle base légale couvre aujourd'hui ce traitement de données de santé, et l'absence totale de traçabilité empêche de vérifier si ce score influence réellement une décision médicale — ce qui bloque à la fois une question RGPD (art. 22) et la qualification sur l'AI Act.
 
👩‍💻 Sur le plan technique, **tout repose sur une seule machine**, déployée à la main, sans sauvegarde, sans suivi automatique des pannes et sans processus de déploiement fiable. Un mot de passe de connexion à la base de données est écrit en clair dans le code. Le coût de calcul du modèle lui-même est faible et n'est pas un sujet d'urgence.
 
👩‍💻 Une alternative plus légère existe, mais **elle ne réglerait pas le biais**.
 
## Glossaire rapide
 
*(défini une fois ici, réutilisé tel quel dans tout le rapport)*
 
| Terme technique | Ce que ça veut dire concrètement |
|---|---|
| Disparate impact | Rapport entre le taux de signalement d'un groupe et d'un autre. Sous 0,80, c'est un signal d'alerte — pas encore une conclusion |
| FNR (taux de manqués) | Part des cas réellement à risque que le modèle **ne signale pas** |
| FPR (taux de fausses alertes) | Part des cas **sans risque réel** que le modèle signale à tort |
| Calibration | Est-ce que le pourcentage de risque annoncé par le modèle correspond à la réalité observée ? Une calibration dégradée veut dire que les probabilités affichées ne sont plus fiables |
| SPOF | Un seul maillon dont la panne arrête tout le système |
| RSS | Mémoire réellement utilisée par le programme pendant son exécution |
 
## 2. Contexte et périmètre
 
MediVox Cliniques a déployé un modèle qui signale les séjours à risque de prolongation, développé par un prestataire aujourd'hui parti, sans documentation. Cet audit répond à une demande conjointe d'Hélène (directrice technique) et de Marc (DPO), avant toute décision sur l'avenir du système.
 
**Ce qui est audité :** le code existant (`legacy/predict.py`, `legacy/train.py`), le modèle entraîné et le dataset des séjours, sur 3 volets — éthique, technique, ressources.
 
**Ce qui est volontairement exclu de cet audit :**
- La correction du code : nous observons et documentons, nous ne modifions rien
- La proposition d'une nouvelle architecture
- Une analyse d'impact (AIPD) juridique complète
- La correction du biais détecté
- Un audit de sécurité offensif

## 3. Volet éthique ⚖️ (pour Marc)
 
**Variable en cause :** le sexe du patient, utilisée directement par le modèle pour prédire le risque, sans justification médicale documentée dans le code.
 
**Le signal d'alerte, puis la preuve :**
Le rapport entre femmes et hommes signalés par le modèle est de 0,29 (largement sous le seuil d'alerte de 0,80). Ce chiffre seul pourrait s'expliquer par une différence réelle entre les patients — mais ce n'est pas le cas ici : les femmes font en réalité un séjour prolongé presque aussi souvent que les hommes (50,4 % contre 49,1 %). La différence de signalement n'a donc **pas d'explication médicale dans les données observées**.
 
En creusant plus loin : le modèle manque 73,7 % des séjours réellement prolongés chez les femmes, contre 24,2 % chez les hommes. Ce biais n'est d'ailleurs pas créé par le modèle — il existait déjà dans la façon dont les séjours ont été historiquement enregistrés (36,3 % de séjours longs non enregistrés comme tels chez les femmes, contre 0 % chez les hommes) — mais **le modèle l'amplifie fortement** plutôt que de le corriger.
 
**Ce que cela veut dire concrètement, selon ce que déclenche le score :**
- Si être signalé apporte un bénéfice au patient (anticipation, organisation de sortie) → les patientes en sont privées 3 fois plus souvent
- Si être signalé expose à un désavantage (patient perçu comme complexe) → ce sont alors davantage les hommes qui le subissent, via un taux de fausses alertes bien plus élevé chez eux (22,3 % contre 1,8 %)
- Dans les deux cas, il y a un traitement différencié significatif, qui n'est pas expliqué par les données cliniques disponibles
**Droit applicable :**
- Les données traitées sont des données de santé — la base légale actuellement retenue par MediVox pour ce traitement **n'est documentée nulle part** dans ce qui nous a été fourni
- Sur la question d'une décision automatisée : le code indique qu'aucune supervision humaine, aucune traçabilité, n'entoure aujourd'hui ce calcul — et la jurisprudence européenne (CJUE, *SCHUFA*, 2023) rappelle qu'un score peut relever de cet article s'il joue un rôle déterminant dans une décision. **Nous ne pouvons pas trancher cette question sans traçabilité des décisions** — un point qui rejoint directement le constat technique de la section 4
- Sur le règlement européen sur l'IA (AI Act) : le niveau de risque du système dépend de ce que le score déclenche réellement, pas du simple fait qu'il touche à la santé.

## 4. Volet technique 👩‍💻 (pour Hélène)
 
**Architecture :** le code est un script unique sans découpage en fonctions ni en modules. La construction des données d'entrée est dupliquée entre l'entraînement et la prédiction — toute modification doit être répétée aux deux endroits, avec un risque d'oubli.
 
**Sécurité :** un mot de passe de connexion à la base de données est écrit en clair dans le code source versionné. Aucune vérification n'est faite sur les données reçues en entrée (le script plante sans message exploitable en cas d'erreur de saisie).
 
**Point de rupture unique (SPOF) :** le modèle et le service ne vivent que sur une seule machine, déployée manuellement, sans sauvegarde connue. Une panne de cette machine arrête intégralement le service. Il n'existe aucun processus automatisé de déploiement, ce qui ralentirait toute reprise après incident.
 
**Suivi et traçabilité :** le système n'enregistre aucune trace de son fonctionnement (pas de journal, « logs »). Concrètement, cela veut dire qu'une panne silencieuse est indétectable tant que personne ne signale l'absence de réponse — et que la question juridique de la section 3 (rôle du score dans la décision) ne peut pas être vérifiée faute de traçabilité.
 
**Scalabilité :** le modèle est rechargé intégralement à chaque appel, et les requêtes sont traitées une par une. L'impact réel de ceci dépend du volume de sollicitations actuel, que nous ne connaissons pas.
 
## 5. Volet ressources
 
**Coût de calcul du modèle actuel :** faible dans l'absolu — environ 5 Mo de mémoire supplémentaire au chargement, un fichier de 5 Mo sur disque, et un temps de réponse de l'ordre de quelques millièmes de seconde par prédiction une fois le modèle chargé.
 
**Comparaison à 2 alternatives plus légères :**
 
| Modèle | Fiabilité indicative | Rapidité (10 000 lignes) | Taille sur disque |
|---|---|---|---|
| Modèle actuel | référence | 34 ms | 5 Mo |
| Alternative légère (régression logistique) | en retrait | 8 ms | 0,001 Mo (~5000× plus léger) |
| Alternative intermédiaire (HistGradientBoosting) | proche du modèle actuel | 71 ms* | 0,4 Mo (~13× plus léger) |
  
**Point de vigilance :** la fiabilité indicative ci-dessus est mesurée sur les mêmes données que l'entraînement (le legacy ne sépare pas données d'entraînement et données de test) — ces chiffres servent à comparer les 3 modèles entre eux, pas à juger leur qualité réelle en production.
  
## 6. Tableau consolidé des risques
 
*(15 constats, hiérarchisés — 🔴 majeur, 🟠 significatif, 🟡 à surveiller)*
 
| # | Constat | Sévérité | Conséquence pour MediVox |
|---|---|---|---|
| 1 | Le modèle sous-détecte les séjours réellement prolongés 3× plus souvent chez les femmes, sans explication clinique | 🔴 | Traitement différencié par sexe, difficile à défendre |
| 2 | Base légale RGPD (art. 9) non documentée pour ce traitement de données de santé | 🔴 | Traitement potentiellement sans fondement juridique vérifié |
| 3 | Mot de passe en clair dans le code source versionné | 🔴 | Accès non autorisé possible à la base de données |
| 4 | Service entier sur une seule machine, déployée à la main, sans sauvegarde | 🔴 | Arrêt total du service en cas de panne, sans plan de reprise connu |
| 5 | Aucune trace de fonctionnement enregistrée (logs) | 🔴 | Incidents indétectables ; empêche aussi de vérifier la conformité (#8) |
| 6 | Le biais existait déjà dans l'historique des données, avant le modèle | 🟠 | Changer uniquement le modèle ne suffira pas à corriger le problème |
| 7 | Le sexe est utilisé comme variable prédictive sans justification médicale documentée | 🟠 | Écart de traitement difficile à justifier en cas de contrôle |
| 8 | Impossible de vérifier si le score joue un rôle déterminant dans une décision (art. 22) | 🟠 | Qualification juridique du traitement impossible à trancher en l'état |
| 9 | Aucune vérification des données reçues en entrée du script | 🟠 | Plantage imprévisible du système en production |
| 10 | Aucun processus de déploiement automatisé | 🟠 | Reprise lente après incident, sans garantie sur la version en service |
| 11 | Niveau de risque réglementaire (AI Act) indéterminé | 🟡 | Obligations légales non identifiables tant que l'usage réel n'est pas connu |
| 12 | Aucune traçabilité de version du modèle en production | 🟡 | Impossible de savoir avec certitude quelle version tourne actuellement |
| 13 | Code dupliqué entre l'entraînement et la prédiction | 🟡 | Risque d'incohérence silencieuse lors d'une future modification |
| 14 | Impact du volume de requêtes non mesurable (volume actuel inconnu) | 🟡 | Coût actuel négligeable, mais non représentatif d'une charge réelle inconnue |
| 15 | Une alternative plus légère existe mais ne résout pas le biais | 🟡 | Risque de fausse solution si un changement de modèle est décidé seul |
 
## 7. Questions ouvertes pour le client
 
**Pour ⚖️ Marc, en priorité :**
- Le score est-il consulté par un soignant qui décide ensuite, ou déclenche-t-il une action automatique (allocation de lit, alerte) ?
- Quelle est la base légale actuellement invoquée pour ce traitement au titre de l'art. 9 RGPD ?
- L'usage du sexe comme variable prédictive repose-t-il sur une justification clinique documentée quelque part ?
- Ce sous-signalement différencié par sexe avait-il déjà été identifié avant cet audit ?
- Le score conditionne-t-il, même indirectement, un accès à des soins ou un triage aux urgences ?
- L'écart constaté dans l'historique des données (avant même le modèle) a-t-il une explication opérationnelle connue ?
**Pour 👩‍💻 Hélène, en priorité :**
- Quel est le volume actuel de prédictions traitées par jour ou par semaine ?
- Existe-t-il une sauvegarde du modèle ou du serveur non documentée dans le code fourni ?
- Le mot de passe trouvé en clair est-il toujours valide en production aujourd'hui ?
- Des incidents ont-ils déjà été signalés par les équipes, et comment ont-ils été traités faute de traçabilité ?
- Un changement de modèle vers une alternative plus légère est-il envisagé, indépendamment de la question du biais ?
