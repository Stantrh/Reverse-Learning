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

