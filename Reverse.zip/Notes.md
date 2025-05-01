
> *Pour cette introduction au reverse, on travaillera sur des **ELF** principalement.*

## Table des matières



---
## Fonctionnement d'un programme
Déjà, on peut définir ce qu'est un programme de façon très simpliste.
> C'est un **bout de code**, transformé en **fichier** **exécutable** par l'**ordinateur**.
> ![[Pasted image 20250430203753.png]]

Il est aussi utile de définir les différents types de langages, on en retrouve deux :  
- *Langages interprétés* : Ceux dont le code est **interprété ligne par ligne** par l'interpréteur. C'est à dire que l'**interpréteur** ne sait pas à l'avance ce qu'il va rencontrer, et qu'en Python par exemple, on ne voit les erreurs qu'au lancement du programme.
  Ils sont par ailleurs **plus simples à reverse** que l'autre type de langage.
- *Langages compilés* : Comme le java, le code est lu dans son **entiereté**, et si la moindre erreur est détectée, cela ne compile même pas. C'est plus dur à reverse, car lors de la compilation on perd pas mal de données (noms des fonctions, structures, objets, énumération...)


Maintenant, pourquoi la **compilation** ? Evidemment, c'est pour les langages qui n'ont pas directement d'**interpréteur**, ce qui signifie que c'est à l'ordinateur de comprendre les instructions. Mais l'ordinateur, il ne comprend pas directement le C, le Java.  
C'est là qu'intervient la **compilation**, elle **transforme** le **code source**, qui est du **texte**, en **assembleur** !

Un tout petit exemple pour définir ce que permet la **compilation** :  
```c
#include "stdio.h"

int main()
{
	puts("Hello World!\n");
}
```
Après compilation, il devient ceci en assembleur :  
![[Pasted image 20250430205634.png]]

Donc oui, on y comprend pas grand chose pour l'instant.

> Il est important de noter qu'un processus est exécuté en mémoire. C'est à dire que les **instructions exécutées** par le processeur lorsqu’un processus est lancé sont **situées en mémoire**.
> Si on lance un **exécutable**, qu'on le laisse tourner, mais qu'entre temps on supprime le fichier de cet exécutable, on pourra toujours utiliser le programme qui est lancé. C'est parce que **tout le processus et ses instructions** sont en **mémoire**

---
## Format d'un ELF
On va maintenant détailler le format d'exécutable Linux afin de comprendre les parties qui le composent ainsi que leur utilité.

1. L'**entête ELF** : Commence par les *magic bytes* (.ELF) et contient les informations générales du programme sur l'architecture (32/64 bits, compilé pour Intel, ARM...).
2. **Program header table** : Liste les **segments** du programme.
3. **Section header table** : Liste les **sections** du programme.
4. Le **reste** : Les instructions, les données, ...

#### L'entête

