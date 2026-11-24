# TP1 : Le modèle client-serveur

Cours d'Applications Client-Serveur et Web, ESP / UCAD, encadré par Dr Keba Gueye.

## Résumé

Dans ce TP, nous avons installé un serveur de base de données MariaDB et un serveur web Apache sur une machine virtuelle (VM1). Nous nous y sommes ensuite connectés depuis deux clients, l'ordinateur Windows et une deuxième machine virtuelle (VM2), pour observer comment un client envoie une demande à un serveur et reçoit une réponse.

| Machine | Adresse IP | Rôle |
|---|---|---|
| Hôte Windows | 192.168.216.1 | Client : HeidiSQL, Chrome, Wireshark |
| VM1 (Ubuntu) | 192.168.216.129 | Serveur : MariaDB (port 3306) et Apache (port 80) |
| VM2 (Ubuntu) | 192.168.216.130 | Client : mariadb, curl |

## Membres

- Mouhamadoul Ousmane Faye
- Mouhamed Fall

## Fichiers

- `rapport_TP1.pdf` : le compte rendu complet avec les captures d'écran

## Réponses aux questions

**Q1. Combien d'étudiants, d'EC et de notes ? Schéma des tables.**
La base `scolarite` contient 30 étudiants, 6 EC et 180 notes (30 × 6 : une note par étudiant et par EC). Elle a quatre tables : `filiere`, `etudiant`, `ec` et `note`. Clés primaires : `filiere.code`, `etudiant.matricule`, `ec.code`, `note.id`. Clés étrangères : `etudiant.filiere` vers `filiere.code`, `note.matricule` vers `etudiant.matricule`, `note.ec` vers `ec.code`. La table `note` impose aussi une seule note par (matricule, ec) et une valeur entre 0 et 20.

**Q2. Le client mariadb et le serveur mariadbd sont-ils le même programme ?**
Non, ce sont deux programmes et deux processus. Avec `ps -ef`, on voit `/usr/sbin/mariadbd` (le serveur, PID 1973, lancé au démarrage par l'utilisateur mysql, sans terminal) et `mariadb` (le client, PID 4175, lancé par root depuis un terminal, qui n'existe que le temps de la session).

**Q3. Que signifie l'erreur 10061 ? VM1 était-elle joignable ? MariaDB fonctionnait-il ?**
10061 veut dire « connexion refusée ». VM1 était joignable et MariaDB tournait, mais il n'écoutait que sur 127.0.0.1, donc pas sur la carte réseau. Sur notre capture, HeidiSQL a affiché 10060 (délai dépassé), ce qui arrive quand un pare-feu ignore les paquets. La conclusion est la même : le service est injoignable depuis le réseau.

**Q4. Différence entre 127.0.0.1 et 0.0.0.0 ?**
127.0.0.1 est l'adresse de bouclage : seules les connexions venant de la machine elle-même sont acceptées. 0.0.0.0 veut dire « toutes les interfaces » : les connexions venant du réseau (Windows, VM2) sont acceptées aussi. Ubuntu choisit 127.0.0.1 par sécurité, pour ne pas exposer la base au réseau sans décision de l'administrateur.

**Q5. Résultat de `SELECT USER(), CURRENT_USER();`**
`USER()` renvoie `etudiant@192.168.216.1` et `CURRENT_USER()` renvoie `etudiant@192.168.216.%`. `USER()` est l'identité présentée par le client (nom et adresse réelle). `CURRENT_USER()` est le compte MariaDB retenu par le serveur pour appliquer les droits. Un compte est un couple (utilisateur, machine d'origine).

**Q6. Peut-on se connecter en root depuis Windows ? Pourquoi est-ce une bonne chose ?**
Non : root n'existe que pour `localhost`, et l'essai depuis Windows est refusé (`Access denied for user 'root'@'192.168.216.1'`). C'est une bonne chose car root a tous les privilèges : lui interdire l'accès réseau protège le compte le plus puissant contre une attaque à distance.

**Q7. PROCESSLIST et ss : adresses et ports sources.**
PROCESSLIST montre `etudiant` depuis 192.168.216.1:49673 (Windows) et `secretariat` depuis 192.168.216.130:56424 (VM2). `ss` confirme les deux connexions vers 192.168.216.129:3306. Les ports sources ont été choisis par le système de chaque client : ce sont des ports éphémères. Ils sont différents de 3306, qui est le port d'écoute fixe du serveur, et différents entre eux pour que le serveur distingue chaque connexion.

