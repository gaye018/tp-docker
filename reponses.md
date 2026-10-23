## Exercice 1 : cycle de vie des processus

**1. Quel est le rôle du processus principal dans un conteneur ?**
C'est le programme que le conteneur exécute. Il détermine la durée de vie du
conteneur : tant qu'il tourne, le conteneur tourne.

**2. Quelle différence entre `docker stop` et `docker kill` ?**
`docker stop` envoie `SIGTERM`, attend 10 secondes, puis envoie `SIGKILL` si le
processus ne s'est pas arrêté. `docker kill` envoie directement un signal
(`SIGKILL` par défaut), sans attente.

**3. Pourquoi un conteneur s'arrête-t-il quand son processus principal se termine ?**
Parce qu'un conteneur n'est qu'un processus isolé. Quand ce processus se
termine, il ne reste plus rien à exécuter. Dans mon test, `sleep 300` s'est
terminé après 5 minutes et le conteneur est passé à `Exited (0)`.

**4. Quelle différence entre `SIGTERM` et `SIGKILL` ?**
`SIGTERM` demande poliment l'arrêt : le programme peut l'intercepter et
s'arrêter proprement, ou l'ignorer. `SIGKILL` force l'arrêt immédiat et ne peut
pas être ignoré. Dans mon test, `sleep` a ignoré `SIGTERM` et a fini tué par
`SIGKILL` (code de sortie 137).

**5. Pourquoi une application doit-elle gérer les signaux d'arrêt ?**
Pour s'arrêter proprement : terminer les requêtes en cours, enregistrer ses
données et fermer ses connexions. Sans cela, Docker doit attendre 10 secondes
puis la tuer brutalement, avec un risque de perte de données.

## Exercice 2 : couches d'une image et système de fichiers

**1. Différence entre les couches d'une image et la couche inscriptible d'un conteneur ?**
Les couches de l'image sont en lecture seule et servent à tous les conteneurs
créés depuis cette image. La couche inscriptible, elle, est propre à un seul
conteneur : c'est là que vont ses modifications.

**2. À quoi sert le copy-on-write ?**
Si un conteneur modifie un fichier de l'image, Docker en fait d'abord une copie
dans la couche inscriptible et modifie cette copie. Le fichier d'origine ne
bouge pas, donc l'image peut être partagée sans risque.

**3. Pourquoi les modifications ne changent-elles pas l'image d'origine ?**
Parce que l'image est en lecture seule : tout ce qu'on écrit va dans la couche
du conteneur. J'ai modifié `/etc/alpine-release` dans mon conteneur, et un
nouveau conteneur a affiché le fichier d'origine.

**4. À quoi sert `docker diff` ?**
À voir ce qui a changé dans un conteneur par rapport à son image : A pour
ajouté, C pour modifié, D pour supprimé. J'y ai retrouvé mon fichier et mon
dossier créés, et le fichier modifié.

**5. Pourquoi les modifications sont-elles perdues à la suppression du conteneur ?**
Parce que la couche inscriptible appartient au conteneur : quand il est
supprimé, elle l'est aussi. Pour garder des données, il faut un volume.


## Exercice 3 : build multi-stage

**1. Pourquoi utiliser plusieurs instructions FROM ?**
Chaque FROM démarre une nouvelle étape. On peut ainsi compiler le programme
dans une première étape, puis ne garder que le résultat dans une image finale
séparée.

**2. Différence entre l'étape builder et l'étape finale ?**
L'étape builder contient le compilateur Go et sert à fabriquer l'exécutable.
L'étape finale part d'une image vide (scratch) et ne contient que cet
exécutable. Seule l'étape finale devient l'image livrée.

**3. Pourquoi l'image finale ne contient-elle pas le compilateur Go ?**
Parce que le compilateur ne sert que pendant la compilation. Avec
`COPY --from=builder`, je ne copie que l'exécutable `/app` dans l'image finale,
donc tout le reste de l'étape builder est laissé de côté.

**4. Avantages d'une image finale minimale ?**
Elle est beaucoup plus légère, donc plus rapide à télécharger et à démarrer.
Mon image finale fait 4,62 Mo, alors que l'image Go utilisée pour compiler
pesait environ 79 Mo à télécharger. Elle contient aussi moins de logiciels,
donc moins de failles possibles.

**5. Précautions avant d'utiliser une image scratch en production ?**
Il faut compiler un exécutable autonome (CGO_ENABLED=0), car l'image n'a
aucune bibliothèque. Elle n'a pas de shell, donc on ne peut pas faire
`docker exec` pour déboguer. Il faut aussi y ajouter soi-même ce dont le
programme a besoin, par exemple les certificats pour les connexions HTTPS, et
prévoir un utilisateur non root.


## Exercice 4 : sécurité d'un conteneur

**1. Pourquoi exécuter une application avec un utilisateur non privilégié ?**
Si l'application est piratée, l'attaquant récupère les droits de l'utilisateur
qui la fait tourner. Avec root, il peut tout modifier. Avec un utilisateur
ordinaire, ses possibilités sont limitées : dans mon test, l'utilisateur 10001
n'a pas pu écrire dans `/root` (Permission denied).

**2. Que sont les capacités Linux ?**
Ce sont les pouvoirs de root découpés en petits droits séparés, par exemple
changer le propriétaire d'un fichier, modifier le réseau ou changer l'heure.
`--cap-drop=ALL` les retire toutes au processus.

**3. Différence entre --read-only et --tmpfs ?**
`--read-only` empêche d'écrire dans le système de fichiers du conteneur : mon
`touch /test.txt` a échoué avec « Read-only file system ». `--tmpfs` ajoute au
contraire un espace où l'on peut écrire, stocké en mémoire et limité (ici
16 Mo). Il disparaît à l'arrêt du conteneur.

