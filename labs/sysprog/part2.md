# Partie 2: Compilation, debug et gestionnaire de signaux


## Exercice 1 : GDB et fichier core

Une fois le simulateur GPS compilé, le lancer grâce au script *gps/run.sh* :

```` bash
$ sh run.sh
PTTY: /dev/pts/X
````

**Question 1** : Que se passe-t-il au bout de quelques secondes? Qu'en
                 déduisez vous?

```` txt
segmentation core dumped
````

**Question 2** : Quel signal a reçu le processus pour se terminer ainsi? Comment
                vérifiez vous le numéro du signal reçu?

un echo $? renvoie 139

Lors d'une terminaison anormale, un fichier *core* peut être généré. Par défaut,
la génération d'un fichier core est généralement désactivée :

```` bash
$ ulimit -c
0
````

Ici la commande renvoie grâce au paramètre *-c* la taille du fichier core à
générer. La taille étant 0, aucun fichier n'est créé. Pour y remédier :

```` bash
$ ulimit -c unlimited
$ ulimit -c
unlimited
````

Relancez le simulateur GPS. Suite au crash, un fichier core doit être généré
dans le répertoire courant.

Nous allons ici utiliser GDB pour analyser le dump mémoire afin de trouver
l'origine de l'erreur. GDB est un outils très complet fournissant de
nombreuses commandes.

Pour lancer GDB et analyser un fichier core :

```` bash
$ gdb ./bin/gps core
````

Ensuite, dans le prompt GDB, utilisez la commande *bt* (pour *backtrace*) afin
de savoir comment votre programme en est arrivé là (image de la pile).

**Question 3** : Grâce à GDB et au fichier *core* généré, analysez la source du problème du binaire *gps*. Quelle partie du code est fausse? Pourquoi?

-> problème :
```` bash
#0  __strlen_avx2 () at ../sysdeps/x86_64/multiarch/strlen-avx2.S:62
#1  0x00007f14171c49d2 in _IO_puts (str=0x0) at ioputs.c:35
#2  0x00007f1417535aab in knot_to_kmh_str (not=5.51000023, size=6, 
    format=0x7f1417535f6f "%05.1f", kmh_str=0x7ffe92078262 "010.2")
    at nmea.c:23
#3  0x00007f1417535ef6 in nmea_vtg (vtg=0x7ffe920782a0) at nmea.c:178
#4  0x000055b3cb0c9c5c in write_vtg (fd=3) at gps.c:40
#5  0x000055b3cb0c9ee1 in main () at gps.c:109
````

GDB peut être aussi lancé de manière interactive :

```` bash
$ gdb ./bin/gps
````

Une fois dans le prompt, il faut lancer la commande *r* (comme *run*) et *bt*
pour récupérer la backtrace.

Il est aussi possible de placer des breakpoint sur une ligne de code, dans une
fonction, etc. Puis on peut avancer pas à pas avec *n* ou *s* (en fonction de
ce que l'on veut faire). Par exemple:

```` bash
$ gdb ./bin/gps
$ break nmea.c:19
$ r
...
$ s
...
$ n
...
````

**Question 4** : Que se passe-t-il quand vous lancez GDB en mode interactif sur
                 le binaire *gps*?

```` bash
linux-vdso.so.1 (0x00007ffd01bf0000)
libptmx.so => not found
libnmea.so => not found
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8a5feb9000)
/lib64/ld-linux-x86-64.so.2 (0x00007f8a604ad000)
````

Suite au problème repéré, allez dans le répertoire *gps/bin* et lancez la
commande suivante :

````
ldd ./gps
````

**Question 5** : À quoi sert la commande *ldd*? Quelle information
                supplémentaire cela vous apporte-t-il?

```` txt
ldd - Affiche les bibliothèques partagées nécessaires  
````

**Question 6** : Comment résoudre ce problème en tant qu'utilisateur? N'hésitez
                 pas à regarder le fichier *gps/run.sh*.

on lance la commande :
```` bash
export LD_LIBRARY_PATH=LD_LIBRARY_PATH=$(pwd)/lib
````

Relancez *ldd* puis GDB pour vérifier que votre solution a porté ses fruits.

on obtient :
```` bash
linux-vdso.so.1 (0x00007fff6497e000)
libptmx.so => /home/colin/Documents/ENSTA/Embsys/Embsys/labs/sysprog/gps/lib/libptmx.so (0x00007f05b6812000)
libnmea.so => /home/colin/Documents/ENSTA/Embsys/Embsys/labs/sysprog/gps/lib/libnmea.so (0x00007f05b660f000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f05b621e000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f05b5e80000)
/lib64/ld-linux-x86-64.so.2 (0x00007f05b6c17000)
````


**Question 7** : Quelle est la différence entre les commandes *s* et *n* dans
                 le prompt gdb suite à un breakpoint?