**Q8. Erreurs obtenues, où sont les droits, qui les vérifie ?**
`etudiant` (Windows) : `UPDATE command denied` (erreur 1142). `secretariat` (VM2) : le `UPDATE` réussit mais le `DELETE` est refusé (erreur 1142). Les droits sont enregistrés sur le serveur, dans la base système `mysql`, et c'est le serveur qui les vérifie, pas le client. La sécurité repose donc sur le serveur, ce qui permet le principe du moindre privilège.

**Q9. Windows voit-il tout de suite la note modifiée depuis VM2 ?**
Oui, le `SELECT` depuis Windows renvoie directement 12,0, sans reconnexion. Les données ne sont stockées qu'à un seul endroit (VM1) : tous les clients lisent la même copie, il n'y a rien à synchroniser.

**Q10. Requête et réponse dans `curl -v`.**
La requête (lignes `>`) commence par `GET / HTTP/1.1` (méthode, ressource, version) avec les en-têtes `Host`, `User-Agent: curl/8.5.0` et `Accept: */*`. La réponse (lignes `<`) commence par `HTTP/1.1 200 OK` (version, code de statut) avec les en-têtes `Date`, `Server: Apache/2.4.58`, `Content-Length`, `Content-Type`, etc. Une ligne vide sépare les en-têtes du corps HTML.

**Q11. Que garde `access.log` sur chaque client ?**
Pour chaque requête : l'adresse IP, la date et l'heure, la requête HTTP, le code de statut, la taille de la réponse et l'User-Agent. Windows (192.168.216.1) : `GET /` code 200 avec Chrome, puis `/favicon.ico` en 404. VM2 (192.168.216.130) : `GET /` code 200 avec `curl/8.5.0`. Le serveur identifie le client par son IP et son logiciel, pas la personne.

**Q12. Comment le système sait-il à quel programme remettre un segment TCP ? Pourquoi connaître l'adresse ET le port ?**
Le système regarde le port de destination : 3306 est remis à `mariadbd`, 80 à `apache2`, car chacun écoute sur son port. Le client doit connaître l'adresse (qui désigne la machine) et le port (qui désigne le programme), car c'est lui qui prend l'initiative et qu'il n'y a aucune découverte automatique.

**Q13. Connexion persistante ou requêtes indépendantes ?**
MariaDB (connexion persistante) : on s'authentifie une fois et le serveur garde le contexte, mais chaque connexion ouverte consomme des ressources. HTTP (requêtes indépendantes) : le serveur ne garde rien, donc il supporte beaucoup de clients, mais il ne reconnaît pas le client d'une requête à l'autre. Un site se souvient d'un utilisateur avec un cookie de session (ou un jeton JWT) renvoyé par le navigateur à chaque requête.

**Q14. Messages quand le serveur est arrêté ? Combien de clients touchés ? Nom du défaut ?**
Windows : `Can't connect to server on '192.168.216.129' (10061)`. VM2 : `ERROR 2002 (HY000): Can't connect to server (115)`. Les deux clients sont touchés. Ce défaut s'appelle le point unique de défaillance (SPOF). On le limite avec de la redondance (réplication, serveur de secours).

**Q15. Que voit-on dans la capture Wireshark ? Le mot de passe est-il en clair ? Solution ?**
On lit en clair le nom du compte (`etudiant`), le texte de la requête (`SELECT nom, valeur FROM note JOIN etudiant ...`) et les données de la réponse (nom SECK et ses notes). Le mot de passe n'est pas en clair : seule une empreinte est envoyée. Tout le reste circule sans protection. Solution : chiffrer avec TLS/SSL (comptes `REQUIRE SSL`), ou passer par un tunnel SSH ou un VPN.

**Q16. Définition du modèle client-serveur, deux avantages, deux inconvénients.**
Le modèle client-serveur est une architecture où un serveur attend des demandes sur une adresse et un port connus (ici MariaDB sur VM1:3306 et Apache sur VM1:80). Les clients (HeidiSQL, mariadb, navigateur, curl) prennent l'initiative, envoient une requête et attendent la réponse, sans jamais communiquer entre eux. Le serveur centralise les données et les droits, sert plusieurs clients à la fois (distingués par IP et port source) et peut héberger plusieurs services (distingués par leur port).
- Avantages : données et droits centralisés (cohérence et contrôle en un seul point) ; clients interchangeables et simples à maintenir, sans copie locale.
- Inconvénients : point unique de défaillance (si le serveur s'arrête, tous les clients perdent le service) ; trafic en clair par défaut tant qu'on n'ajoute pas TLS.