**4. Pourquoi un conteneur peut-il avoir besoin de zones inscriptibles malgré un système de fichiers en lecture seule ?**
Parce que beaucoup de programmes écrivent des fichiers temporaires, des
caches ou des fichiers de verrouillage pendant leur fonctionnement. Dans mon
test, `touch /tmp/test.txt` a fonctionné grâce au tmpfs, alors que le reste
du système restait protégé.

**5. Ces options suffisent-elles à garantir une isolation parfaite ?**
Non. Les conteneurs partagent le noyau de la machine : une faille du noyau ou
une mauvaise configuration (mode privilégié, dossiers sensibles partagés)
peut permettre de sortir du conteneur. Ces options réduisent les risques, mais
il faut aussi des images fiables, des mises à jour et d'autres protections.

## Exercice 5 : diagnostic d'un conteneur qui se termine anormalement

**1. Différence entre un conteneur arrêté et un conteneur supprimé ?**
Un conteneur arrêté existe encore : il apparaît dans `docker ps -a`, et on peut
lire ses journaux et ses métadonnées. Un conteneur supprimé (`docker rm`) a
disparu avec ses données, on ne peut plus rien consulter.

**2. Que signifie un code de sortie différent de zéro ?**
Que le programme s'est terminé avec une erreur. Le code 0 veut dire que tout
s'est bien passé. Dans mon test, le code 42 était celui que j'avais choisi
avec `exit 42`.

**3. Différence entre sortie standard et sortie d'erreur ?**
La sortie standard sert aux messages normaux, la sortie d'erreur aux messages
d'erreur. Ce sont deux canaux séparés, ce qui permet de les traiter
différemment. Dans mon test, « Demarrage de l application » est passé par la
sortie standard et « Erreur simulee » par la sortie d'erreur (`>&2`).

**4. Pourquoi les journaux sont-ils utiles quand un conteneur s'arrête trop vite ?**
Parce que le conteneur n'est plus là pour qu'on regarde ce qu'il fait. Ses
journaux restent disponibles avec `docker logs` et indiquent ce qui s'est
passé avant l'arrêt.

**5. Pourquoi examiner le code de sortie plutôt que se fier au message d'erreur ?**
Un message peut être absent, incomplet ou trompeur, alors que le code de sortie
est toujours enregistré par Docker. Il dit si le programme a réussi ou échoué,
et permet de distinguer plusieurs causes (par exemple 137 pour un arrêt forcé).

## Exercice 6 : rotation des journaux

**1. Pourquoi des journaux trop volumineux peuvent-ils poser problème ?**
Les journaux sont écrits dans des fichiers sur le disque de la machine. S'ils
grossissent sans limite, ils peuvent remplir le disque et faire planter les
conteneurs ou le serveur.

**2. À quoi sert un pilote de journaux Docker ?**
Il décide où et comment Docker enregistre ce que les conteneurs affichent
(fichiers locaux, serveur de logs, etc.). Dans mon test, j'ai utilisé le
pilote `local`, alors que mon autre conteneur utilise `json-file`.

**3. Différence entre `docker logs` et le stockage des journaux sur le long terme ?**
`docker logs` relit les journaux stockés pour ce conteneur : ils disparaissent
avec lui, et la rotation supprime les plus anciens. Pour garder l'historique
longtemps, il faut les envoyer vers un système de stockage dédié.

**4. Pourquoi la rotation des journaux est-elle importante sur un serveur ?**
Un serveur tourne en continu et ses conteneurs écrivent des journaux sans
arrêt. Sans rotation, ils finissent par saturer le disque. Avec `max-size` et
`max-file`, la taille des fichiers reste limitée.

**5. Pourquoi choisir une stratégie de journalisation adaptée à la production ?**
Parce qu'il faut un compromis : garder assez de journaux pour comprendre les
pannes, sans remplir le disque. En production, on adapte la taille de rotation
à la quantité de logs, et on les envoie souvent vers un outil centralisé.

## Exercice 7 : explorer un conteneur en fonctionnement

**1. Différence entre `docker exec` et `docker run` ?**
`docker run` crée un nouveau conteneur et lance son programme principal.
`docker exec` lance une commande dans un conteneur qui tourne déjà, sans
toucher à son programme principal.

**2. À quoi sert `docker top` ?**
À voir les processus qui tournent dans un conteneur. Dans mon test, il a
montré le shell de la boucle et le `sleep 30`.

**3. Pourquoi `docker diff` est-il utile pendant un diagnostic ?**
Il montre ce qui a changé dans le conteneur par rapport à son image. Dans mon
test, il a montré le fichier `/tmp/diagnostic.txt` que j'avais ajouté.

**4. Quelles informations fournit `docker stats` ?**
L'utilisation du processeur, de la mémoire, du réseau et du disque. Mon
conteneur utilisait 0,00 % de CPU et 512 Ko de mémoire.

**5. Pourquoi éviter de modifier un conteneur en production sans documenter ?**
Ces modifications sont perdues si le conteneur est recréé, et personne ne sait
ce qui a été changé. Il vaut mieux modifier l'image, puis redéployer.

## Conclusion

Le build multi-stage a réduit l'image : mon application Go ne pèse que
4,62 Mo, car elle ne contient que l'exécutable et pas le compilateur.

Pour la sécurité, j'ai utilisé un utilisateur non root, un système de fichiers
en lecture seule avec un espace temporaire, et j'ai retiré les capacités Linux.
Cela réduit les risques, mais ne garantit pas une isolation parfaite, car les
conteneurs partagent le noyau de la machine.

J'ai aussi appris à diagnostiquer un conteneur (code de sortie, journaux,
`docker top`, `docker diff`, `docker stats`) et à limiter la taille des
journaux avec la rotation.