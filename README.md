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
