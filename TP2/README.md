# TP2 : Architectures 2-tiers et 3-tiers comparées

Cours d'Applications Client-Serveur et Web, ESP / UCAD, encadré par Dr Keba Gueye.

## Résumé

Dans ce TP, nous avons comparé deux façons de construire la même application de scolarité avec la même base MariaDB. La première est une architecture 2-tiers : un programme Python (client lourd) se connecte directement à la base. La deuxième est une architecture 3-tiers : une application web PHP se place entre l'utilisateur et la base. Nous les avons comparées sur le déploiement d'une nouvelle règle, la cohérence, le volume de données sur le réseau et la sécurité.

| Machine | Adresse IP | Rôle |
|---|---|---|
| Hôte Windows | 192.168.56.1 | Client : `client_lourd.py` (2-tiers), navigateur (3-tiers) |
| VM1 | 192.168.56.11 | Serveur de données MariaDB |
| VM2 | 192.168.56.12 | Client en 2-tiers, puis serveur d'application Apache + PHP en 3-tiers |

## Membres

- Mouhamadoul Ousmane Faye
- Mouhamed Fall

## Fichiers

- `rapport_TP2.pdf` : le compte rendu complet avec les captures d'écran
- `client_lourd.py` : le client lourd (2-tiers)
- les fichiers PHP de l'application 3-tiers (`metier.php`, `classement.php`, etc.)

## Réponses aux questions

**Q1. Présentation, logique métier, accès aux données : quelles lignes, sur quelle machine ?**
Dans `client_lourd.py` : la présentation (les `print` et le menu avec `input`), la logique métier (seuil de validation, moyenne pondérée par les crédits, décision VALIDÉ / NON VALIDÉ, tri) et l'accès aux données (connexion pymysql et requêtes SELECT). Les trois s'exécutent sur la même machine : le poste client. VM1 exécute seulement les requêtes SQL et renvoie les lignes.

**Q2. Que découvre un étudiant qui récupère `client_lourd.py` ?**
Le code donne l'adresse du serveur (192.168.56.11), le port (3306), la base (`scolarite`), l'identifiant et le mot de passe, en clair. Avec ces informations, il peut se connecter à la base sans passer par le programme (client mysql, outil graphique, script) et faire tout ce que le compte permet : lire les notes de tous les étudiants, explorer les tables.

**Q3. Que fallait-il installer sur 300 postes en 2-tiers ?**
Sur chaque poste : Python 3, la bibliothèque pymysql, le programme `client_lourd.py` (avec l'adresse et le mot de passe du serveur) et l'accès réseau au port 3306 de VM1. À chaque évolution, il faut mettre à jour les 300 postes.

**Q4. Les deux architectures donnent-elles les mêmes résultats ? Pourquoi est-ce indispensable ?**
Oui. Pour ESP260011, le client lourd et l'application web affichent la même moyenne pondérée : 12,55 / 20. Les deux lisent les mêmes données et appliquent la même formule. C'est indispensable car l'expérience compare des architectures, pas des données ni des règles : si les résultats différaient, on ne saurait pas d'où vient l'écart. *(Nombre d'admis de la base initiale : à compléter.)*

**Q5. Qu'a installé Windows pour le 3-tiers ?**
Rien : un navigateur suffit et Windows en a déjà un. En 2-tiers, il fallait Python, pymysql et le programme sur chaque poste. En 3-tiers, tout est installé une seule fois sur VM2, quel que soit le nombre de postes.

**Q6. Que renvoie la requête vers `config.php` ?**
`HTTP/1.1 200 OK` avec `Content-Length: 0` : le corps est vide. Apache confie le fichier `.php` à l'interpréteur PHP, qui l'exécute sur VM2. `config.php` ne fait que définir des constantes et n'affiche rien, donc le code source et le mot de passe ne sont jamais envoyés au navigateur.

**Q7. Combien de comptes MariaDB en 3-tiers ? Et en 2-tiers ?**
3-tiers : un seul compte (`app_scolarite`, autorisé depuis 192.168.56.12 uniquement), que l'on ait 10 ou 10 000 utilisateurs. 2-tiers : chaque poste se connecte à la base, donc soit un compte partagé dont le mot de passe est distribué à tous, soit jusqu'à 300 comptes à créer, tenir à jour et révoquer.

**Q8. Que contient le code source de la page (Ctrl+U) ?**
Aucune trace de SQL ni de la règle de calcul. Le navigateur reçoit uniquement du HTML déjà mis en forme, avec la moyenne et la décision déjà calculées. Le calcul et les requêtes se font sur VM2 et VM1.

**Q9. Décisions pour ESP260011 sur chaque poste en 2-tiers. Que constate-t-on ?**

| Poste | Version | Décision ESP260011 | Moyenne |
|---|---|---|---|
| Windows (client lourd) | 1.0 | VALIDÉ | 12,55 |
| VM2 (client lourd) | 1.1 | NON VALIDÉ | 12,55 |

Pour le même étudiant et les mêmes données, les deux postes donnent des décisions contradictoires : seule la version du programme diffère. Conséquences pour une scolarité : listes d'admis contradictoires, un étudiant admis à un guichet et ajourné à l'autre, contestations, procès-verbaux de jury invalides, et impossibilité de savoir quels postes ont quelle version.

