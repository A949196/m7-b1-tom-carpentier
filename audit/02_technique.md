# Audit — technique
> Observation du code fourni dans `legacy/`, sans modification, conformément au périmètre de l'audit.
 
## 1. Architecture (modularité, couplage)
 
- **Aucune séparation des responsabilités** : `predict.py` mélange chargement du modèle, parsing des arguments, inférence et affichage dans un seul script linéaire de moins de 20 lignes — pas de fonctions, pas de classes, pas de module réutilisable
- **Couplage fort au système de fichiers local** : le chemin vers le modèle est en dur (`"legacy/dms_predictor_v1.joblib"`), relatif au répertoire d'exécution — aucune configuration externalisée (pas de variable d'environnement, pas de fichier de config)
- **`train.py` et `predict.py` sont deux scripts indépendants sans code partagé** : la construction des features (`age`, `nb_comorbidites`, `imc`, `sexe_bin`) est dupliquée entre les deux fichiers plutôt que factorisée dans une fonction commune
- **Pas de pipeline scikit-learn** (`Pipeline`, `ColumnTransformer`) : l'encodage de `sexe` en `sexe_bin` est fait à la main dans les deux scripts.

## 2. Sécurité (secrets, validation, transport)
 
- **Secret en clair dans le code source** : `train.py` contient `DB_PASSWORD = "medivox_prod_2024"` en dur — commis dans un fichier versionné, visible par quiconque a accès au dépôt ou au serveur.
- **Aucune validation des entrées** : `predict.py` caste directement `sys.argv[1:4]` (`int`, `int`, `float`, `int`) sans schéma, sans bornes, sans gestion d'exception — un argument manquant, mal typé, ou hors plage clinique plausible provoque un plantage non explicite
- **Transport non observable dans le code fourni** : `predict.py` est invoqué via SSH selon le brief — aucune trace dans le code d'un chiffrement de bout en bout applicatif ni d'authentification au niveau du script lui-même
- **Aucune journalisation des accès ou des résultats** : le seul output est un `print()` console — pas de log applicatif.

## 3. Scalabilité
 
- **Rechargement complet du modèle à chaque appel** : `joblib.load(...)` s'exécute en tête de script à chaque invocation SSH — pas de processus persistant, pas de cache mémoire. Chaque prédiction paie intégralement le coût de chargement, pas seulement le coût d'inférence
- **Traitement strictement séquentiel** : aucune trace de service d'inférence (API REST, gRPC), de queue de requêtes ou de parallélisation — le système traite une requête = une session SSH, sur une seule machine
- **Pas de mesure de charge documentée** : aucune indication dans le code ou les fichiers fournis sur le nombre de prédictions/jour actuellement traité ni sur une éventuelle saturation déjà observée

## 4. Points de rupture (SPOF — single point of failure)
 
- **Le modèle vit uniquement sur le disque local du serveur de production**, déployé historiquement par `scp` manuel (commentaire de `train.py` : *« déploie par scp sur le serveur de prod »*) — aucune redondance, aucune sauvegarde documentée.
- **Aucun CI/CD** : le déploiement manuel par `scp` signifie qu'il n'existe aucun pipeline reproductible pour redéployer rapidement en cas d'incident, et aucune garantie que le code en prod correspond à une version tracée
- **Aucun monitoring** : conséquence directe de l'absence de journalisation — une panne silencieuse est indétectable tant qu'un utilisateur ne signale pas l'absence de réponse
- **Absence de versionning du modèle** : le fichier `dms_predictor_v1.joblib` n'a ni metadata, ni date, ni hash documenté — impossible de savoir avec certitude quelle version du modèle tourne en prod à un instant donné, ni de revenir à une version antérieure de façon fiable

## 5. Constat transverse (relié au volet éthique)
 
- L'absence de journalisation **bloque une question posée dans le volet éthique** : il est aujourd'hui impossible de mesurer le taux de désaccord entre le score et la décision clinique finale — donnée nécessaire pour trancher si le score joue un rôle déterminant. C'est donc un risque technique avec une conséquence directe côté conformité.

## 6. Questions ouvertes (volet technique)
 
- Quel est le volume actuel de prédictions traitées par jour/semaine, et une saturation ou une lenteur a-t-elle déjà été observée en pratique ?
- Existe-t-il une sauvegarde du modèle ou du serveur, même manuelle, en dehors de ce qui est documenté dans le code fourni ?
- Le mot de passe en clair dans `train.py` est-il toujours valide en production actuellement ?
- Y a-t-il eu des incidents (script planté, résultat incohérent) déjà remontés par les équipes cliniques, et comment ont-ils été traités faute de logs ?