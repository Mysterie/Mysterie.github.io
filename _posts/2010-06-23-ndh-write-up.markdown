---
layout: post
title:  "Write Up – NDH 2010 – Crackme"
date:   2010-06-23 17:40:00 +0100
categories: [security]
---
Le 19 et 20 Juin a eu lieu la 8eme édition de la Nuit du hack. Pour cette occasion, j'ai réalisé différentes épreuves dont des crackmes accessibles aux personnes participant au challenge public (par le wifi) et aux participants du Capture The Flag.  
Dans la section Crackme j'ai réalisé les niveaux 1, 3 et 4.

__Crackme Niveau 1:__

Le crackme n'a aucun mécanisme d'anti-debug. Ce qui lui est donné en entrée est comparé directement à une chaîne en mémoire. Rien de bien compliqué en soit:

{% highlight C %}
GetWindowText(hEdit, text, 255);
theSecret[0]++;
theSecret[2]++;
theSecret[4]++;
if(strlen(text) > 0 && !strcmp(text, theSecret)) {
    ShowWindow(hVrai, SW_SHOW);

    strcpy(text, getMD5(text));
    SetWindowText(hMd5, text);
    ShowWindow(hMd5, SW_SHOW);
    ShowWindow(hFaux, SW_HIDE);
} else {
    ShowWindow(hVrai, SW_HIDE);
    ShowWindow(hMd5, SW_HIDE);
    ShowWindow(hFaux, SW_SHOW);
}
theSecret[0]--;
theSecret[2]--;
theSecret[4]--;
{% endhighlight %}

La source du crackme:  
cm1/

__Crackme Niveau 3:__

Le mot de passe du programme n'est pas stocké dans sa mémoire ou dans le fichier exécutable. Au chargement, il crée 8 plages mémoires contenant des opcodes de saut assembleur (5 octets). Chaque plage contient 256 sauts, chaque saut équivaut à un caractère ascii. Dans chacune des plages un seul saut est valide et pointe vers une routine de décryptage. Les autres sauts pointent indirectement vers une routine qui affiche mot de passe faux.  
J'utilise aussi l'instruction rdtsc pour calculer le temps effectué entre le premier saut et le dernier afin de vérifier que le programme s'exécute en moins d'une seconde. Sinon c'est que le programme est débuggé et donc j'affiche mot de passe faux dans tous les cas.  
La source du crackme:  
cm2/

__Crackme Niveau 4:__

Ce niveau est destiné à des personnes ayant l'habitude de ce genre d'épreuve. La difficulté principale de ce challenge est donc de débugger le programme (l'analyser). J'ai mis beaucoup de routine d'anti-débug. Tout d'abord à l'entrée du programme, j'ai incrusté des instructions assembleur pour modifier la signature de l'exécutable (ces instructions ne sont jamais exécutées). Le but est de faire croire que l'exécutable est packé, c'est-à-dire qu'il est compressé par un autre logiciel.  
Ensuite, j'appelle une fonction spéciale un peu partout dans le code de mon programme. (junk code)

{% highlight C %}
void brack() {
    __asm {
        _emit 0xEB;  // 0xEB01 = jmp short relatif saute au dessus
        _emit 0x01;  // de l'octet 0x1D. Donc ne fait rien au final
        _emit 0x1D;  // 0x1D fait croire au debugger que l'instruction qui suit est SBB
    }
}
{% endhighlight %}

Le programme crée ensuite un thread. La fonction du thread est la suivante:

{% highlight C %}
while(1) {
    brack();
    Sleep(1000);
    __try {
        __asm {
            int 3;
        }
    }
    __except(eps = GetExceptionInformation(), EXCEPTION_EXECUTE_HANDLER) {
        brack();
        isDbg = false;
    }
    if(isDbg) {
        quit();
        break;
    }
    isDbg = true;
}
{% endhighlight %}

Dans le cas où un débugger serait présent au démarrage du programme ou s'il s'attache au programme durant son exécution, l'instruction `int 3` (point d'arrêt) est exécutée le débugger la récupère. Le programme détecte ainsi qu'un débugger est présent et le mot de passe n'est jamais révélé.

La fonction `IsDebuggerPresent` est aussi appelée dans le programme. Elle est appelée indirectement (récupération de l'adresse de la fonction dynamique).

{% highlight C %}
GetProcAddress(GetModuleHandle("Kernel32.dll"), "IsDebuggerPresent");
{% endhighlight %}

Et enfin, j'utilise l'instruction `rdtsc` dans le programme pour vérifier que le temps d'exécution de celui-ci n'est pas trop long.

Si le challenger contourne toutes ces protections alors le programme alloue une plage mémoire, écrit à l'emplacement de cette plage des octets spécifiques (fonction de déchiffrement encodé), puis la décode avec un XOR. Lorsque l'utilisateur entre son mot de passe, la fonction déchiffrée effectue plusieurs opérations aritmétriques sur le mot de passe:
* Longueur de la chaîne.
* Est-ce que la deuxième lettre est égale à la 5eme?
* OU / ET logique, XOR etc...

Par un calcul mathématique ou un peu de logique on retrouve assez simplement le passe. Il n'y a aucune collision, une seule solution est valide.  
La source du crackme:  
cm3/

__Au niveau des résultats sur 180 participants (public/ctf):__
* Level1 10 validations
* Level3 5 validations
* Level4 3 validations

Des solutions sont déjà disponibles sur le net:  
[http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-1](http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-1)  
[http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-2](http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-2)
[http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-3](http://w4kfu.free.fr/blog/index.php?post/2010/06/20/Write-up-NDH-2010-crackme-lvl-3)

Et si vous êtes férus de crackme [ce week-end Eset NOD32 organise un challenge](www.crackme.fr).
