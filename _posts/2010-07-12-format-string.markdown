---
layout: post
title:  "Format string automatique"
date:   2010-07-12 18:25:00 +0100
categories: [security]
---
Cet été j'ai décidé d'apprendre le python, et pour que se soit fun je me suis mis en tête de faire un script qui automatise les format strings.

En plus c'est la mode:  
[http://esec.fr.sogeti.com/blog/index.php?2010/07/09/88-exploitation-de-format-string-avec-metasm](http://esec.fr.sogeti.com/blog/index.php?2010/07/09/88-exploitation-de-format-string-avec-metasm)
[http://sh4ka.fr/codes/fmt_string_builder.html](http://sh4ka.fr/codes/fmt_string_builder.html)
ou pas:  
[http://www.ouah.org/REMOTEFMT-HOWTO](http://www.ouah.org/REMOTEFMT-HOWTO)
[http://nibbles.tuxfamily.org/?p=887](http://nibbles.tuxfamily.org/?p=887)
[http://www.redspin.com/blog/2009/11/25/automatic-format-string-exploitation/](http://www.redspin.com/blog/2009/11/25/automatic-format-string-exploitation/)

Le script se décline en 3 fichiers:  
main.py  
warper.py (détecte ou se situe les arguments faillibles du programme et où placer le payload)  
frmtStr.py (exploite la format string)  

La format string est exploitable dans la cas où:
* On est sur une archi x86 (32bit), sous linux
* La pile est exécutable
* Pas d'aslr (Quoi qu'avec un gros padding/nopsleed ca passe)
* La section dynamic est +W
* Le programme vulnérable n'est pas trop verbeux (ca peut fausser la détection)

En bref sur un wargame c'est parfait (ca permet de gagner du temps), sur un programme réel ca l'est moins.

[https://www.youtube.com/watch?v=InfAAmJqMFI](https://www.youtube.com/watch?v=InfAAmJqMFI)

Je vous conseille de regarder la vidéo en fullscreen (pas super lisible).
