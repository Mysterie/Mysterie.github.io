---
layout: post
title:  "VMWare Cloudburst"
date:   2009-08-05 20:48:00 +0100
categories: [security]
---
Le [document](http://www.blackhat.com/presentations/bh-usa-09/KORTCHINSKY/BHUSA09-Kortchinsky-Cloudburst-PAPER.pdf) sur la faille VMWare découverte par [Kostya](http://expertmiami.blogspot.com/) est enfin sorti. Le principe est de s'évader de la machine qui est virtualisée (guest) pour exécuter du code sur la machine hôte (host). La faille en elle-même est située dans le filtrage des requêtes vidéos. Petite explication:

VMWare virtualise une carte vidéo appelée “VMWare SVGA II”, on peut la retrouver par le biais du bus PCI. Pour plus d'explications sur le bus PCI, je vous conseille [cet article](http://rce.servhome.org/blog/?p=1). La carte vidéo a pour Vendor ID: `0x15ad` et Product ID: `0x0405` et est composée de 3 plages mémoires:

* La première représente la plage de port I/O (Port mapped I/O). Pour simplifier, ce sont grâce à elles qu’on pourra communiquer avec la carte graph’ virtuelle à l'aide d'instruction assembleur IN/OUT.
* La deuxième est généralement la portion de mémoire la plus grande représentant le frame buffer, c'est-à-dire l'espace mémoire en RAM réservé à la mémoire vidéo. (Chaque pixel ne correspond pas forcément à 4 bytes dans cette mémoire, mais c’est généralement le pitch par défaut sur la plupart des box).
* La troisième est dénommée `SVGA FIFO`. C’est celle qui nous intéresse. Cet espace mémoire est utilisé pour stocker des commandes vidéos. Elles sont filtrées du côté de l’host (`vmware-vmx.exe`).

Les quatre premiers DWORDs de la mémoire SVGA FIFO représentent des offsets précis:
```
SVGA_FIFO_MIN
SVGA_FIFO_MAX
SVGA_FIFO_NEXT_CMD
SVGA_FIFO_STOP
```

`MIN` et `MAX` représentent la plage mémoire fifo réellement utilisée, ces valeurs ne changent pas quand l'OS est chargé. `NEXT_CMD` représente la prochaine commande vidéo à exécuter. `STOP` représente le point de synchronisation, il se situe à un offset supérieur à `NEXT_CMD`. Quand `NEXT_CMD` est égal à `STOP` ou dépasse `MAX` alors la carte se resynchronise et attend ensuite de nouvelles requêtes.

```
<- début de la ram dédier a la mémoire FIFO

<- SVGA_FIFO_MIN

<- SVGA_FIFO_NEXT_CMD
<- SVGA_FIFO_STOP

<- SVGA_FIFO_MAX
```

Dans cette mémoire se trouve les commandes, elles suivent un schéma bien précis. On va prendre l’exemple de la commande `RECT_COPY` qui prend 6 paramètres (`Source X, Source Y, Dest X, Dest Y, Width, Height`) et deux paramètres supplémentaires appelés capacities dont je n'ai pas trouvé l'utilité. Ce qui donne:

```
+0x0    3 // RECT_COPY (id)
+0x4    0 // Source X
+0x8    0 // Source Y
+0xc   10 // Dest X
+0x10  10 // Dest Y
+0x14  50 // Width en px
+0x18  50 // Height en px
+0x1c  ?? // cap1
+0x20  ?? // cap2
```

J'essaye donc d'écrire dans la mémoire FIFO pour faire pointer `SVGA_FIFO_NEXT_CMD` sur une commande craftée. Mais vu la rapidité des traitements de la mémoire vidéo, il va falloir passer par des commandes `IN/OUT` pour bloquer les traitements vidéos. Créer notre commande puis relancer le traitement. Pour cela, on va communiquer avec la carte graph virtualisée. On a besoin de deux données: l’index port et le value port. Pour le SVGA de type 2 les ports se trouvent comme ceci:

```
Dans svga_reg.h on as les define:
#define SVGA_INDEX_PORT		0x0
#define SVGA_VALUE_PORT		0x1
#define SVGA_BIOS_PORT		0x2
#define SVGA_NUM_PORTS		0x3
#define SVGA_IRQSTATUS_PORT	0x8

IndexPort = (IOportsBase & PCI_ADDRESS_IO_MASK ) + SVGA_INDEX_PORT;
ValuePort = (IOportsBase & PCI_ADDRESS_IO_MASK ) + SVGA_VALUE_PORT;
```

On peut donc ensuite envoyer nos commandes... Sachant qu’elles ne sont pas bien filtrées: “For example if Dest X + Width falls out the screen, the operation is said to “clip” and is aborted. Yet the comparisons done on the DWORD and the results of the additions are signed. This opens the door to some malicious usage of the command.”

Pendant 2 jours j’ai bloqué là dessus. Puis j’ai décidé de contacter Kostya pour plus de détails (et je le remercie encore pour son aide). Pour récupérer la mémoire de l’host, il suffit que la commande `RECT_COPY` ait comme paramètre `Source X > 0` et `Source X + width < 0`. Il faut donc jouer avec la valeur `0×80000000` pour que la comparaison signée échoue. Je ferai un article sur l’exploitation du bug. En tout cas le travail de Kostya est vraiment impressionnant surtout du côté de la communication entre l’host et le guest (MOSDEF over Direct3D) pour la fiabilisation de l’exploit.

Lire la suite...

Le code:  
[Du driver xfree86 de VMWare](http://ftp.kaist.ac.kr/NetBSD/NetBSD-current/xsrc/xfree/xc/programs/Xserver/hw/xfree86/drivers/vmware/)  
De mon driver (sans exploit pour le moment)  
Quelques liens:  
[Documentation sur le driver XF86 & SVGAII](http://sourceware.org/ml/ecos-devel/2006-10/msg00008/README.xfree86)  
[Ring -3 Rootkits & Intel BIOS attack](http://invisiblethingslab.com/itl/Resources.html)  
[zf05 Mass 0wn4g3](http://www.milw0rm.com/papers/360)  
