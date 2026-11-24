# TP1 - Le modèle client-serveur

**Cours :** Applications Client-Serveur et Web (ACSW), DUT2 Réseaux et Télécommunications  
**Établissement :** UCAD / École Supérieure Polytechnique, Département Génie Informatique  
**Année universitaire :** 2026-2027

## Binôme

Fatou Ndiollé Diouf  
Diarietou Oumoul Kalsoum Niang

## Contenu du dossier

| Fichier | Description |
|---|---|
| `README.md` | Réponses aux questions du TP (ce fichier) |
| Compte rendu PDF | Compte rendu complet, avec toutes les captures d'écran |

Les captures d'écran (commandes, résultats, Wireshark, HeidiSQL) sont dans le compte rendu PDF.

## Objectif

Mettre en pratique le modèle client-serveur avec deux services installés sur VM1 : MariaDB (base de données, port 3306) et Apache (serveur web, port 80). Les clients sont l'hôte Windows et VM2. Le TP permet de distinguer le rôle du client et celui du serveur, d'observer plusieurs connexions simultanées, de vérifier les droits des comptes, de comparer MariaDB et HTTP, puis d'examiner une panne du serveur et le trafic réseau.

## Environnement utilisé

Les adresses attribuées par VMware diffèrent de celles du sujet (192.168.56.x). Adresses utilisées :

| Machine | Adresse | Rôle |
|---|---|---|
| Hôte Windows | 192.168.8.1 | Client : HeidiSQL, navigateur |
| VM1 (serveur) | 192.168.8.130 | MariaDB (3306), Apache (80) |
| VM2 (client) | 192.168.8.131 | Client : `mariadb`, `curl` |

`bind-address` de MariaDB réglé sur `0.0.0.0`.

Pare-feu UFW inactif.

Comptes `etudiant` et `secretariat` créés par `utilisateurs_tp1.sql` pour `192.168.56.%`, puis comptes complémentaires créés pour `192.168.8.%` (et `192.168.8.1` pour `etudiant` depuis Windows), avec les mêmes droits.

## Mise en place

Exécution de `scolarite.sql` sur VM1 : création de la base `scolarite` et des tables `filiere`, `etudiant`, `ec`, `note`.

Exécution de `utilisateurs_tp1.sql` : création des comptes `etudiant` (lecture seule) et `secretariat` (lecture, insertion et modification dans `note`, pas de suppression).

Installation d'Apache2 sur VM1 et copie de `index.html` dans `/var/www/html/index.html`.

# Réponses aux questions

## Partie 1 : Préparer le serveur

### Q1. Combien d'étudiants, d'EC et de notes ? Schéma des quatre tables.

La base contient 30 étudiants, 6 EC et 180 notes.

| Table | Clé primaire | Clés étrangères | Autres colonnes |
|---|---|---|---|
| `filiere` | `code` |  | `filiere` |
| `etudiant` | `matricule` | `filiere` → `filiere.code` | `prenom`, `nom`, `email` |
| `ec` | `code` |  | `intitule`, `credit` |
| `note` | `id` | `ec` → `ec.code`, `matricule` → `etudiant.matricule` | `valeur` |

Une filière regroupe plusieurs étudiants, un étudiant obtient plusieurs notes, et un EC fait l'objet de plusieurs notes. Une note est unique pour un couple étudiant-EC.

### Q2. Le client `mariadb` et le serveur `mariadbd` sont-ils le même programme ?

