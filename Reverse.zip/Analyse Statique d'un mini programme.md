> L'**analyse statique** signifie qu'on ne va ni **exécuter** ni **déboguer** le programme.

Ici, on va travailler avec du **x86**, donc **32 bits**. Même si on a un processeur **64 bits**, il est rétrocompatible et peut donc exécuter des instructions sur **32 bits** (puisqu'il a des registres de **64 bits**, qui se déclinent en **sous registres** qui peuvent faire **32 bits**, et ainsi de suite..).

On utilisera ce petit programme en C, tout simple !  

```c
int main()
{
	int a = 2;
	int b = 3;

	return a + b;
}
```


Maintenant, on va passer à la **compilation**. On installe d'abord ce paquet :  

```sh
sudo apt-get install gcc-multilib
```

Puis, on peut **compiler** avec ces options.  

```sh
gcc -m32 -fno-pie main.c -o exe
```

- `-m32` : Pour compiler en **32 bits**.
- `-fno-pie` : *A détailler plus tard...*
- `-o` : Le nom du **fichier d'output**, donc la **version compilée** de notre code, notre **exécutable**.

Et on lui donne bien les permissions d'être exécuté !  

```sh
chmod +x exe
```

Pour obtenir notre premières informations sur l'exécutable, on peut utiliser la commande **`file`** : 

```sh
┌──(naxyl㉿kali)-[~/Desktop/Reverse/ReverseZip/AnalyseStatiqueMiniProgramme]
└─$ file exe 
exe: ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=d2b00fbb13a07b8a98dea2bcaf274eae7072d113, for GNU/Linux 3.2.0, not stripped
```

- `ELF` ➡️ Le programme est un **ELF**, logique, nous sommes sous Linux.
- `LSB` ➡️ _Least Significan Bit_. Il s’agit donc d’un programme **_Little Endian_**.
- `dynamically linked` ➡️ les bibliothèques utilisées sont **chargées dynamiquement**, à l’exécution, et ne sont donc pas présentes **statiquement** dans le programme
- `not stripped` ➡️ les **symboles** ne sont pas supprimés et sont encore présents dans le programme

> 💡 Les **symboles** sont des informations, principalement des chaînes de caractères, présentes dans un programme et sont notamment utilisées pour le déboguer. Elles ne sont pas pas nécessaires et peuvent être supprimées avec la commande `strip`.
> 
> Par exemple, les **noms de variables globales** et les **noms de fonctions** font partie des symboles.

Pour lancer le programme, et obtenir le code de retour (*la dernière ligne de notre fonction*), on peut utiliser cet enchaînement de commandes.  

```sh
./exe ; echo $?
5
```

On a bien **5**, car c'est le résultat de **a** + **b**.

---

### Désassembleur
Maintenant qu'on a pu **exécuter** notre programme, on va pouvoir entamer le **reverse**.

On l'a vu avant, on a le **segment** avec les permissions **RW** qui contient le **code**, plus précisément les **opcodes**. Pour rappel, c'est ce segment :  

```sh
.init .plt .plt.got .text .fini
```

Et autre petit rappel, les **opcodes** (en bleu), c'est les **instructions assembleur** (en vert) mais **encodées** en **hexadécimal**, car le processeur ne comprend pas direct `push rbp` :    

![[Pasted image 20250501175737.png]]

Maintenant, on va pas aller lire directement dans le **segment** qui comporte le **code**, car on trouverait des **opcodes**, mais impossible de lire ça comme ça en hexadécimal.
C'est là qu'interviennent les désassembleurs comme [IDA](https://hex-rays.com/ida-free) qui permettent de transformer les **opcodes** en **instructions assembleur** afin qu'on puisse comprendre en tant qu'humains.

Pour l'apprentissage, on utilisera justement **IDA**, car il y a une version gratuite qui existe. 

### IDA

On ouvre donc le fichier ``exe`` avec **IDA**, et on peut commencer !  

Il y a cette partie de [reverse.zip](https://reverse.zip/posts/introduction_au_reverse_partie_5/#apprendre-%C3%A0-utiliser-ida-freeware) qui explique l'interface, inutile de la redétailler.

On a donc la **vue des fonctions** à gauche, on retrouve bien notre fonction ``main``, mais contrairement à ce qu'on pourrait croire, ce n'est pas la première appelée. 

Avant ça, on a ``start``, suivie de `__libc_start_main` qui est exécutée pour appeler le `main` en lui fournissant les bons arguments `argv` (*nombre d'arguments*) et `argc` (*tableau qui contient les arguments*).

`__libc_start_main` provient de la bibliothèque standard `libc`. 
> La **libc** est la bibliothèque standard du C sous Linux, son équivalent sous Windows est **msvcrt.dll**.
> 
> Cette bibliothèque contient les **fonctions de base en C** telles que : `printf`, `puts`, `scanf`, `malloc`, `free`, `strcpy` …4

![[Pasted image 20250501185601.png]]

Maintenant, tentons de comprendre les **instructions assembleur** de la fonction `main`. On peut retrouver un résultat similaire à ce qui est affiché sur **IDA** avec ce [site](https://godbolt.org/) :   

```asm
main:
	push ebp
	mov ebp, esp
	sub esp, 16
	mov DWORD PTR [ebp-4], 2
	mov DWORD PTR [ebp-8], 3
	mov edx, DWORD PTR [ebp-4]
	mov eax, DWORD PTR [ebp-8]
	add eax, edx
	leave
	ret
```

Et on peut comprendre à quoi chaque section correspond via cette image :  

![[Pasted image 20250501191408.png]]

- Les zones **1** et **5** sont ce qu'on appelle le **prologue** et **épilogue** d'une fonction. On y reviendra plus tard.

- Les zones **2** et **3** correspondent à l'initialisation des deux variables **a** et **b**.

- La zone **4** correspond à l'addition `a+b`.

Mais là, on comprend pas vraiment ce que veulent dire les **instructions**, donc on va comprendre comment fonctionnent les **registres**.

---

### Registres
#### Registres principaux
Puisqu'on travaille sur du **x86**, on va voir les **registres principaux** à connaître, donc en **32 bits**.

| Nom du registre | Taille en bits | Utilisation usuelle                                                              |
| --------------- | -------------- | -------------------------------------------------------------------------------- |
| `eax`           | *32*           | **Stocker** la **valeur de retour** d'une **fonction**                           |
| `ebx`           | *32*           | Utilisations **diverses**                                                        |
| `ecx`           | *32*           | Utilisé en tant que **compteur** dans les **boucles**                            |
| `edx`           | *32*           | Utilisé lors des **multiplications** et **divisions**                            |
| `edi`           | *32*           | Utilisé comme **pointeur** vers une **zone mémoire** de **destination**          |
| `esi`           | *32*           | Utilisé comme **pointeur** vers une **zone mémoire source**                      |
| `ebp`           | *32*           | Utilisé comme **pointeur** vers la **base** de la **pile**                       |
| `esp`           | *32*           | **Toujours** utilisé comme **pointeur** vers le **haut** de la **pile**          |
| `eip`           | *32*           | **Toujours** utilisé comme **pointeur** vers l'**instruction courante** exécutée |
Comme on peut le voir, certains **registres** sont destinés à **toujours** faire la même chose, comme `esp`, `eip` et `eax`. Mais `eax` est quand même parfois utilisé autrement (*stockage, multiplication, division*).

Donc là, on vient de voir ça pour du **x86**, mais qu'en est-il pour l'**AMD64** ? 
On y retrouve les **mêmes registres**, mais également des supplémentaires de taille de **64 bits**. 

> En fait, en assembleur x86, il est possible d’utiliser **différentes tailles** pour un **même registre** selon ce que l’on souhaite faire. Par exemple, si je souhaite stocker un `char` dans le registre `eax`, je n’ai besoin que d’un octet (8 bits) : je n’aurai donc pas besoin de tous les **32 bits** ou **64 bits** qui sont présents dans le registre.

C'est pour ça qu'on retrouve les déclinaisons des registres. Par exemple, le registre `eax` en **32 bits** possède sa version **64 bits** pour de l'**AMD64** : `rax`. Il suffit de remplacer le `e` par un `r` pour passer du **32** au **64 bits**.  
![[Pasted image 20250501214809.png]]

> `AH` représente les 8 bits de poids fort (_High_) de `AX` `AL` représente les 8 bits de poids faible (_Low_) de `AX`


#### Les autres registres
##### EFLAGS (ou RFLAGS en 64 bits)

Ici, il s'agit d'un registre où chacun de ses **bits** a une **signification particulière** et représente un **état** bien précis du processeur.
Donc **chaque bit** représente en fait un **flag**, et chacun de ces bits peut être modifié selon les circonstances.

Par exemple, on a le flag `ZF`. Lorsqu'une **opération** impliquera un **résultat nul**, il sera à **1**. Et quand ce résultat sera **non nul**, alors `ZF` deviendra 0.

Dans `RFLAGS` (**64 bits**) on à ces **flags** à ces **positions**. Si besoin on peut faire des recherches pour comprendre ce que chacun signifie.

![[Pasted image 20250501224330.png]]

---

### La pile
Evidemment, on a vu que c'est un **segment** d'un programme, mais à quoi elle sert ? Comment est-elle utilisée ? 

> Fidèle à son nom, elle utilise la structure de données qu'est **la pile**. Donc du **LIFO**(*Last In First Out*).

Ici, on utilisera l'exemple d'une **pile** pour du **x86**, donc **32 bits**. La seule différence avec du **64 bits**, c'est la **taille maximum** de chaque **élément** que la pile peut stocker.
On oublie pas que les **pointeurs** `esp` et `ebp` sont utilisés pour pointer vers le **haut** et le **bas** de la pile.

*On va suivre l'exemple donné par reverse.zip, auquel je vais ajouter mes notes complémentaires pour comprendre davantage comment la pile fonctionne..*

Donc on imagine une pile qui a un seul élément (*en gris*), les deux pointeurs pointent donc vers cet élément. 

La première chose qu'on doit remarquer, c'est les *adresses mémoire*, le **premier élément ajouté** (qui sera le **dernier** à être **dépilé**), il est à l'adresse **la plus haute** !  

![[Pasted image 20250504222013.png]]


Si on ajoute des éléments à la pile, forcément `esp` va changer de position, et pointer vers l'**adresse** **la plus** **basse** de la pile, qui contient le **dernier élément** ajouté:  

![[Pasted image 20250504223120.png]]

Et maintenant, si on veut récupérer `0xdeadbeef`, et le mettre dans un registre, comment on peut faire ? 

Puisqu'on connaît la structure de données, on sait qu'on peut soit **empiler** (ajouter un élément en haut de la pile, donc adresse + basse) ou **dépiler** (pop le dernier élément ajouté, et récupérer sa valeur).

Là, pour récupérer `0xdeadbeef`, on va **dépiler 3 fois** :  

```asm
pop edi ; edi = 0x41 (encodage ASCII de 'A')
pop edi ; edi = 0x00000000
pop edi ; edi = 0xdeadbeef
```

Mais alors comment ça marche réellement ? 

#### Empiler
On utilise 

```asm
push ELT
```

Avec `ELT` étant l'élément en tête de la **stack**. Par exemple, `ELT` peut être :  
- Une valeur concrète, comme `0xdeadbeef`.
- Un registre comme `rax`, et ce sera la valeur que le registre contient qui sera empilée.

#### Dépiler
On utilise

```asm
pop DEST 
```

Avec `DEST` étant la destination où la valeur qu'on pop sera stockée. C'est donc **toujours** un **registre** !  

### Stack Frame
Maintenant, comment fonctionne la pile pour tout ce qui est **appels de fonctions**, **variables locales** ou encore **arguments**.

> 	L'idée globale est que **chaque fonction** puisse **gérer de façon autonome** :  
> 	- Ses **variables locales**
> 	- L'accès aux **arguments**

Pour comprendre tout ça, on va utiliser un code qui comporte une fonction **`main`** qui fait appel à une fonction `discriminant`.  

```c
#include "stdio.h"

int discriminant(int a, int b, int c)
{
	int result = b*b - (4*a*c);
	return result;
}

int main()
{
	// Le polynome x² + 10x + 3
	int a = 1;
	int b = 10;
	int c = 3;

	int result = discriminant(a, b, c);

	printf("Le discriminant de mon polynôme est %d\n", result);
	
}
```

On peut voir qu'avant d'appeler `discrminant`, la fonction `main` initialise **trois variables**. Elle possède donc déjà une **stack frame**. 

Ca peut se représenter comme ça :  

![[Pasted image 20250504225431.png]]

> Maintenant, comment faire pour que `discriminant` ait sa propre **stack frame** sans empiéter sur celle de `main` ? 
> Et c'est justement là que sont impliqué **l'appel à la fonction** le **prologue** !
> 

#### L'appel de fonction
Comment est effectué l'appel à la fonction `discriminant`en **x86** ? 

##### Arguments
La fonction `discriminant` va devoir récupérer ses **3 arguments**. Donc ils sont ajoutés à la pile.

![[Pasted image 20250504231100.png]]

> Il faut bien noter que les arguments sont ajoutés dans le sens inverse de l'appel (**c**, **b**, puis **a**) puisque la **pile** est une **LIFO**, donc pour accéder à `a` en premier, il faut la placer en haut pour la dépiler en premier.


##### Adresse de retour
Dès que `discriminant` sera exécutée, il faut bien qu'elle puisse retourner à l'**instruction** qui suit son appel dans `main`.

Cette **instruction**, c'est l'affectation du résultat renvoyé par `discriminant` dans la variable `result` :  

```c
int result = discriminant(a, b, c);
```

Donc en assembleur, cette instruction elle ressemblera à un truc de ce style :  

```asm
mov result, eax
```

On a donc une **pile** qui ressemble à ça après l'ajout des **variables** ainsi que de l'**adresse de retour** :  

![[Pasted image 20250507115356.png]]

### Prologue
Maintenant qu'on a nos informations nécessaires dans la pile, à savoir les **variables** (leur valeur) que va utiliser `discriminant` ainsi que l'**adresse de retour**, on peut rentrer dans le code de cette dernière.

Pour rappel, on avait vu que le prologue c'était ces instructions :  

```asm
push ebp
mov ebp, esp
sub esp, 0x10
```

La première ligne, elle ajoute en haut de la pile `ebp`, donc l'**adresse** du **premier élément** ajouté à la **pile**.

Elle ressemble maintenant à ça :  

![[Pasted image 20250507120525.png]]

Et c'est là qu'on a la création de la nouvelle **stack frame**. La deuxième instruction :  

```asm
mov ebp, esp
```

Elle copie la valeur de `esp` (le sommet de la pile) dans `ebp`. Cela change donc la base.

> On pourra la retrouver quand on aura tout dépilé puisqu'on a la valeur de base de `ebp` qu'on a stocké à `0x700000F4`.

Maintenant, pour la nouvelle **stack frame**, on à `esp` et `ebp` qui pointent à la même **adresse**. C'est un peu comme si on avait une **deuxième pile** pour `discriminant` mais c'est tout l'intérêt des **stack frames**.

![[Pasted image 20250507121425.png]]

Maintenant, il faut pouvoir réserver un peu d'espace pour les **variables locales**, en l'occurrence `result` :  

```c
int discriminant(int a, int b, int c)
{
	int result = b*b - (4*a*c);
	return result;
}
```

Et c'est ce que fait la **dernière instruction** du **prologue** :  

```asm
sub esp, 0x10
```

En fait, cette instruction elle **soustrait** `esp` de l'**espace désiré**. Ce qui réserve **16 octets**, soit 4 cases de **4 octets** chacune.

> *On fait une **soustraction** pour réserver de la mémoire car on rappelle que les **adresses basses** sont en **haut de la pile**, donc plus on ajoute d'éléments à la pile, plus ça va bas dans la pile.*

Le nouvel état de la pile ressemble à ça :  

![[Pasted image 20250507122753.png]]

Maintenant, un truc un peu bizarre, c'est pourquoi avoir alloué **16 octets** pour `result` en sachant qu'elle n'a une taille que de **4 octets** puisque c'est un **int**.

> Le processeur aime bien que `esp` soit aligné sur **8 bits**, c’est-à-dire qu’il ait la forme suivante : `0xXXXXXXX0`. Autrement dit, que `esp` soit un multiple de 16. C’est pourquoi que l’on ait **4, 8 ou 12 octets** de variables locales, le processeur réservera **16 octets**.


### Fonction

Une fonction, comme on l'a vu, est constituée de 3 choses :  
1. Le [**prologue**](#prologue)
2. Faire des trucs (stockage, calculs..)
3. L'[**épilogue**](#epilogue)

Et en fait, on retrouve exactement cette structure sur IDA quand on regarde `main` :  

![[Pasted image 20250507124002.png]]

### Epilogue
On va donc s'intéresser à la troisième partie, l'**épilogue**.

On a ces deux instructions :  

```asm
leave
ret
```

#### leave

> Il faut savoir que `leave` n’est pas une **instruction atomique**. Cela signifie que lorsque le processeur exécute cette instruction, il exécutera en réalité plusieurs instructions.

En l'occurrence, en **32 bits**, ``leave`` est l'équivalent de :  

```asm
mov esp, ebp
pop ebp
```

C'est donc l'**inverse** du **prologue**. Car ça remet `esp` à la base de la **stack frame**, et ça libère les **variables locales**.

On a donc ça :  

![[Pasted image 20250507163930.png]]

#### ret
Puis cette deuxième instruction de l'**épilogue**, elle est équivalente à :  

```asm
pop eip
```

Ce qui veut dire qu'on met l'**adresse de retour** (là où `esp` et `ebp` pointent actuellement) dans `eip` qui est un registre spécialement conçu pour **pointer** vers l'**instruction courante exécutée**.

> Comme c’est la fonction `main` qui s’est chargée de mettre les arguments sur la pile, c’est à elle de s’en débarrasser 😏 !

Et on peut aussi noter une chose, puisqu'on a dit que c'est `main` qui s'occupe de dépiler les arguments auxquels devait accéder `discriminant`, ça voudrait dire que discriminant n'a pas pu accéder à ces arguments ? 

Et bien si, en accédant par exemple à `EBP - 4`, ce qui correspond à l'élément juste en dessous de l'élément de la base de la pile (de la stack frame précisément). Etc.