```bash
┌──(naxyl㉿kali)-[~]
└─$ xxd /usr/bin/xclock | head
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 6049 0000 0000 0000  ..>.....`I......
00000020: 4000 0000 0000 0000 d0cf 0000 0000 0000  @...............
00000030: 0000 0000 4000 3800 0d00 4000 1f00 1e00  ....@.8...@.....
00000040: 0600 0000 0400 0000 4000 0000 0000 0000  ........@.......
00000050: 4000 0000 0000 0000 4000 0000 0000 0000  @.......@.......
00000060: d802 0000 0000 0000 d802 0000 0000 0000  ................
00000070: 0800 0000 0000 0000 0300 0000 0400 0000  ................
00000080: 1803 0000 0000 0000 1803 0000 0000 0000  ................
00000090: 1803 0000 0000 0000 1c00 0000 0000 0000  ................
```

### Segments et sections
> Un **segment** est une zone mémoire qui contient **plusieurs sections** qui ont les mêmes attributs (ex : Lecture seule, exécutable …).  
> ![[Pasted image 20250501143617.png]]

Mais qu'est ce qu'une **section** plus précisément ?
On retrouve plusieurs **types** de **sections** (liste non exhaustive) :  
- `.init` : La section d'**initialisation**.
- `.text` : Celle qui contient le **code**.
- `.fini` : La section de **fin**.
Et selon le type de **section**, les instructions contenues doivent avoir des **attributs différents** de **lecture** et **exécution**.
Et bien, ces trois sections ci-dessus, elles doivent posséder les **mêmes** attributs de **lecture** et **exécution**, ce qui fait qu'elles se retrouvent dans le **même segment** : Un segment ayant les **droits** **RX**.

Puis pour d'autres types de sections, comme celles qui contiennent des **données modifiables** :  
- `.bss` : Données **initialisées** à 0.
- `.data` : Données **initialisées modifiables** comme des **variables globales**, **statiques**..
On les retrouve donc dans un **segment** ayant les **droits** **RW**.

Et pour finir cet exemple, un autre type de **section** : 
- `.rodata` : Données **non modifiables uniquement**. (Une string par exemple) 
On la retrouve dans un **segment** qui aura les **droits R**.


> On peut afficher les différents **segments** d'un **programme ELF** avec `readelf -l programme`  :  
> ![[Pasted image 20250501145726.png]]

Pour nous aider un peu à voir quels sont les segments intéressants, on a cette petite image :  
![[Pasted image 20250501150027.png]]
- le segment 🔴 contient du code exécutable et doit donc avoir les attributs de **lecture** et d’**exécution**
- le segment 🟣 contient des données qui ne sont qu’en **lecture seule** ( comme des chaînes de caractères qui n’ont pas besoin d’être modifiées)
- le segment 🟢 contient des données modifiables mais n’ayant pas besoin d’être exécutées : cette zone mémoire n’aura besoin que des droits de **lecture** et **écriture**
- le segment 🔵 contient la section `.dynamic` et contiendra, comme son nom l’indique, les données allouées dynamiquement. Comme les allocations réalisées par `malloc`. C’est ce que l’on appelle le **tas** (ou **_heap_**).
- Le segment 🟡 contient la **pile d’exécution** appelée **_stack_**. C’est une zone mémoire qui contiendra notamment les variables locales et qui fonctionne, comme son nom l’indique, sous forme de pile : **Premier arrivé, dernier servi**.

> La _heap_ est désignées par “tas” en français mais cela **n’a rien à voir** avec la structure de données nommée [tas](https://fr.wikipedia.org/wiki/Tas_\(informatique\)). On l’appelle tas car il y a un tas de choses dedans allouées dynamiquement et qui sont souvent hétérogènes.

### Agencement en mémoire
Maintenant qu'on a compris dans les grandes lignes les **segments** et **sections**, comment est-ce-que c'est **structuré en mémoire** une fois le **programme exécuté** ?
![[Pasted image 20250501150822.png]]
Et quelque chose de très important à noter sur le **tas**.

> Etant donné que le **tas** est dédié à **l’allocation dynamique** de données, c’est un **segment** que l’on ne voit pas dans le programme **tant qu’il n’est pas exécuté** car cette zone mémoire est **créée dynamiquement au lancement du programme**.
> 
> Il en est de même pour la **pile** qui est une **zone mémoire allouée** lors du **lancement du programme**.

Pour le schéma, les adresses de début et de fin utilisées sont juste à titre indicatif, c'est pour appuyer le fait que l'on représente les **adresses basses vers le haut** et les **adresses hautes vers le bas**.


#### Sémantique
Au final, on a vu les **sections**, **segments**, **agencement en mémoire**, mais pourquoi ? 

Pour le reverse, on va devoir aller chercher des données à différents endroits, et en sachant maintenant que tel **segment** contient telle **permission**, on saura où se trouve ce qu'on cherche. Par exemple, on ira pas chercher du **code** dans la **section de données** ou **modifier la valeur** d'une **variable globale** depuis la **section de code**.

On peut faire un petit récap des différentes sections.

##### Zone de code

```sh
.init .plt .plt.got .text .fini
```
Les **instructions** des différentes fonctions dont le `main`.

##### Zone des données en lecture seule

```sh
.rodata .eh_frame_hdr .eh_frame
```
Les données non modifiables telles que les chaînes de caractères présentes en tant qu’arguments pour les fonctions `puts`, `printf` … Exemple : `Hello world!\n`

##### Zone des données modifiables

```sh
.init_array .fini_array .data.rel.ro .dynamic .got .got.plt .data .bss
```
Les **variables globales** (déclarées en dehors de toute fonction), les **variables statiques** (déclarée avec le mot clé `static`) comme `static int var;`…

##### Tas

```sh
.dynamic
```
Les **variables allouées dynamiquement** avec `malloc` (en C) ou `new` (en C++). Ce sont des variables dont on ne connaît pas la taille avant l’exécution du programme tel que le nom de l’utilisateur. Exemple : `char *username = malloc(n);`

##### Pile

```sh
           
```
**Les variables locales**, c’est-à-dire la majorité des variables que l’on utilise. Il s’agit de celles qui ne sont pas allouées dynamiquement et sont déclarées au sein des fonctions sans le mot clé `static`. Exemple : `int a; int b = 0x213;` …

---

## L'assembleur

### Versions et processeur

Evidemment, on doit passer par cette étape, comprendre les registres, les instructions, et tout ce qui suit.

> **L’assembleur** ou **langage machine**, est le langage le plus bas niveau qu’il puisse y avoir. Par “plus bas niveau” on entend qu’il s’agit d’un langage compris **directement** par l’ordinateur et plus précisément par le **processeur**.
> 
> En fait, c’est le **processeur** qui se charge **d’exécuter** toutes les instructions assembleur. Que ce soit les accès en mémoire vive (RAM), les calculs, les appels de fonctions, bref, c’est lui **LE cerveau** de l’ordinateur.

Puisque l'assembleur est le langage que comprend directement le **processeur**, on retrouve plusieurs **versions d'assembleur** pour correspondre aux différentes **versions de processeurs**. (x86, ARM, MIPS, RISC-V, ...)

Un petit exemple pour illustrer ces **différences**. Si on prend ce **code** :  

```c
int rien()
{
	return 0;
}
```

On peut voir comment il va être **compilé en assembleur** grâce à ce [site](https://godbolt.org/) selon les **différentes architectures** de processeur. 

###### x86_64 alias AMD64

```asm
	push rbp
	mov rbp, rsp
	mov eax, 0
	pop rbp
	ret
```

###### ARM

```asm
	mov     w0, 0
	ret
```

etc..

Et au sein d'une même **architecture**, on retrouve plusieurs **versions**. Par exemple, l'assembleur **x86** d'Intel et AMD est une **version 32 bits**, alors que la version **x86_64** est une version 64 bits.

Ce que ça change, c'est au niveau de la taille des données que peut traiter le processeur. Sur une architecture **64 bits**, il pourra traiter **64 bits par 64 bits**, alors qu'il devrait faire **32 bits par 32 bits** sur du **x86**.

Et ça, ça s'explique par la **taille des registres** ! On en compte dans l'ordre de la **dizaine**, voire **vingtaine**.

> Les **registres** sont des petites zones mémoire **dans le processeur**. Cela lui permet de faire certaines opérations (calculs, déplacement de valeurs, stockage …) sans avoir à passer par la **RAM** qui se situe plus loin, ce qui implique des **performances moins élevées** dans le cas où la mémoire serait utilisée.