Non, ce sont deux programmes distincts. Dans une session locale, ils tournent sur la même machine (VM1), mais le client envoie ses requêtes au processus serveur, qui gère la base et renvoie les résultats. `ps -ef | grep -i mariadb` montre bien deux processus différents : `mariadbd` (lancé par l'utilisateur `mysql`) et `mariadb` (le client, lancé via `sudo`).

## Partie 2 : Ouvrir MariaDB au réseau

### Q3. Que signifie l'erreur 10061 ? VM1 était-elle joignable ? MariaDB fonctionnait-il ?

HeidiSQL n'a pas pu se connecter à VM1 sur le port 3306. La commande `ss` montrait que MariaDB écoutait uniquement sur `127.0.0.1:3306`. Le serveur fonctionnait sur VM1, mais n'acceptait pas encore les connexions venant du réseau.

### Q4. Différence entre écouter sur 127.0.0.1 et sur 0.0.0.0 ? Pourquoi Ubuntu choisit-il 127.0.0.1 ?

`127.0.0.1` est l'adresse de boucle locale : seules les connexions venant de VM1 elle-même peuvent atteindre le service. `0.0.0.0` demande au serveur d'écouter sur toutes les interfaces IPv4 de la machine, ce qui permet l'accès distant. La configuration par défaut est locale car elle est plus restrictive, donc plus sûre.

### Q5. Résultat de `SELECT USER(), CURRENT_USER();` et différence.

Depuis VM2 :

- `USER()` : `etudiant@192.168.8.131`
- `CURRENT_USER()` : `etudiant@192.168.8.%`

`USER()` indique le nom utilisé par le client et son adresse. `CURRENT_USER()` indique le compte MariaDB retenu par le serveur pour authentifier la connexion et vérifier les privilèges. Ici, la connexion venant de 192.168.8.131 a été associée au compte `etudiant` autorisé depuis `192.168.8.%`.

### Q6. Connexion en `root` depuis le réseau ?

La tentative depuis Windows a été refusée (`Access denied`, erreur 1045). Le compte `root` est associé uniquement à `localhost`. C'est une bonne mesure de sécurité, car `root` possède des privilèges très étendus. Pour l'accès distant, on utilise des comptes dédiés (`etudiant`, `secretariat`) qui n'ont que les droits nécessaires.

## Partie 3 : Plusieurs clients, un seul serveur

### Q7. Sortie de PROCESSLIST et de `ss`, adresses IP et ports source.

Pendant `SELECT SLEEP(20)` lancé en même temps depuis Windows et VM2, `PROCESSLIST` montre deux connexions :

- `etudiant` depuis `192.168.8.1:56310`
- `secretariat` depuis `192.168.8.131:53426`

Les ports 56310 et 53426 sont des ports source éphémères choisis par le système d'exploitation de chaque client. Ils sont différents entre eux et différents de 3306, car 3306 est le port d'écoute du serveur, tandis que chaque connexion cliente a besoin de son propre port source.

### Q8. Messages d'erreur, où sont les droits, qui les vérifie ?

Erreurs observées :

- `ERROR 1045` (accès refusé) pour des identifiants non acceptés.
- `ERROR 1142` quand `etudiant` tente un `UPDATE`, ou quand `secretariat` tente un `DELETE`, sans en avoir le droit.
- `ERROR 2002` quand MariaDB est arrêté.

Les comptes et leurs privilèges sont enregistrés et vérifiés par le serveur, à chaque connexion et à chaque requête. Le client ne peut pas s'accorder des droits lui-même. Cela renforce la sécurité, à condition de limiter les droits au strict nécessaire et de protéger les échanges réseau (par exemple avec TLS).

### Q9. Le client Windows voit-il immédiatement la note modifiée depuis VM2 ?

Oui. HeidiSQL et VM2 consultent la même base `scolarite` hébergée sur VM1. La modification faite depuis VM2 est enregistrée sur ce serveur central, et le client Windows lit la valeur mise à jour dès qu'il relance la requête. La centralisation évite que chaque client garde sa propre copie de la base, ce qui garantit la cohérence.

## Partie 4 : Serveur web

### Q10. Requête et réponse dans `curl -v`.

Les lignes commençant par `>` sont envoyées par le client, celles commençant par `<` sont renvoyées par le serveur.

Requête : première ligne `GET / HTTP/1.1` (méthode, ressource demandée, version HTTP). En-têtes : `Host: 192.168.8.130` (serveur visé), `User-Agent: curl/7.81.0` (logiciel client), `Accept: */*` (types acceptés).

Réponse : première ligne `HTTP/1.1 200 OK` (version HTTP et résultat : succès). En-têtes : `Date`, `Server: Apache/2.4.52 (Ubuntu)`, `Content-Type: text/html`, `Content-Length`, etc. Puis le contenu HTML de la page.

Depuis Windows, `http://192.168.8.130/` affiche la page « Service scolarité » servie par Apache.

### Q11. Informations enregistrées dans `access.log`.

Le journal montre les visites de Windows (`192.168.8.1`) et de VM2 (`192.168.8.131`) : adresse IP, date et heure, requête `GET /`, code de réponse `200`, taille et navigateur (`Chrome/Edge` ou `curl/7.81.0`). Les erreurs `404` sur `favicon.ico` concernent l'icône facultative demandée par le navigateur, pas la page.

### Q12. Comment le système remet-il un segment TCP au bon programme ? Pourquoi le client doit-il connaître l'adresse ET le port ?

Les deux serveurs partagent l'adresse de VM1 mais écoutent sur des ports différents. Le système remet le segment au service qui écoute sur le port de destination : MariaDB sur 3306, Apache sur 80. Le client doit connaître l'adresse IP (pour identifier la machine) et le port (pour identifier le service précis sur cette machine).

### Q13. MariaDB (connexion persistante) et HTTP (requête-réponse) : avantages, inconvénients, mémorisation de l'utilisateur.

MariaDB : avantage : plusieurs requêtes sans refaire la connexion ni l'authentification ; inconvénient : une connexion ouverte consomme des ressources du serveur même quand elle est inactive.

HTTP : avantage : simple, le serveur n'a pas à garder chaque connexion ouverte entre les pages ; inconvénient : il ne reconnaît pas automatiquement qu'une nouvelle requête vient du même utilisateur.

Pour « se souvenir » d'un utilisateur connecté, le site utilise une session : après l'identification, le serveur crée un identifiant de session et l'envoie au navigateur dans un cookie. Le navigateur renvoie ce cookie à chaque requête et le serveur retrouve la session correspondante.

## Partie 5 : Les limites du modèle

### Q14. Que se passe-t-il quand le serveur est arrêté ?

Quand MariaDB est arrêté, les clients reçoivent une erreur de connexion (`ERROR 2002` ou `10061` selon le client). Les deux clients testés (Windows avec HeidiSQL, VM2 avec `mariadb`) sont affectés. Ce défaut s'appelle un point unique de défaillance (single point of failure) : si le serveur central tombe, tous ses clients perdent l'accès au service.

### Q15. Ce qu'on voit dans la capture réseau.

Capture Wireshark sur l'interface VMnet8 avec le filtre `tcp.port == 3306`, complétée par une capture `tcpdump` sur VM1.

Compte utilisé : `etudiant`

Requête : `SELECT nom, valeur FROM scolarite.note JOIN scolarite.etudiant USING (matricule) WHERE matricule = 'ESP260001';`

Réponse : les noms d'EC et les valeurs des notes.

Le mot de passe n'apparaît pas lisiblement (mécanisme `mysql_native_password`), mais la requête et les données échangées restent lisibles sans chiffrement. Solution : activer le chiffrement TLS/SSL entre le client et MariaDB.

## Synthèse

| Caractéristique | MariaDB | Apache (HTTP) |
|---|---|---|
| Machine et port du serveur | VM1, 192.168.8.130:3306 | VM1, 192.168.8.130:80 |
| Logiciel client utilisé | HeidiSQL (Windows), `mariadb` (VM2) | Navigateur (Windows), `curl` (VM2) |
| Qui prend l'initiative ? | Le client ouvre la connexion | Le navigateur ou `curl` envoie une requête |
| Durée de vie d'une connexion | Maintenue pendant la session | Échange requête-réponse ; état géré à part (session, cookie) |
| Où sont vérifiés les droits ? | Sur le serveur MariaDB | Apache sert la page ; règles web ou applicatives si une protection est configurée |
| Effet d'un arrêt du serveur | Les clients ne peuvent plus interroger la base | Les clients ne peuvent plus obtenir la page |

### Q16. Définition du modèle client-serveur, avantages et inconvénients.

Dans le modèle client-serveur, des clients prennent l'initiative d'envoyer des requêtes à un serveur qui écoute sur un port connu d'une machine connue. Le serveur centralise les données et les règles d'accès, traite les demandes et renvoie une réponse. Plusieurs clients peuvent utiliser le même service en parallèle, sans communiquer entre eux.

**Avantages :**

- Partage des données et contrôle centralisé des permissions.
- Administration simplifiée, car les données résident au même endroit.

**Inconvénients :**

- Les clients dépendent de la disponibilité du serveur (point unique de défaillance).
- Sans chiffrement, les échanges peuvent être observés sur le réseau.

## Conclusion

Ce TP a permis de mettre en pratique le modèle client-serveur avec MariaDB et Apache sur VM1, testés depuis Windows et VM2. La configuration a été adaptée aux adresses VMware, les droits des comptes ont été vérifiés et plusieurs clients simultanés ont été observés. L'arrêt de MariaDB a montré la dépendance des clients envers le serveur, et la capture réseau a rendu visibles les échanges. Le modèle client-serveur facilite le partage et la gestion des données, mais il faut contrôler les accès et protéger les échanges sur le réseau.