**Q10. Combien de fichiers et de machines modifiés en 3-tiers ? Qui en a bénéficié, et quand ?**
Un seul fichier (`inc/metier.php`) sur une seule machine (VM2). Tous les utilisateurs de l'application en ont bénéficié immédiatement, dès le rechargement de la page, sans rien installer côté client. En 2-tiers, il aurait fallu modifier chaque poste. Remarque : cette propagation immédiate vaut aussi pour les erreurs : une faute de syntaxe dans `metier.php` a rendu toutes les pages indisponibles (erreur 500) jusqu'à sa correction. Il faut donc tester avant de déployer (`php -l`).

**Q11. Où intervenir pour la règle éliminatoire dans chaque cas ?**
2-tiers : toute la règle est dans le code Python du client. On intervient à deux endroits (le relevé et le décompte des admis), plus une nouvelle fonction, et cela sur chaque poste. 3-tiers : l'intervention est centralisée dans un seul fichier, mais touche deux niveaux : du PHP pour le relevé, et du SQL pour le décompte des admis (ajout de `MIN(n.valeur) AS pire` et de la condition `pire >= ?`).

**Q12. Volume de données reçu par le poste client, et effet sur 300 postes.**

| Mesure | 2-tiers | 3-tiers |
|---|---|---|
| Reçu par le poste client | 120 180 lignes, 2 407,3 Kio (≈ 2,35 Mio) | Une page HTML : le top 10 et le bilan |
| Temps du classement | 955 ms | *(à compléter)* |
| Qui calcule | Le poste client | Le serveur (VM1 calcule les moyennes, VM2 met en forme) |

L'écart vient de l'endroit du calcul : en 2-tiers, le client rapatrie les données brutes pour calculer lui-même ; en 3-tiers, le calcul est près de la base et seul le résultat traverse le réseau. Pour 300 postes à 10 Mbit/s : un classement fait ≈ 19,7 Mbit, soit ≈ 2 s pour un poste, et 300 × 19,7 = 5 916 Mbit, soit ≈ 592 s (près de 10 minutes) de liaison saturée si tous le lancent en même temps. En 3-tiers, chaque poste ne reçoit qu'une petite page. Sur une connexion mobile, 2,35 Mio par consultation prennent plusieurs secondes et consomment le forfait de données.

**Q13. Que mesure `Bytes_sent` ? Pourquoi est-ce une mesure honnête ?**
`SHOW SESSION STATUS LIKE 'Bytes_sent'` donne le nombre d'octets envoyés par le serveur sur cette connexion depuis son ouverture. En le lisant avant puis après la requête, la différence donne le volume réellement envoyé pour cette requête. C'est honnête car la valeur vient du serveur qui envoie (pas d'une estimation du client) et qu'elle est propre à la session, donc les autres clients ne la faussent pas. Limite : elle ne compte que le sens serveur vers client, sans les en-têtes TCP/IP.

**Q14. Que se passe-t-il quand le port 3306 n'est autorisé que depuis VM2 ?**
2-tiers : `client_lourd.py` sur Windows ne peut plus se connecter à MariaDB, car l'adresse de Windows (192.168.56.1) n'est pas autorisée. 3-tiers : l'application web continue de fonctionner, car le navigateur parle à VM2 (port 80) et c'est VM2, seule autorisée, qui interroge la base. En 2-tiers, on ne peut pas appliquer cette protection : chaque poste doit joindre la base directement, il faudrait ouvrir le port 3306 aux 300 adresses des postes, ce qui revient à exposer la base à tout le réseau.

**Q15. Deux autres risques de sécurité du client lourd.**
- Identifiants dans le programme : le mot de passe de la base est écrit dans `client_lourd.py` et tout utilisateur peut l'extraire. En 3-tiers, il reste dans `config.php` sur VM2 et n'est jamais envoyé au client.
- Logique métier chez l'utilisateur : il peut modifier le programme (changer la règle, se donner une meilleure moyenne) ou garder une version obsolète. En 3-tiers, la logique reste sur le serveur, sous le contrôle de l'administrateur.

**Q16. Quand le 2-tiers est-il acceptable ? Pourquoi le 3-tiers pour une scolarité universitaire ?**
Le 2-tiers est acceptable quand il y a peu de postes, un réseau local fermé et de confiance, des utilisateurs peu nombreux et connus, des évolutions rares, des données peu sensibles, et qu'on veut un développement rapide et peu coûteux.
Le 3-tiers s'impose pour une scolarité universitaire :
- Échelle : des milliers d'utilisateurs, aucun logiciel à installer ni à mettre à jour.
- Règles qui évoluent : une nouvelle règle s'applique partout en même temps, sans listes d'admis contradictoires.
- Données sensibles : la base n'est ouverte qu'au serveur d'application.
- Réseau et administration : moins de trafic par poste, un seul compte de base de données à protéger.

## Comparaison

| Critère | 2-tiers | 3-tiers |
|---|---|---|
| Installation sur chaque poste | Python, pymysql, programme | Un navigateur |
| Nouvelle règle | Chaque poste à mettre à jour | 1 fichier sur 1 serveur, effet immédiat |
| Cohérence | Risque de décisions contradictoires | Tous voient la même règle |
| Trafic réseau côté client | Élevé (2 407,3 Kio) | Faible (une page HTML) |
| Exposition de la base | Forte | Faible (VM2 seulement) |
| Simplicité de développement | Plus simple | Plus complexe |

## Conclusion

Sur la même application et les mêmes données, le 3-tiers déploie une règle en un seul endroit, réduit fortement le trafic réseau et permet de fermer l'accès direct à la base. Le 2-tiers est plus simple à développer mais ne convient qu'à un petit parc de postes de confiance. Pour une scolarité ouverte à toute l'université, le 3-tiers est le bon choix.
