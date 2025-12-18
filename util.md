# Documentation Détaillée - Module utils.asm

**Projet :** Simulateur de Gestion de Mémoire Virtuelle  
**Fichier :** `utils.asm`  
**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025

---

## Table des Matières

1. [Vue d'ensemble du module](#vue-ensemble)
2. [Structure et conventions](#structure-conventions)
3. [Fonction PrintString](#printstring)
4. [Fonction PrintHexQWord](#printhexqword)
5. [Fonction PrintDecimal](#printdecimal)
6. [Fonction GetUserInput](#getuserinput)
7. [Fonction GetUserChoice](#getuserchoice)
8. [Fonction ParseHex](#parsehex)
9. [Dépendances et intégration](#dependances)

---

## Vue d'ensemble du module {#vue-ensemble}

### Rôle du module

Le module `utils.asm` est une **bibliothèque de fonctions utilitaires** qui fournit toutes les opérations de base nécessaires pour l'affichage, la lecture d'entrées utilisateur et les conversions numériques. C'est la **couche d'abstraction** entre les autres modules du projet et les API Windows.

### Pourquoi ce module existe-t-il ?

**Problème à résoudre :** En assembleur pur, nous n'avons **pas accès à la bibliothèque C standard** (`printf`, `scanf`, `atoi`, etc.). Chaque opération d'affichage, de lecture ou de conversion doit être implémentée manuellement en utilisant directement les API Windows.

**Solution :** Centraliser toutes ces fonctions dans un module dédié pour :
- **Éviter la duplication** : Une seule implémentation utilisée partout
- **Cohérence** : Tous les affichages utilisent le même mécanisme
- **Maintenabilité** : Modifier une seule fonction pour changer le comportement
- **Réutilisabilité** : Les fonctions sont utilisables par tous les autres modules

### Statistiques d'utilisation

- **PrintString** : Appelée **87 fois** dans tout le projet
- **PrintHexQWord** : Appelée **6 fois** (adresses mémoire critiques)
- **PrintDecimal** : Appelée **6 fois** (tailles et statistiques)
- **GetUserChoice** : Appelée à **chaque itération** du menu principal
- **ParseHex** : Utilisée pour la **protection d'accès mémoire**

### Fonctions exportées

Le module exporte 6 fonctions publiques :
1. `PrintString` : Affichage de chaînes de caractères
2. `PrintHexQWord` : Affichage de nombres 64 bits en hexadécimal
3. `PrintDecimal` : Affichage de nombres en décimal
4. `GetUserInput` : Lecture générique depuis la console
5. `GetUserChoice` : Lecture spécialisée pour les choix du menu
6. `ParseHex` : Conversion chaîne hexadécimale → nombre

---

## Structure et conventions {#structure-conventions}

### Déclarations externes

Le module dépend de variables définies dans `data.asm` :

```9:12:utils.asm
EXTERN hStdIn:QWORD, hStdOut:QWORD
EXTERN bytesRead:DWORD, bytesWritten:DWORD
EXTERN inputBuffer:BYTE, outputBuffer:BYTE, hexBuffer:BYTE
EXTERN msgNewLine:BYTE
```

**Explication :**
- `hStdIn` / `hStdOut` : Handles de la console (initialisés dans `main.asm`)
- `bytesRead` / `bytesWritten` : Compteurs pour les opérations d'E/S
- Buffers : Espaces de stockage temporaires
- `msgNewLine` : Chaîne contenant le caractère de retour à la ligne

### API Windows utilisées

```16:19:utils.asm
EXTERN GetStdHandle:PROC
EXTERN WriteConsoleA:PROC
EXTERN ReadConsoleA:PROC
EXTERN lstrlenA:PROC
```

**Explication :**
- `WriteConsoleA` : Écrit sur la console de sortie
- `ReadConsoleA` : Lit depuis la console d'entrée
- `lstrlenA` : Calcule la longueur d'une chaîne
- `GetStdHandle` : Déclaré mais non utilisé (initialisé dans `main.asm`)

### Convention d'appel Windows x64

Toutes les fonctions suivent la convention Windows x64 :

**Paramètres :**
- 1er paramètre : `RCX`
- 2ème paramètre : `RDX`
- 3ème paramètre : `R8`
- 4ème paramètre : `R9`
- Paramètres supplémentaires : Sur la pile `[RSP+20h]`, `[RSP+28h]`, etc.

**Valeurs de retour :**
- Entiers : `RAX`
- Pointeurs : `RAX`

**Shadow space :**
- 32 bytes (20h) doivent être réservés sur la pile même si non utilisés
- Alignement : La pile doit être alignée sur 16 bytes

**Registres sauvegardés :**
- RBX, RBP, RSI, RDI, R12-R15 doivent être sauvegardés si utilisés

---

## Fonction PrintString {#printstring}

### Spécification

```29:56:utils.asm
PrintString PROC
    push rbp
    mov rbp, rsp
    sub rsp, 30h
    push rbx
    
    mov rbx, rcx                    ; Sauvegarder le pointeur
    
    ; Calculer la longueur
    call lstrlenA
    test rax, rax
    jz print_done
    
    ; Écrire sur la console
    mov rcx, hStdOut
    mov rdx, rbx
    mov r8d, eax
    lea r9, bytesWritten
    xor rax, rax
    mov [rsp+20h], rax
    call WriteConsoleA
    
print_done:
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
PrintString ENDP
```

### Paramètres d'entrée

- **RCX** : Pointeur vers la chaîne de caractères à afficher (terminée par NULL)
  - Type : `const char*`
  - Exemple : `lea rcx, msgTitle` où `msgTitle` est une chaîne définie dans `data.asm`

### Valeur de retour

- **Aucune** (procédure void)
- La fonction affiche la chaîne sur la console standard

### Comment ça marche ?

**Étape 1 : Préparation de la pile**
```assembly
push rbp
mov rbp, rsp
sub rsp, 30h      ; Shadow space (20h) + alignement
push rbx          ; Sauvegarder RBX (callee-saved)
```

**Pourquoi :**
- Conforme à la convention Windows x64
- RBX sera utilisé pour sauvegarder le pointeur

**Étape 2 : Sauvegarde du pointeur**
```assembly
mov rbx, rcx      ; Sauvegarder RCX dans RBX
```

**Pourquoi :**
- `lstrlenA` utilise RCX comme paramètre (la chaîne)
- On doit sauvegarder le pointeur car RCX sera modifié

**Étape 3 : Calcul de la longueur**
```assembly
call lstrlenA     ; RAX = longueur de la chaîne
test rax, rax     ; Vérifier si longueur = 0
jz print_done     ; Si oui, sortir immédiatement
```

**Pourquoi :**
- `WriteConsoleA` nécessite la longueur exacte de la chaîne
- Si la chaîne est vide, pas besoin d'appeler `WriteConsoleA`
- Optimisation : Évite un appel API inutile

**Étape 4 : Appel à WriteConsoleA**
```assembly
mov rcx, hStdOut              ; 1er param : Handle console sortie
mov rdx, rbx                  ; 2ème param : Pointeur vers chaîne
mov r8d, eax                  ; 3ème param : Nombre de caractères (longueur)
lea r9, bytesWritten          ; 4ème param : Pointeur vers variable de retour
xor rax, rax                  ; Zéro RAX
mov [rsp+20h], rax            ; 5ème param : lpReserved = NULL
call WriteConsoleA
```

**Paramètres de WriteConsoleA :**
- `hStdOut` : Handle obtenu via `GetStdHandle(STD_OUTPUT_HANDLE)` dans `main.asm`
- `rbx` : Pointeur vers la chaîne à afficher
- `eax` : Nombre de caractères à écrire (longueur calculée)
- `bytesWritten` : Variable qui recevra le nombre de bytes réellement écrits
- `[RSP+20h]` : 5ème paramètre réservé, mis à NULL

**Étape 5 : Restauration et retour**
```assembly
pop rbx          ; Restaurer RBX
mov rsp, rbp     ; Restaurer la pile
pop rbp          ; Restaurer RBP
ret              ; Retourner à l'appelant
```

### Utilisation dans le projet

**Exemple 1 : Affichage du menu**
```115:119:main.asm
lea rcx, menuTitle
call PrintString

lea rcx, menuOptions
call PrintString
```

**Résultat :** L'utilisateur voit le menu principal s'afficher.

**Exemple 2 : Message de chargement**
```64:65:program.asm
lea rcx, msgEnterFile
call PrintString
```

**Résultat :** L'utilisateur voit "Entrez le nom du fichier programme: ".

**Exemple 3 : Messages d'état**
```280:282:memory.asm
lea rcx, msgNewLine
call PrintString
lea rcx, msgRAMState
call PrintString
```

**Résultat :** Affichage de l'état de la mémoire avec sauts de ligne.

### Pourquoi cette fonction est importante

1. **Fondation de l'interface utilisateur** : Sans elle, aucun message ne serait visible
2. **Utilisée 87 fois** : Fonction la plus utilisée du projet
3. **Abstraction** : Cache la complexité de `WriteConsoleA` aux autres modules
4. **Cohérence** : Tous les affichages utilisent le même mécanisme

### Erreurs possibles

- **Chaîne non terminée par NULL** : `lstrlenA` pourrait parcourir trop loin
- **RCX = NULL** : Crash si le pointeur est invalide
- **hStdOut non initialisé** : L'affichage échouerait (initialisé dans `main.asm`)

---

## Fonction PrintHexQWord {#printhexqword}

### Spécification

```62:103:utils.asm
PrintHexQWord PROC
    push rbp
    mov rbp, rsp
    sub rsp, 40h
    push rbx
    push rsi
    push rdi
    
    mov rax, rcx
    lea rdi, hexBuffer
    add rdi, 15                     ; Pointer vers la fin (16 caractères hex)
    mov BYTE PTR [rdi+1], 0         ; Null terminator
    
    mov rcx, 16                     ; 16 chiffres hexadécimaux
    
convert_hex_loop:
    mov rbx, rax
    and rbx, 0Fh                    ; Obtenir les 4 bits de poids faible
    cmp rbx, 10
    jl is_digit
    add rbx, 'A' - 10
    jmp store_char
is_digit:
    add rbx, '0'
store_char:
    mov [rdi], bl
    dec rdi
    shr rax, 4                      ; Décaler de 4 bits
    dec rcx
    jnz convert_hex_loop
    
    inc rdi
    mov rcx, rdi
    call PrintString
    
    pop rdi
    pop rsi
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
PrintHexQWord ENDP
```

### Paramètres d'entrée

- **RCX** : Nombre 64 bits (QWORD) à afficher en hexadécimal
  - Type : `unsigned long long` (64 bits)
  - Exemple : `mov rcx, ramBase` où `ramBase` est une adresse mémoire

### Valeur de retour

- **Aucune** (procédure void)
- La fonction affiche le nombre en hexadécimal sur la console

### Comment ça marche ?

**Étape 1 : Initialisation**
```assembly
mov rax, rcx                ; RAX = nombre à convertir
lea rdi, hexBuffer          ; RDI = pointeur vers le buffer
add rdi, 15                 ; Pointer vers la position 15 (16ème caractère)
mov BYTE PTR [rdi+1], 0     ; Mettre NULL terminator à la position 16
```

**Pourquoi pointer vers la fin ?**
- On construit la chaîne **de droite à gauche** (du LSB au MSB)
- Chaque chiffre hexadécimal représente 4 bits
- Un QWORD (64 bits) = 16 chiffres hexadécimaux
- On part de la fin et on remplit vers le début

**Étape 2 : Boucle de conversion (16 itérations)**

```assembly
convert_hex_loop:
    mov rbx, rax
    and rbx, 0Fh            ; Extraire les 4 bits de poids faible (nibble)
```

**Extraction du nibble :**
- `and rbx, 0Fh` isole les 4 bits de poids faible
- Exemple : `0x1234ABCD & 0xF = 0xD`

```assembly
    cmp rbx, 10
    jl is_digit             ; Si < 10, c'est un chiffre (0-9)
    add rbx, 'A' - 10       ; Sinon, c'est une lettre (A-F)
    jmp store_char
is_digit:
    add rbx, '0'            ; Convertir 0-9 en '0'-'9'
```

**Conversion en ASCII :**
- Valeurs 0-9 → Ajouter '0' (48) → '0' à '9'
- Valeurs 10-15 → Ajouter 'A' - 10 (55) → 'A' à 'F'

```assembly
store_char:
    mov [rdi], bl           ; Stocker le caractère dans le buffer
    dec rdi                 ; Déplacer le pointeur vers la gauche
    shr rax, 4              ; Décaler RAX de 4 bits vers la droite
    dec rcx                 ; Décrémenter le compteur
    jnz convert_hex_loop    ; Continuer si pas terminé
```

**Décalage :**
- `shr rax, 4` décale de 4 bits vers la droite
- Permet de traiter le nibble suivant à la prochaine itération
- Exemple : `0x1234ABCD >> 4 = 0x01234ABC`

**Étape 3 : Ajustement et affichage**

```assembly
inc rdi                     ; RDI pointe maintenant un caractère avant le début
mov rcx, rdi                ; Passer le pointeur à PrintString
call PrintString
```

**Pourquoi incrémenter ?**
- Après la boucle, RDI pointe un caractère avant le début réel
- On incrémente pour pointer au début de la chaîne hexadécimale

### Exemple de conversion

**Nombre :** `0x0000020E61770900` (adresse RAM typique)

**Itérations :**
1. `0x20E61770900 & 0xF = 0x0` → '0'
2. `0x20E6177090 & 0xF = 0x0` → '0'
3. `0x20E617709 & 0xF = 0x9` → '9'
4. `0x20E61770 & 0xF = 0x0` → '0'
5. ... (continuer pour tous les 16 chiffres)

**Résultat affiché :** `0000020E61770900`

### Utilisation dans le projet

**Exemple 1 : Affichage de l'adresse de début de la RAM**
```284:285:memory.asm
mov rcx, ramBase
call PrintHexQWord
```

**Résultat :** L'utilisateur voit `Adresse debut: 0x0000020E61770900`

**Exemple 2 : Affichage de l'adresse d'un programme**
```807:808:program.asm
mov rcx, [rsi].ProgramInfo.startAddr
call PrintHexQWord
```

**Résultat :** L'utilisateur voit où chaque programme est chargé en mémoire.

**Exemple 3 : Affichage d'une adresse lors de la vérification d'accès**
```720:721:program.asm
mov rcx, r12
call PrintHexQWord
```

**Résultat :** Confirmation de l'adresse à laquelle l'accès a été autorisé.

### Pourquoi cette fonction est importante

1. **Visualisation des adresses** : Essentielle pour comprendre où se trouvent les programmes
2. **Format standard** : Les adresses sont traditionnellement en hexadécimal
3. **Débogage** : Permet de tracer les problèmes d'allocation
4. **Conformité** : Suit les conventions d'affichage des outils système

### Caractéristiques techniques

- **Précision** : 16 chiffres hexadécimaux (64 bits complets)
- **Performance** : Conversion manuelle optimisée, pas d'appel de fonction complexe
- **Pas de zéros de tête supprimés** : Affiche toujours 16 chiffres (peut être modifié)

---

## Fonction PrintDecimal {#printdecimal}

### Spécification

```238:271:utils.asm
PrintDecimal PROC
    push rbp
    mov rbp, rsp
    sub rsp, 40h
    push rbx
    push rsi
    push rdi
    
    mov rax, rcx
    lea rdi, hexBuffer
    add rdi, 30                     ; Pointer vers la fin du buffer
    mov BYTE PTR [rdi], 0           ; Null terminator
    
    mov rbx, 10
    
convert_dec_loop:
    xor rdx, rdx
    div rbx                         ; RAX / 10, reste dans RDX
    add dl, '0'                     ; Convertir en ASCII
    dec rdi
    mov [rdi], dl
    test rax, rax
    jnz convert_dec_loop
    
    mov rcx, rdi
    call PrintString
    
    pop rdi
    pop rsi
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
PrintDecimal ENDP
```

### Paramètres d'entrée

- **RCX** : Nombre 64 bits (QWORD) à afficher en décimal
  - Type : `unsigned long long` (64 bits)
  - Exemple : `mov rcx, RAM_SIZE` pour afficher "4096"

### Valeur de retour

- **Aucune** (procédure void)
- La fonction affiche le nombre en décimal sur la console

### Comment ça marche ?

**Étape 1 : Initialisation**
```assembly
mov rax, rcx                ; RAX = nombre à convertir
lea rdi, hexBuffer          ; Utilise hexBuffer comme buffer de travail
add rdi, 30                 ; Pointer vers la fin (30 caractères suffisent pour un QWORD)
mov BYTE PTR [rdi], 0       ; Null terminator
mov rbx, 10                 ; Diviseur = 10
```

**Pourquoi pointer vers la fin ?**
- On construit la chaîne **de droite à gauche** (des unités vers les dizaines)
- On part de la fin et on remplit vers le début

**Étape 2 : Boucle de conversion**

```assembly
convert_dec_loop:
    xor rdx, rdx            ; Mettre RDX à zéro (NÉCESSAIRE pour DIV)
    div rbx                 ; RAX = RAX / 10, RDX = RAX % 10
```

**Division :**
- `div rbx` divise **RDX:RAX** (128 bits) par RBX (10)
- Quotient → RAX
- Reste → RDX (chiffre de droite)
- **Important :** RDX doit être à zéro avant DIV pour un QWORD

**Exemple :**
- `12345 / 10 = 1234`, reste `5`
- `1234 / 10 = 123`, reste `4`
- etc.

```assembly
    add dl, '0'             ; Convertir le reste (0-9) en ASCII ('0'-'9')
    dec rdi                 ; Déplacer vers la gauche
    mov [rdi], dl           ; Stocker le chiffre
    test rax, rax           ; Vérifier si RAX = 0
    jnz convert_dec_loop    ; Continuer si non nul
```

**Condition d'arrêt :**
- Quand RAX = 0, tous les chiffres ont été extraits
- La boucle s'arrête

**Étape 3 : Affichage**
```assembly
mov rcx, rdi                ; Passer le pointeur au début de la chaîne
call PrintString
```

### Exemple de conversion

**Nombre :** `12345`

**Itérations :**
1. `12345 / 10 = 1234`, reste `5` → Stocker '5'
2. `1234 / 10 = 123`, reste `4` → Stocker '4'
3. `123 / 10 = 12`, reste `3` → Stocker '3'
4. `12 / 10 = 1`, reste `2` → Stocker '2'
5. `1 / 10 = 0`, reste `1` → Stocker '1'
6. RAX = 0 → Sortir de la boucle

**Résultat :** Chaîne construite de droite à gauche : "12345"

### Utilisation dans le projet

**Exemple 1 : Affichage de l'utilisation de la RAM**
```328:329:memory.asm
mov rcx, r12                ; r12 = octets utilisés
call PrintDecimal
```

**Résultat :** L'utilisateur voit "Utilise: 1536 octets"

**Exemple 2 : Affichage de la taille d'un programme**
```799:800:program.asm
mov rcx, [rsi].ProgramInfo.progSize
call PrintDecimal
```

**Résultat :** L'utilisateur voit "Taille: 512 octets"

**Exemple 3 : Affichage du pourcentage d'utilisation**
```349:350:memory.asm
mov rcx, rax                ; rax = pourcentage calculé
call PrintDecimal
```

**Résultat :** L'utilisateur voit "37%"

### Pourquoi cette fonction est importante

1. **Lisibilité** : Les nombres décimaux sont plus naturels que l'hexadécimal pour les tailles
2. **Interface utilisateur** : Améliore l'expérience utilisateur
3. **Statistiques** : Permet d'afficher les pourcentages et tailles de manière compréhensible

### Caractéristiques techniques

- **Précision** : Jusqu'à 19 chiffres décimaux (maximum pour un QWORD non signé)
- **Pas de formatage** : Affiche le nombre tel quel, sans séparateurs de milliers
- **Zéros de tête supprimés** : Affiche seulement les chiffres significatifs

---

## Fonction GetUserInput {#getuserinput}

### Spécification

```110:136:utils.asm
GetUserInput PROC
    push rbp
    mov rbp, rsp
    sub rsp, 40h
    push rbx
    push rsi
    
    mov rbx, rcx                    ; Buffer
    mov rsi, rdx                    ; Taille
    
    mov rcx, hStdIn
    mov rdx, rbx
    mov r8, rsi
    lea r9, bytesRead
    xor rax, rax
    mov [rsp+20h], rax
    call ReadConsoleA
    
    ; Retourner le nombre de bytes lus
    mov eax, bytesRead
    
    pop rsi
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
GetUserInput ENDP
```

### Paramètres d'entrée

- **RCX** : Pointeur vers le buffer de réception
  - Type : `char*`
  - Le buffer doit être pré-alloué (défini dans `data.asm`)
  
- **RDX** : Taille maximale du buffer (en octets)
  - Type : `size_t`
  - Exemple : `mov rdx, 256` pour un buffer de 256 octets

### Valeur de retour

- **RAX** : Nombre de bytes réellement lus
  - Type : `DWORD` (32 bits)
  - N'inclut pas le caractère de retour à la ligne (CR/LF)
  - Exemple : Si l'utilisateur tape "hello" + Enter, retourne 5

### Comment ça marche ?

**Étape 1 : Sauvegarde des paramètres**
```assembly
mov rbx, rcx                ; Sauvegarder le pointeur buffer
mov rsi, rdx                ; Sauvegarder la taille
```

**Pourquoi :** RCX et RDX seront utilisés pour les paramètres de `ReadConsoleA`

**Étape 2 : Appel à ReadConsoleA**
```assembly
mov rcx, hStdIn             ; 1er param : Handle console entrée
mov rdx, rbx                ; 2ème param : Buffer de destination
mov r8, rsi                 ; 3ème param : Taille maximale
lea r9, bytesRead           ; 4ème param : Pointeur vers variable de retour
xor rax, rax
mov [rsp+20h], rax          ; 5ème param : lpReserved = NULL
call ReadConsoleA
```

**Paramètres de ReadConsoleA :**
- `hStdIn` : Handle obtenu via `GetStdHandle(STD_INPUT_HANDLE)` dans `main.asm`
- `rbx` : Buffer où stocker les caractères lus
- `rsi` : Taille maximale (limite pour éviter le débordement)
- `bytesRead` : Variable qui recevra le nombre de bytes réellement lus
- `[RSP+20h]` : Paramètre réservé

**Comportement de ReadConsoleA :**
- Lit depuis la console jusqu'à :
  - La taille maximale atteinte, OU
  - Un retour à la ligne (utilisateur appuie sur Enter)
- Stocke les caractères dans le buffer
- Inclut CR (13) et LF (10) dans la lecture

**Étape 3 : Retour de la valeur**
```assembly
mov eax, bytesRead          ; Retourner le nombre de bytes lus
```

### Utilisation dans le projet

**Utilisation actuelle :** Cette fonction est définie mais **peu utilisée directement**. La plupart des lectures utilisent `ReadConsoleA` directement ou `GetUserChoice` pour le menu.

**Usage typique :**
```assembly
lea rcx, inputBuffer        ; Buffer défini dans data.asm
mov rdx, 256                ; Taille maximale
call GetUserInput           ; RAX = nombre de caractères lus
```

### Pourquoi cette fonction existe-t-elle ?

1. **Fonction générique** : Peut être utilisée pour lire n'importe quelle entrée
2. **Abstraction** : Cache la complexité de `ReadConsoleA`
3. **Extensibilité** : Permet d'ajouter facilement de nouvelles fonctionnalités de lecture

### Note importante

Cette fonction retourne le nombre de bytes **avec** le CR/LF. Si vous voulez seulement le nombre de caractères utiles, il faut soustraire 2 (ou 1 selon le système).

---

## Fonction GetUserChoice {#getuserchoice}

### Spécification

```142:163:utils.asm
GetUserChoice PROC
    push rbp
    mov rbp, rsp
    sub rsp, 40h
    
    ; Lire l'entrée utilisateur
    mov rcx, hStdIn
    lea rdx, inputBuffer
    mov r8d, 256
    lea r9, bytesRead
    xor rax, rax
    mov [rsp+20h], rax
    call ReadConsoleA
    
    ; Convertir le premier caractère en nombre
    movzx rax, BYTE PTR inputBuffer
    sub rax, '0'
    
    mov rsp, rbp
    pop rbp
    ret
GetUserChoice ENDP
```

### Paramètres d'entrée

- **Aucun** (fonction sans paramètres)

### Valeur de retour

- **RAX** : Choix utilisateur sous forme de nombre (1-7 normalement)
  - Type : `DWORD` (32 bits, mais retourné dans RAX)
  - Conversion ASCII → nombre : '1' → 1, '2' → 2, etc.
  - **Attention :** Si l'utilisateur tape autre chose, la conversion peut donner un résultat inattendu

### Comment ça marche ?

**Étape 1 : Lecture de l'entrée**
```assembly
mov rcx, hStdIn             ; Handle console entrée
lea rdx, inputBuffer        ; Buffer défini dans data.asm
mov r8d, 256                ; Taille maximale
lea r9, bytesRead           ; Variable de retour
call ReadConsoleA
```

**Comportement :**
- Lit jusqu'à 256 caractères OU jusqu'au retour à la ligne
- Stocke dans `inputBuffer` (variable globale dans `data.asm`)
- `bytesRead` contient le nombre de caractères lus (incluant CR/LF)

**Étape 2 : Conversion du premier caractère**
```assembly
movzx rax, BYTE PTR inputBuffer  ; Charger le premier caractère (zero-extend)
sub rax, '0'                     ; Convertir ASCII → nombre
```

**Conversion :**
- `movzx` : Charge un byte et remplit les bits supérieurs avec des zéros
- `sub rax, '0'` : Conversion ASCII
  - '0' (48) → 0
  - '1' (49) → 1
  - '2' (50) → 2
  - ...
  - '7' (55) → 7

**Exemple :**
- Utilisateur tape "3" + Enter
- `inputBuffer[0]` = '3' (51 en ASCII)
- `51 - 48 = 3` → Retourne 3

### Utilisation dans le projet

**Dans main.asm, boucle principale :**
```52:54:main.asm
menu_loop:
    call DisplayMenu
    call GetUserChoice
```

**Puis routage selon le choix :**
```57:71:main.asm
cmp rax, 1
je option_load_program
cmp rax, 2
je option_display_memory
...
cmp rax, 7
je exit_program
jmp menu_loop
```

**Résultat :** Cette fonction est appelée **à chaque itération** de la boucle menu pour permettre la navigation.

### Pourquoi cette fonction est importante

1. **Cœur de l'interface utilisateur** : Sans elle, pas de navigation possible
2. **Simplicité** : Retourne directement un nombre, pas besoin de parsing complexe
3. **Performance** : Lit seulement le premier caractère, très rapide
4. **Spécialisée** : Optimisée pour le menu (7 options)

### Limitations

**Validation limitée :**
- Ne vérifie pas que l'entrée est bien un chiffre valide
- Si l'utilisateur tape "A", retourne 17 ('A' - '0')
- Si l'utilisateur tape "10", retourne seulement 1 (premier caractère)

**Gestion :**
- La validation est faite dans `main.asm` via les comparaisons
- Si le choix n'est pas 1-7, la boucle continue (pas d'action)

### Alternative possible

On pourrait améliorer avec validation :
```assembly
cmp rax, 1
jl invalid_choice
cmp rax, 7
jg invalid_choice
; Choix valide
```

Mais pour ce projet, la validation implicite dans `main.asm` est suffisante.

---

## Fonction ParseHex {#parsehex}

### Spécification

```170:232:utils.asm
ParseHex PROC
    push rbp
    mov rbp, rsp
    push rbx
    
    xor rax, rax
    xor rbx, rbx
    
parse_hex_loop:
    movzx rcx, BYTE PTR [rsi]
    
    ; Vérifier fin de chaîne
    cmp cl, 0
    je parse_hex_done
    cmp cl, 13
    je parse_hex_done
    cmp cl, 10
    je parse_hex_done
    cmp cl, ' '
    je parse_hex_done
    
    ; Convertir caractère
    cmp cl, '0'
    jl parse_hex_done
    cmp cl, '9'
    jle is_digit_hex
    
    ; Vérifier A-F
    cmp cl, 'A'
    jl check_lowercase
    cmp cl, 'F'
    jle is_upper_hex
    jmp check_lowercase
    
is_digit_hex:
    sub cl, '0'
    jmp add_hex_digit
    
is_upper_hex:
    sub cl, 'A'
    add cl, 10
    jmp add_hex_digit
    
check_lowercase:
    cmp cl, 'a'
    jl parse_hex_done
    cmp cl, 'f'
    jg parse_hex_done
    sub cl, 'a'
    add cl, 10
    
add_hex_digit:
    shl rax, 4
    movzx rbx, cl
    add rax, rbx
    inc rsi
    jmp parse_hex_loop
    
parse_hex_done:
    pop rbx
    pop rbp
    ret
ParseHex ENDP
```

### Paramètres d'entrée

- **RSI** : Pointeur vers la chaîne hexadécimale à convertir
  - Type : `const char*`
  - La chaîne peut contenir des chiffres (0-9) et lettres (A-F ou a-f)
  - Exemple : `lea rsi, inputBuffer` où inputBuffer contient "1A2B"

### Valeur de retour

- **RAX** : Nombre 64 bits converti
  - Type : `unsigned long long` (64 bits)
  - Exemple : "1A2B" → 6699 (0x1A2B en décimal)

### Comment ça marche ?

**Étape 1 : Initialisation**
```assembly
xor rax, rax                ; RAX = résultat (initialisé à 0)
xor rbx, rbx                ; RBX = registre temporaire
```

**Étape 2 : Boucle de parsing**

**a) Lecture du caractère**
```assembly
movzx rcx, BYTE PTR [rsi]   ; Charger le caractère actuel
```

**b) Détection de fin**
```assembly
cmp cl, 0                   ; Fin de chaîne (NULL)
je parse_hex_done
cmp cl, 13                  ; Retour chariot (CR)
je parse_hex_done
cmp cl, 10                  ; Retour à la ligne (LF)
je parse_hex_done
cmp cl, ' '                 ; Espace
je parse_hex_done
```

**Pourquoi ces conditions :**
- Arrêt automatique sur caractères de fin
- Gère les espaces et retours à la ligne automatiquement
- Robustesse : Ne parse pas au-delà de la chaîne utile

**c) Validation et conversion du caractère**

**Chiffres 0-9 :**
```assembly
cmp cl, '0'
jl parse_hex_done           ; < '0' → invalide
cmp cl, '9'
jle is_digit_hex            ; '0'-'9' → chiffre valide

is_digit_hex:
    sub cl, '0'             ; '0' → 0, '1' → 1, ..., '9' → 9
```

**Lettres majuscules A-F :**
```assembly
cmp cl, 'A'
jl check_lowercase          ; < 'A' → vérifier minuscules
cmp cl, 'F'
jle is_upper_hex            ; 'A'-'F' → lettre valide

is_upper_hex:
    sub cl, 'A'             ; 'A' → 0, 'B' → 1, ...
    add cl, 10              ; 'A' → 10, 'B' → 11, ..., 'F' → 15
```

**Lettres minuscules a-f :**
```assembly
check_lowercase:
    cmp cl, 'a'
    jl parse_hex_done       ; < 'a' → invalide
    cmp cl, 'f'
    jg parse_hex_done       ; > 'f' → invalide
    sub cl, 'a'             ; 'a' → 0, 'b' → 1, ...
    add cl, 10              ; 'a' → 10, 'b' → 11, ..., 'f' → 15
```

**d) Construction du nombre**
```assembly
add_hex_digit:
    shl rax, 4              ; Décaler RAX de 4 bits vers la gauche (×16)
    movzx rbx, cl           ; Charger la valeur du chiffre hex (0-15)
    add rax, rbx            ; Ajouter au résultat
    inc rsi                 ; Passer au caractère suivant
    jmp parse_hex_loop      ; Continuer
```

**Algorithme :**
- Chaque chiffre hexadécimal représente 4 bits
- On décale le résultat de 4 bits (multiplie par 16)
- On ajoute le nouveau chiffre
- Construction de gauche à droite

### Exemple de conversion

**Chaîne :** "1A2B"

**Itérations :**
1. '1' → 1, RAX = 0×16 + 1 = 1
2. 'A' → 10, RAX = 1×16 + 10 = 26 (0x1A)
3. '2' → 2, RAX = 26×16 + 2 = 418 (0x1A2)
4. 'B' → 11, RAX = 418×16 + 11 = 6699 (0x1A2B)

**Résultat :** RAX = 6699 (0x1A2B)

### Utilisation dans le projet

**Dans program.asm, RequestMemoryAccess :**
```703:706:program.asm
; Convertir hex en nombre
lea rsi, inputBuffer
call ParseHex
mov r12, rax                ; r12 = adresse demandée
```

**Contexte :**
1. L'utilisateur entre une adresse en hexadécimal (ex: "20E61770900")
2. `ReadConsoleA` lit la chaîne dans `inputBuffer`
3. `ParseHex` convertit la chaîne en nombre 64 bits
4. Le nombre est utilisé pour vérifier les limites d'accès

**Résultat :** Permet à l'utilisateur d'entrer des adresses mémoire en hexadécimal pour tester la protection d'accès.

### Pourquoi cette fonction est importante

1. **Protection d'accès** : Essentielle pour la fonctionnalité de vérification d'accès mémoire
2. **Flexibilité** : Support des majuscules ET minuscules (meilleure expérience utilisateur)
3. **Robustesse** : Arrêt automatique sur caractères invalides
4. **Format standard** : Les adresses sont traditionnellement en hexadécimal

### Caractéristiques techniques

- **Support insensible à la casse** : 'A'-'F' et 'a'-'f' sont acceptés
- **Arrêt automatique** : Sur espace, retour à la ligne, ou caractère invalide
- **Précision** : Jusqu'à 16 chiffres hexadécimaux (64 bits)

---

## Dépendances et intégration {#dependances}

### Variables externes nécessaires

Le module dépend de variables définies dans `data.asm` :

```9:12:utils.asm
EXTERN hStdIn:QWORD, hStdOut:QWORD
EXTERN bytesRead:DWORD, bytesWritten:DWORD
EXTERN inputBuffer:BYTE, outputBuffer:BYTE, hexBuffer:BYTE
EXTERN msgNewLine:BYTE
```

**Initialisation :**
- `hStdIn` et `hStdOut` : Initialisés dans `main.asm` via `GetStdHandle`
- Buffers : Définis dans `data.asm` comme variables globales
- `bytesRead` / `bytesWritten` : Utilisés par les API Windows

### Modules utilisant utils.asm

**Tous les modules** du projet utilisent `utils.asm` :

1. **main.asm** : `PrintString`, `GetUserChoice`
2. **memory.asm** : `PrintString`, `PrintHexQWord`, `PrintDecimal`
3. **program.asm** : `PrintString`, `PrintHexQWord`, `PrintDecimal`, `ParseHex`

### Chaîne de dépendances

```
data.asm (variables globales)
    ↑
main.asm (initialise hStdIn/hStdOut)
    ↑
utils.asm (utilise hStdIn/hStdOut)
    ↑
memory.asm, program.asm (utilisent les fonctions utilitaires)
```

### Compilation et linkage

Le module est compilé séparément :
```batch
ml64 /c /Zi utils.asm
```

Puis lié avec les autres modules :
```batch
link ... utils.obj ... /OUT:memory_simulator.exe
```

---

## Résumé

Le module `utils.asm` est la **fondation** du projet. Il fournit :

1. **6 fonctions publiques** utilisées par tous les autres modules
2. **87+ utilisations** dans tout le projet
3. **Abstraction complète** des API Windows pour l'E/S
4. **Cohérence** : Tous les affichages utilisent les mêmes fonctions

**Sans ce module :**
- ❌ Aucun affichage possible
- ❌ Aucune entrée utilisateur
- ❌ Pas de conversions numériques
- ❌ **Le projet ne fonctionnerait pas**

**Avec ce module :**
- ✅ Interface utilisateur complète
- ✅ Affichage professionnel des adresses et nombres
- ✅ Navigation dans le menu
- ✅ Protection d'accès mémoire fonctionnelle

---

**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025  
**Projet :** Simulateur de Gestion de Mémoire Virtuelle - INF22107-MS

