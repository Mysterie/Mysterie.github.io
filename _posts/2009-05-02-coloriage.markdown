---
layout: post
title:  "Aujourd'hui c'est coloriage."
date:   2009-05-02 12:50:00 +0100
categories: [security]
---
Je me suis demandé comment fonctionnait l'affichage sous windows (afficher un pixel à l'écran). Quand j'ai fait mes premiers pas en assembleur je me suis amusé à afficher une lettre, c'était sous win98 grâce au mode [https://fr.wikipedia.org/wiki/Color_Graphics_Adapter](cga) qu'on obtient avec ces instructions [http://www.emu8086.com/assembly_language_tutorial_assembler_reference/8086_bios_and_dos_interrupts.html#int10h_06h](INT 10h / AH 6h). Puis j'ai essayé de lancer le même code sous XP et là c'est le drame, rien. Depuis windows xp & co il y a eu pas mal de changements et mon programme ne fonctionne plus. En réalité on ne peut plus lire/écrire directement dans la mémoire physique réservée au mode cga car:

* Les systèmes d'exploitation actuels utilisent un mécanisme de [https://fr.wikipedia.org/wiki/M%C3%A9moire_virtuelle](mémoire virtuelle).
* L'espace mémoire en question se situe après l'adresse 0x7ffeffff ([https://fr.wikipedia.org/wiki/Anneau_de_protection](Ring 0)).

Je me suis mis à la recherche de la mémoire vidéo, le mode cga n'étant plus utilisé depuis longtemps. Une des façons de procéder est d'écrire un driver (pour accéder au noyau) et d'envoyer des requêtes au bus pci pour trouver la carte graphique et ses caractéristiques. Je me suis basé sur [http://rce.servhome.org/blog/?p=1](un article de rAsM) à lire absolument si vous voulez comprendre le code. J'ai ajouté des [http://msdn.microsoft.com/en-us/library/ms801656.aspx](KeAcquireSpinLock) et [http://msdn.microsoft.com/en-us/library/ms801909.aspx](KeReleaseSpinLock) histoire de ne pas avoir de surprise avec les READ/WRITE_PORT.

[http://mysterie.fr/prog/blog/pcidisp.c](Le code du driver), Le résultat:

```
Base address #0 (BAR0) // domaines de port I/O
Physical [0x1401 - 0x1411]
Base address #1 (BAR1) // 1er partie de la mémoire vidéo
Physical [0xf0000000 - 0xf8000000]
Base address #2 (BAR2) // 2eme partie de la mémoire vidéo
Physical [0xec000000 - 0xec800000]
```

Bien sûr se sont des adresses physique, on peu ensuite utiliser la fonction [http://msdn.microsoft.com/en-us/library/aa932608.aspx](MmMapIoSpace) pour y accéder mais sa reviendrai à faire une copie de la mémoire physique vers la mémoire virtuelle et on se retrouve avec DEUX buffer vidéo en ram :/.   
Alors on vas essayer de retrouver le premier qui est déjà en mémoire. J'ai donc fait un programme en C qui affiche un pixel rose en haut à gauche de l'écran.

{% highlight C %}
//Rien de complexe pour le programme.
HDC hScreenDC;
hScreenDC = GetDC(0); // screen
DWORD color = 0x00FF00FF; // rose :}
SetPixel(hScreenDC, 0, 0, color);
ReleaseDC(0, hScreenDC);
{% endhighlight %}

Et ensuite je débugge tout ça, donc dans notre programme on commence par un appel a la fonction [http://msdn.microsoft.com/en-us/library/dd145078%28VS.85%29.aspx](SetPixel) qui se trouve dans gdi32.dll. La [http://fr.wikipedia.org/wiki/Graphics_Device_Interface](GDI) permet de faire le lien entre les applications et les pilotes graphiques. et à la fin de gdi32!SetPixel on as un appel à une petite routine assez spécifique:

{% highlight Assembly %}
mov eax, 111Ah
mov edx, 7FFE0300h
call dword ptr [edx] ; ntdll!KiFastSystemCall
{% endhighlight %}

Pour ceux qui n'ont pas compris la subtilité de ses quelques instructions, en fait on a un appel à KiFastSystemCall() avec comme argument 0x111A. [http://www.ivanlef0u.tuxfamily.org/?p=22](Puis vient le SYSENTER et on entre dans le noyau) (pas directement bien sûr), la fonction KiFastCallEntry() récupère notre argument puis on passe par la fonction KiSystemService() qui va se charger de trouver la fonction équivalente a 0x111A. Ce numéro ne représente pas vraiment une fonction, je pense qu'il y a un 0x1000 AND 0x111A, pour indiquer si on a affaire à la [http://fr.wikipedia.org/wiki/System_Service_Dispatch_Table](SSDT) ou à la [http://0vercl0k.blogspot.com/2009/03/flirt-with-session-space.html](SSDT Shadow).

```
0: kd> dd nt!KeServiceDescriptorTableShadow l 0x8
805614c0 804e48b0 00000000 0000011c 80517fc4
805614d0 bf999c00 00000000 0000029b bf99a910
```

On voit bien que le nombre de fonction dans la SSDT Shadow est de 0x29b, donc 0x111A c'est trop grand. Par contre après avoir viré ce 0x1000 on se retrouve avec le numéro 0x11A et là ça le fait.

```
0: kd> dd bf999c00 l 0x11b
...
bf99a060 bf949a50 bf841b72 bf8751a3
0: kd> ln bf8751a3
(bf8751a3) win32k!NtGdiSetPixel
```

Le nom de la fonction est assez explicite, elle fait appel a de moult autre fonctions dans win32k.sys. je trace un peu tout ça pour en ressortir les fonctions qui nous intéressent.

```
// arbre d'appel
win32k!NtGdiSetPixel
 -> win32k!SURFACE::pfnBitBlt
  -> win32k!SpBitBlt
   -> win32k!OffBitBlt
    -> win32k!CLIPOBJ_vOffset
     -> win32k!EngBitBlt
      -> win32k!PDEVOBJ::vSync
       -> win32k!vDIBSolidBlt
        -> win32k!vSolidFillRect1
// code qui rox
mov eax,dword ptr [ebp+18h] ; eax=00ff00ff code de la couleur
mov dword ptr [edi],eax ; edi=f9234000 adresse pointant dans la mémoire vidéo.
```

Je vérifie que cette adresse corresponde bien à l'adresse physique 0xf0000000.

```
0: kd> !vtop 0 f9234000
X86VtoP: Virt f9234000, pagedir d4a6000
X86VtoP: PDE d4a6f90 - 01014163
X86VtoP: PTE 10148d0 - f000017b
X86VtoP: Mapped phys f0000000
Virtual address f9234000 translates to physical address f0000000.
```

Woot! Je ne sais pas comment procède windows pour retrouver l'adresse 0xf9234000 (si quelqu'un peut m'éclairer). Par contre je m'aperçois qu'après f9234000+1D4C00 (1D4C00 = 800*600*4 la résolution de l'écran) la mémoire est à zéro, de 0xf9408c00 à 0xf9434000 ce qui fait 177ko de libre. Vu la place je pense à la technique du [http://www.phrack.org/issues.html?issue=63&amp;id=8&amp;mode=txt](Shadow Walker), le principe serait de faire croire au système que les pages non utilisées de la mémoire vidéo sont swappées. On pourrait y cacher du code ou des datas. Mais cette technique commence à dater et nécessite un hook dans l'[http://fr.wikipedia.org/wiki/Interrupt_Descriptor_Table](idt) ce qui peut être [http://www.ivanlef0u.tuxfamily.org/?p=262](détectable). Et dans le cas où l'utilisateur changerait sa résolution d'écran il y aurait surement un conflit, dommage...
