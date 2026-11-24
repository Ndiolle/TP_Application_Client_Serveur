# TP2  -  Architectures 2-tiers et 3-tiers comparées




\## Contenu

\- `compte\_rendu\_tp2\_application\_cleint-serveur.pdf` : compte rendu du TP

\- `index.html` : page d’accueil du serveur web

\- `style.css` : feuille de style de l’application



\## Environnement

\- VM1 : 192.168.8.130

\- CLIENT / VM2 : 192.168.8.131

\- Poste Windows : 192.168.8.1



Les adresses utilisées diffèrent de celles de l’énoncé. Les commandes et les tests ont été adaptés à cet adressage.





## Objectif

Comparer les architectures client-serveur à 2-tiers et à 3-tiers, en observant leur fonctionnement, leur déploiement, le trafic réseau et la sécurité.

## Adressage utilisé

Le TP indique VM1 à `192.168.56.11` et VM2 à `192.168.56.12`. Dans notre environnement, VM1 utilise `192.168.8.130`, la machine CLIENT/VM2 utilise `192.168.8.131` et le poste Windows utilise `192.168.8.1`. Les commandes et tests ont été adaptés à ces adresses.

## Partie 1 — Application en 2-tiers

### Q1. Présentation, logique métier et accès aux données

Dans `client_lourd.py`, la présentation correspond aux menus et aux résultats affichés avec `print` et `input`. La logique métier comprend les calculs de moyennes et les règles de validation. L’accès aux données comprend la connexion à MariaDB, les requêtes SQL et la récupération des résultats. Ces fonctions s’exécutent sur la machine cliente qui lance le programme, Windows ou VM2.

### Q2. Informations exposées par `client_lourd.py`

Une personne qui récupère le programme peut voir l’adresse et le port du serveur de base de données, le nom de la base et les identifiants présents dans le code. Elle pourrait utiliser ces informations pour tenter de se connecter directement à MariaDB et effectuer des requêtes selon les droits du compte.

### Q3. Déploiement sur 300 postes

Il faudrait installer Python et PyMySQL sur chaque poste, copier `client_lourd.py` sur les 300 postes et configurer chacun d’eux pour joindre directement le serveur MariaDB.

## Partie 2 — Application en 3-tiers

### Q4. Comparaison des résultats

Les deux architectures ont donné une moyenne pondérée de `10,75/20` pour ESP260001 et `18 admis sur 30`. Elles utilisent la même base et les mêmes règles de calcul ; la séparation en couches ne change donc pas les résultats.

### Q5. Installation sur Windows

Pour utiliser la version 3-tiers, aucun logiciel particulier n’a été installé sur Windows : un navigateur web suffisait. En 2-tiers, il fallait installer Python et PyMySQL et copier le client lourd sur chaque poste.

### Q6. Requête vers `config.php`

La requête retourne une réponse HTTP `200 OK` avec un contenu vide. Le serveur exécute le fichier PHP ; son code source et le mot de passe ne sont pas envoyés au navigateur.

### Q7. Comptes MariaDB

En 3-tiers, un seul compte MariaDB suffit : le serveur d’application se connecte à la base pour tous les utilisateurs. En 2-tiers, chaque poste client doit pouvoir se connecter directement à la base ; il faut donc gérer un compte utilisable par chaque client.

### Q8. Contenu reçu par le navigateur

Le code source affiché avec `Ctrl+U` ne révèle ni les requêtes SQL ni les règles de calcul exécutées côté serveur. Le navigateur reçoit le HTML généré par l’application, ainsi que les éléments nécessaires à l’affichage.

## Partie 3 — Évolution d’une règle métier

### Q9. Comparaison des clients 2-tiers

Sur Windows, la version 1.0 valide ESP260011 avec une moyenne de `12,55` et affiche `18/30` admis. Sur Ubuntu/VM2, la version 1.1 refuse la validation, car la note de MAT est `7,50`, inférieure à 8 ; elle affiche `16/30` admis. Les résultats diffèrent parce que le client Windows n’a pas reçu la nouvelle règle. Dans un service de scolarité, cela pourrait entraîner des décisions incohérentes pour un même étudiant.

### Q10. Mise à jour en 3-tiers

En 3-tiers, le fichier `metier.php` a été modifié sur le serveur d’application VM2. Tous les utilisateurs de l’application web bénéficient de la nouvelle règle dès leur prochaine requête, sans mise à jour de leur poste. Le client lourd Windows conserve sa version locale 1.0.

### Q11. Emplacement de la règle éliminatoire