```` txt
Avec "step" on rentre dans les sous-fonctions, avec "next" on parcourt l'arbre d'appel sans rentrer en profondeur.
````

Il existe aussi une version de GDB pour déboguer à distance. Il y
a alors un GDBServer tournant sur la cible où le programme à déboguer est
exécuté. Ensuite, un client GDB tourne sur la machine servant à déboguer
et communique avec le serveur grâce au réseau.

**Question 8** : Dans quel contexte ce type d'outils peut être intéressant?

### À retenir

  * l'utilité de *ulimit* et comment déclencher la génération d'un fichier core
  * à quoi sert *GDB* et comment l'utiliser
  * l'utilité de *ldd*
  * pourquoi, quand et comment utiliser la variable d'environnement
    *LD_LIBRARY_PATH*

## Exercice 2 : LD_PRELOAD et sigaction

Maintenant que le problème est identifié, nous allons le résoudre. Cependant,
nous partons du principe que le code source du simulateur **NE DOIT PAS ÊTRE
MODIFIÉ**. Pour corriger le problème, nous allons utiliser la variable
d'environnement *LD_PRELOAD*. Cette variable permet de *hooker* (comprendre
*usurper*) certaines fonctions d'une application.

Utilisation :

````
LD_PRELOAD=libhook.so ./bin/gps
````

En faisant ainsi, le binaire cherchera en priorité les fonctions dont il
a besoin dans *libhook.so*! Pour que cela fonctionne, il faut que les fonctions
définies dans libhook aient exactement le même prototype.

Pour les questions suivantes, allez dans le répertoire de travail
*ld_preload*. Vous devrez travailler sur trois fichiers:

  * hook.c
  * Makefile
  * run.sh

**Question 1** : Implémentez dans le fichier hook.c la fonction à l'origine du
                 problème repéré au sein du simulateur GPS mais cette fois-çi
                 sans erreur.

**Question 2** : Éditez le Makefile pour compiler *hook.c* sous la forme d'une
                 librairie partagée nommée *libhook.so* (s'inspirer de
                 *gps/src/lib/ptmx/Makefile*). Testez la compilation.

**Question 3** : Éditez le fichier *run.sh* pour utiliser LD_PRELOAD au moment
                 de lancer le simulateur et ainsi hooker le binaire avec la
                 librairie libhook.so. Exécutez run.sh : le simulateur ne doit
                 plus partir en segfault.

Nous avons ici hooké une fonction définie dans une librairie "utilisateur". On
peut réaliser la même opération sur les librairies systèmes. Par exemple, le
simulateur GPS utilise la fonction *printf* dès son lancement.

**Question 4** : Utilisez le *man* pour déterminer le prototype de la fonction
                 *printf* (expliquez comment vous utilisez *man* dans ce cas et
                 pourquoi). Comment est appelé ce type de fonction?

**Question 5** : Analysez *gps/src/bin/gps/gps.c* er repérez où se trouve le
                 gestionnaires de signaux. Décrivez les fonctions utilisez
                 ainsi que les signaux gérés.

**Question 6** : Hookez le simulateur pour que ce dernier ne puisse plus
                 être interrompu par le signal SIGINT (Ctrl-C) en
                 réimplémentant la fonction *printf* dans libhook.so. Pour
                 cela, utilisez la fonction *sigaction* pour mettre en place
                 un gestionnaire de signaux au sein même de la fonction
                 *printf*  réimplémentée.

**Question 7** : Comment faire pour interrompre le processus étant donné
                 que ce dernier ne répond plus au Ctrl-C? Citez deux méthodes.

**Question 8** : En regardant le fichier *gps/Makefile*, que pouvez-vous dire
                 de la règle *ok*? À quoi sert-elle et comment
                 fonctionne-t-elle?

### À retenir

  * comment utiliser le *man*
  * la mise en place d'un gestionnaire de signaux grâce à *sigaction*
  * l'utilité et le fonctionnement de *LD_PRELOAD*


## Exercice 3 : Terminal série (minicom)

Dans le cas d'une communication série avec du matériel, il est toujours
intéressant d'observer les trames. En effet, on peut par exemple comprendre un
protocole de communication grâce à de la rétro ingénierie. Pour cela, il existe
des programmes tel que *gtkterm*, *PuTTY*, *picocom* et bien d'autres. Lors de
ce TP, nous allons utiliser *minicom*.

**Question 1** : Grâce à l'aide en ligne de commande de *minicom* (option
                 *-h*), se connecter au port du simulateur GPS et observez les
                 trames passées.

**Question 2** : Naviguez dans l'aide de minicom (Ctrl-A Z) pour récupérer la
                 configuration du port de communication. Expliquez brièvement
                 la signification de ces paramètres.

**Question 3** : Utiliser minicom pour capturer un extrait de la
                 communication dans un fichier.