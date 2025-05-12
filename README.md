# Reverse Engineering

Ici, seront toutes mes notes liées au reverse, notamment suite au cours disponible sur [reverse.zip](https://reverse.zip)

---
## Commandes utiles

Trouver l'**emplacement** d'un binaire (ou programme).
```sh
which /usr/bin/xclock
```

--- 

Voir l'**entête** d'un exécutable :  
```bash
xxd /usr/bin/xclock | head
```
Une chose qu'on peut remarquer avec un **xxd**, c'est qu'on trouve parfois des lettres, parfois des `.` :  
```bash
┌──(naxyl㉿kali)-[~]
└─$ xxd /usr/bin/xclock | head
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 6049 0000 0000 0000  ..>.....`I......
00000020: 4000 0000 0000 0000 d0cf 0000 0000 0000  @...............
```
> Tout ce qui est en-dessous de `0x20` (espace) ou au-dessus de `0x7E` (tilde `~`) est remplacé par `.` car non représentable dans l'ASCII simple.

---

Afficher les **segments** d'un **ELF** :  
```sh
readelf -l /usr/bin/xclock
```

---

Afficher les **sections** d'un **PE** :  
```sh
readpe hello.exe
```

A noter qu'il faut préinstaller :  
```sh
sudo apt install mingw-w64
sudo apt install readpe
```

---

Voir l'**architecture** de son PC :  

Linux.
```sh
lscpu | grep Arch
```

Windows.
```sh
$env:PROCESSOR_ARCHITECTURE
```

---


## Installer IDA
On télécharge le `.run` depuis [ici](https://my.hex-rays.com/dashboard/download-center/9.1/ida-free).  

Puis, on lui ajoute les permissions d'être exécuté :  

```sh
┌──(naxyl㉿kali)-[~/Downloads]
└─$ chmod +x ida-free-pc_91_x64linux.run
```

On le run :

```sh
┌──(naxyl㉿kali)-[~/Downloads]
└─$ ./ida-free-pc_91_x64linux.run 
```

On finit l'installation, et ça nous dit qu'on a besoin d'une **license**. On la récupère [ici](https://my.hex-rays.com/dashboard/licenses).

Une fois qu'on a le fichier ``.hexclic``, on le met dans le dossier de l'installation :  

```sh
┌──(naxyl㉿kali)-[~/Downloads]
└─$ mv idafree_96-51DE-DD49-FF.hexlic ~/ida-free-pc-9.1
```

Et c'est installé, on peut le lancer via le menu démarrer ! 

## Assembleur

### Tableau comparatif
Ici, un tableau qui compare les différentes **architectures** **matérielles** (ISA).

| Architecture       | Registres connus   | Exemples d'utilisation           |
| ------------------ | ------------------ | -------------------------------- |
| **x86**            | eax, ebx, ecx, edx | *PC, serveurs, anciens systèmes* |
| **x86-64 / AMD64** | rax, rbx, rcx, rdx | *Tous les PC modernes*           |
| **ARM**            | r0, r1, r2, …      | *Smartphones, Raspberry Pi*      |
| **MIPS**           | $t0, $s0, …        | *Routeurs, systèmes embarqués*   |
| **RISC-V**         | x0, x1, x2, …      | *Open Source, recherche*         |

### Syntaxe d'écriture assembleur
On retrouve deux façon d'écrire une instruction en assembleur.

> A noter qu'en **AT&T**, les registres sont préfixés par `%` et les constantes par `$`

| Syntaxe | Ordre des opérandes   | Exemple         | Format des pointeurs                                                                 | Exemple   |
| ------- | --------------------- | --------------- | ------------------------------------------------------------------------------------ | --------- |
| Intel   | `destination, source` | `mov eax, 1`    | placés entre **crochets**                                                            | `[ebp+8]` |
| AT&T    | `source, destination` | `movl $1, %eax` | placés entre **parenthèses** et les offsets sont placés avant la première parenthèse | `8(%ebp)` |

```ad-note
title: Définition : Mnémonique
C’est en quelque sorte le nom de l’instruction exécutée. (`mov`, `push`, `pop`...)
```

```ad-note
title: Définition : Opérandes
Ce sont les arguments qu'on donne aux **mnémoniques**. Cela peut être des registres, pointeurs ou valeurs concrètes utilisées par l’instruction.
```

### Registres
Pour du **x86**, les principaux **registres** :  

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

### Instructions