Dans le client lourd, la règle a été ajoutée en Python, dans le relevé de notes et dans le calcul du nombre d’admis. En 3-tiers, elle a été ajoutée dans `metier.php` : le relevé vérifie les notes individuelles et le classement exclut les étudiants ayant une note inférieure à 8.

## Partie 4 — Volume de données et réseau

### Q12. Mesures comparées

| Mesure | 2-tiers | 3-tiers |
|---|---:|---:|
| Lignes reçues ou volume transféré au client | 120 180 lignes (2 407,3 Kio) | 1 326 octets |
| Temps de calcul du classement | 7 024 ms | 7 154 ms |
| Machine ayant effectué le calcul | Client (VM2) | Serveur de base de données (VM1) |

L’écart de volume vient du fait qu’en 2-tiers le client reçoit les nombreuses lignes nécessaires au calcul, alors qu’en 3-tiers le serveur effectue le calcul et transmet principalement le résultat affiché. Avec 300 postes sur une liaison de 10 Mbit/s, ou avec un client en connexion mobile, le transfert important du 2-tiers pourrait ralentir la consultation et consommer davantage de bande passante.

### Q13. Mesure `Bytes_sent`

`Bytes_sent` est un compteur cumulatif du nombre d’octets envoyés par MariaDB à ses clients. La différence entre les valeurs avant et après une requête donne le volume envoyé pendant la mesure. Cette mesure est plus directe qu’une estimation, mais une activité simultanée d’autres clients peut aussi influencer le compteur.

## Partie 5 — Sécurité

### Q14. Effet des règles UFW

Nous avons autorisé les ports 22 (SSH) et 80 (HTTP), autorisé le port 3306 (MariaDB) depuis `192.168.8.131`, puis ajouté un refus général pour le port 3306. Avec UFW actif, le client lourd lancé depuis Windows (`192.168.8.1`) n’a pas pu se connecter directement à MariaDB sur VM1 (`192.168.8.130:3306`). En revanche, le serveur web répondait sur le port 80.

Cette protection ne peut pas être appliquée de la même manière en 2-tiers : le client lourd doit joindre directement MariaDB. Si son accès au port 3306 est bloqué, il ne peut plus fonctionner. En 3-tiers, le serveur d’application accède à la base ; l’accès à MariaDB peut donc être réservé à ce serveur.

### Q15. Deux autres risques du client lourd

1. Les identifiants de la base présents sur les postes clients peuvent être récupérés. En 3-tiers, ils restent sur le serveur d’application.
2. Un client connecté directement à MariaDB peut tenter des requêtes non prévues ou des modifications. En 3-tiers, le serveur d’application contrôle les opérations accessibles.

Après les tests, UFW a été désactivé avec `sudo ufw disable`. Le client lourd a alors pu se connecter à MariaDB.

## Synthèse

| Critère | 2-tiers | 3-tiers |
|---|---|---|
| Logiciel à installer sur chaque poste | Client lourd, Python et PyMySQL | Navigateur web |
| Déploiement d’une nouvelle règle | Mise à jour de chaque client | Mise à jour centralisée sur le serveur d’application |
| Cohérence entre utilisateurs | Peut varier selon la version du client | Même application et mêmes règles pour tous |
| Trafic réseau côté client | Échanges directs avec la base ; volume potentiellement élevé | Échanges avec le serveur web ; résultats calculés côté serveur |
| Exposition du SGBD et des mots de passe | Accès direct depuis les clients ; identifiants potentiellement présents sur les postes | Accès réservé au serveur d’application ; identifiants centralisés |
| Charge des serveurs | Davantage de traitement sur les clients | Davantage de traitement sur les serveurs |
| Simplicité de développement | Plus simple pour une petite application | Plus complexe initialement, mais centralisée et plus facile à faire évoluer |

### Q16. Quand choisir le 2-tiers ou le 3-tiers ?

Le 2-tiers peut convenir à une petite application utilisée par peu de postes maîtrisés, sur un réseau de confiance, avec des données peu sensibles et peu de changements à déployer. Pour un service de scolarité utilisé par toute l’université, le 3-tiers est préférable : l’application et les règles sont centralisées, la base est moins exposée et la maintenance est facilitée.

## Conclusion

Ce TP a montré qu’en 2-tiers, le client se connecte directement à la base de données. Cette architecture peut convenir à un petit environnement, mais elle expose davantage la base et complique les mises à jour. En 3-tiers, le serveur d’application centralise les échanges avec MariaDB et les clients utilisent le service web. Cette architecture facilite la maintenance et permet de limiter l’accès direct à la base ; elle convient mieux à un service utilisé à l’échelle d’une université.
