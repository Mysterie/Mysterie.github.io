---
layout: post
title:  "Write Up – NDH 2010 – VM Windows"
date:   2010-06-24 19:40:00 +0100
categories: [security]
---
Pour les personnes n'ayant pas participé au capture the flag ou pour ceux qui n'ont pas exploité les services présents sur la machine Windows, je mets à disposition les sources des épreuves que j'ai réalisées et une solution pour chacunes d'elles.

Premier point d'entrée, utilisateur `toto` mot de passe `azerty`.

Deuxième point d'entrée, un remote buffer overflow avec réécriture du `SEH`. Sauf que sur la VM le binaire était full [safeSEH](https://msdn.microsoft.com/en-us/library/9a89h429.aspx), alors que je l'ai testé/exploité sur ma box, je me suis trompé à la compilation, bref désolé pour ceux qui ont passé du temps dessus.

Troisième point d'entrée, sur le bureau de chaque challenger se trouvait un pdf. Le lecteur par défaut était adobe reader 8.0. Le pdf en question était [crafté](http://seclabs.org/origami/) de façon à exploiter la faille [cve-2009-0927](http://seclabs.org/origami/).  
J'ai ensuite utilisé un [shellcode windows trouvé sur le net](http://www.shell-storm.org/shellcode/files/shellcode-162.php) et retravaillé pour télécharger et exécuter un malware se trouvant à l'url `http://192.168.3.200/tmp.exe`.  
Étant donné les quelques problèmes de mise en place du CTF, l'url n'a pas été disponible tout de suite. Quatre équipes se sont infectées mais il est possible que bien plus d'équipes aient ouvert le pdf.

Le shellcode modifié :  
source ici

Le pseudo botnet :  
source ici

Le botnet réécrit l'[IAT](http://en.wikipedia.org/wiki/Portable_Executable#Import_Table) d'`explorer.exe` et de tous les processus lancés par explorer, afin de se cacher (hook de `findFirstFile` / `findNextFile`) et se connecte à `irc`. Malgré le fait que le pdf soit sur le bureau et que quatre équipes soient infectées (dont l'équipe gagnante) personne n'a creusé, et personne n'a rejoint irc pour essayer de prendre le contrôle du bot, donc j'ai fait mumuse tout seul:  
screen1  
screen2  
screen3

Viens ensuite quelques épreuves plus triviales comme un serveur java qui sérialise l'objet qu'il reçoit. Cet objet lit un fichier contenant de l'ascii art et renvoie son contenu sur le réseau, le but étant de modifier l'objet afin d'afficher un fichier contenant le hash de l'utilisateur ascii sur chaque machine.  
source ici  
Il y avait aussi deux épreuves web mais je n'en suis pas l'auteur donc je n'en parlerai pas. :)

Et enfin un "[Local Privilege Escalation](https://en.wikipedia.org/wiki/Privilege_escalation)", le binaire checksum.exe communique avec un driver. Il envoie le contenu du fichier qu'on lui donne au driver, celui-ci fait un [checksum](http://fr.wikipedia.org/wiki/Somme_de_contr%C3%B4le) des données et les renvoie au programme. La communication entre eux se fait par la méthode [METHOD_NEITHER](http://msdn.microsoft.com/en-us/library/ms809962.aspx#drvrreliab_topic3).  
Les challengers ont accès en lecture à checksum.sys. Ils pourront remarquer que le driver contient des failles dont un déréférencement de pointeur (Adresse du buffer d'output non vérifiée). Les symboles de windows 2003 sp2 étaient disponibles en ftp.   
sources ici  
Une petite difficulté était présente, comme le font pas mal d'antivirus le code `ioctl` de gestion du checksum était aléatoire. En analysant le driver ou l'exécutable on retrouvait facilement le code. Le but était de freiner les challengers en cas d'exploitation massive, mais personne n'a exploité cette épreuve. :(

That's all folks !
