# Guide Complet du Projet - Simulateur de Gestion de Mémoire Virtuelle

**Projet :** Simulateur de Gestion de Mémoire Virtuelle  
**Cours :** INF22107-MS - Infrastructure de programmation  
**Université :** Université du Québec à Rimouski (UQAR)  
**Équipe :** OLYMPIO Marien, Léa BIKASSA, Joël KAYEMBA

---

## Table des Matières

1. [Vue d'ensemble du projet](#vue-ensemble)
2. [Objectifs pédagogiques](#objectifs)
3. [Architecture générale](#architecture-generale)
4. [Conception modulaire](#conception-modulaire)
5. [Fonctionnement détaillé](#fonctionnement-detaille)
6. [Mécanismes implémentés](#mecanismes)
7. [Choix de conception](#choix-conception)
8. [Flux d'exécution](#flux-execution)
9. [Structures de données](#structures-donnees)
10. [Algorithmes clés](#algorithmes-cles)

---

## Vue d'ensemble du projet {#vue-ensemble}

### Qu'est-ce que ce projet ?

Ce projet est un **simulateur de gestion de mémoire virtuelle** écrit en assembleur x64 (MASM) pour Windows. Il simule les mécanismes fondamentaux utilisés par les systèmes d'exploitation modernes pour gérer la mémoire, notamment :

- **RAM physique** : Mémoire principale limitée (4 Ko dans notre simulation)
- **Mémoire virtuelle** : Espace de stockage plus grand (10 Mo) simulant le disque dur
- **Swapping** : Mécanisme qui déplace les programmes entre RAM et mémoire virtuelle
- **Protection d'accès** : Vérification que chaque programme n'accède qu'à sa propre mémoire

### Pourquoi ce projet ?

Ce simulateur permet de **comprendre concrètement** comment fonctionne la gestion mémoire dans un système d'exploitation, en manipulant directement :
- Les adresses mémoire
- L'allocation et la libération
- Les mécanismes de swapping
- La protection mémoire

### Ce que le projet démontre

1. **Allocation mémoire dynamique** : Réservation d'espace pour les programmes
2. **Gestion de la fragmentation** : Organisation des programmes en mémoire
3. **Politique de remplacement** : Choix du programme à swapper (FIFO)
4. **Swap in/out** : Transfert de données entre RAM et mémoire virtuelle
5. **Protection d'accès** : Vérification des limites d'adressage

---

## Objectifs pédagogiques {#objectifs}

### Concepts abordés

**1. Mémoire virtuelle**
- Séparation entre adresses logiques et physiques
- Mécanisme de pagination simulé
- Espace d'adressage virtuel

**2. Gestion mémoire**
- Allocation contiguë
- Recherche d'espace libre
- Libération de mémoire

**3. Swapping**
- Swap out : RAM → Mémoire virtuelle
- Swap in : Mémoire virtuelle → RAM
- Politique de remplacement (FIFO)

**4. Protection mémoire**
- Vérification des limites
- Contrôle d'accès par programme
- Isolation des processus

**5. Programmation système**
- API Windows (VirtualAlloc, VirtualFree)
- Convention d'appel x64
- Manipulation bas niveau de la mémoire

---

## Architecture générale {#architecture-generale}

### Structure du projet

```
memory_simulator/
├── constants.inc          # Définitions partagées
├── data.asm              # Variables globales
├── utils.asm             # Fonctions utilitaires
├── memory.asm            # Gestion mémoire
├── program.asm           # Gestion programmes
├── main.asm              # Point d'entrée
└── compile.bat           # Script compilation
```

### Hiérarchie des modules

```
┌─────────────────────────────────────┐
│         main.asm                    │
│   (Point d'entrée, orchestration)   │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌─────────────┐  ┌──────────────┐
│  utils.asm  │  │ program.asm  │
│ (E/S, conv) │  │  (Logique    │
│             │  │  programmes) │
└──────┬──────┘  └──────┬───────┘
       │                │
       └────────┬───────┘
                │
                ▼
         ┌──────────────┐
         │ memory.asm   │
         │ (Logique     │
         │  mémoire)    │
         └──────┬───────┘
                │
                ▼
         ┌──────────────┐
         │  data.asm    │
         │ (Variables   │
         │  globales)   │
         └──────────────┘
```

### Flux de données

```
Utilisateur
    ↓
main.asm (menu, choix)
    ↓
program.asm ou memory.asm (traitement)
    ↓
utils.asm (affichage, conversions)
    ↓
API Windows (écriture console, allocation mémoire)
```

---

## Conception modulaire {#conception-modulaire}

### Pourquoi une architecture modulaire ?

**1. Séparation des responsabilités**
- Chaque module a un rôle clair et bien défini
- Facilite la compréhension et la maintenance
- Permet le développement en parallèle par l'équipe

**2. Réutilisabilité**
- Les fonctions utilitaires sont utilisables partout
- Pas de duplication de code
- Maintenance centralisée

**3. Testabilité**
- Chaque module peut être testé indépendamment
- Isolation des erreurs plus facile

### Description des modules

#### 1. constants.inc - Définitions partagées

**Rôle :** Contient toutes les constantes et structures partagées entre les modules.

**Contenu :**
- Tailles mémoire : `RAM_SIZE` (4 Ko), `VIRTUAL_MEM_SIZE` (10 Mo)
- Limites système : `MAX_PROGRAMS` (20), `MAX_PATH_LEN` (260)
- Constantes API Windows : `MEM_COMMIT`, `PAGE_READWRITE`, etc.
- Structure `ProgramInfo` : Définition de la structure de données pour les programmes

**Pourquoi séparé :**
- Évite la duplication des définitions
- Modification centralisée
- Facilite la maintenance

**Exemple de définition :**

```26:34:constants.inc
ProgramInfo STRUCT
    progName        BYTE MAX_PATH_LEN DUP(0)    ; Nom du fichier programme
    progSize        QWORD ?                      ; Taille mémoire requise
    startAddr       QWORD ?                      ; Adresse de début
    endAddr         QWORD ?                      ; Adresse de fin
    progStatus      DWORD ?                      ; 0=fermé, 1=en RAM, 2=swappé, 3=actif
    instructCount   DWORD ?                      ; Nombre d'instructions
    instructions    QWORD ?                      ; Pointeur vers instructions
ProgramInfo ENDS
```

#### 2. data.asm - Variables globales

**Rôle :** Définit toutes les variables globales utilisées par le système.

**Contenu :**
- **Mémoire** : `ramBase`, `virtualMemBase` (adresses de base)
- **Programmes** : `programs[]` (tableau de structures), `programCount`, `activeProgramIndex`
- **Messages** : Tous les messages affichés à l'utilisateur (en français)
- **Handles** : `hStdIn`, `hStdOut` (handles console)
- **Buffers** : `inputBuffer`, `fileBuffer`, `hexBuffer`

**Pourquoi centralisé :**
- Visibilité globale des données
- Gestion unique de l'état du système
- Facilite le débogage

**Exemple de variables :**

```6:11:data.asm
ramBase             QWORD ?                      ; Adresse de base de la RAM
virtualMemBase      QWORD ?                      ; Adresse de base mémoire virtuelle
programs            ProgramInfo MAX_PROGRAMS DUP(<>)  ; Tableau de programmes
programCount        DWORD 0                      ; Nombre de programmes chargés
activeProgramIndex  DWORD -1                     ; Index du programme actif
```

#### 3. utils.asm - Fonctions utilitaires

**Rôle :** Fournit toutes les fonctions d'entrée/sortie et de conversion.

**Fonctions principales :**
- `PrintString` : Affichage de chaînes
- `PrintHexQWord` : Affichage d'adresses en hexadécimal
- `PrintDecimal` : Affichage de nombres en décimal
- `GetUserChoice` : Lecture du choix utilisateur (menu)
- `GetUserInput` : Lecture générique d'entrée
- `ParseHex` : Conversion chaîne hex → nombre

**Pourquoi un module séparé :**
- Réutilisable par tous les autres modules
- Pas de dépendance à la bibliothèque C
- Cohérence de l'affichage

**Utilisation :**
- Appelé par **tous** les autres modules pour l'affichage
- Fondation de l'interface utilisateur

#### 4. memory.asm - Gestion de la mémoire

**Rôle :** Gère l'allocation, la libération et l'affichage de l'état de la mémoire.

**Fonctions principales :**
- `InitializeMemory` : Alloue RAM (4 Ko) et mémoire virtuelle (10 Mo) via VirtualAlloc
- `FreeMemory` : Libère la mémoire allouée via VirtualFree
- `FindFreeSpace` : Trouve un espace libre en RAM (algorithme First Fit)
- `FindFreeSpaceInVirtual` : Trouve un espace libre en mémoire virtuelle
- `DisplayMemoryState` : Affiche l'état complet de la mémoire

**Algorithme de recherche d'espace :**
1. Parcourt tous les programmes chargés
2. Trouve l'adresse de fin la plus élevée
3. Vérifie si l'espace restant est suffisant
4. Retourne l'adresse de début de l'espace libre

**Exemple d'utilisation :**

```119:194:memory.asm
FindFreeSpace PROC
    ; ... sauvegarde registres ...
    mov r12, rcx                    ; R12 = taille requise
    
    ; Si aucun programme, retourner le début de la RAM
    mov eax, programCount
    test eax, eax
    jz return_ram_base
    
    ; Trouver l'adresse la plus haute utilisée en RAM
    mov r13, ramBase                ; R13 = plus haute adresse de fin
    xor ebx, ebx                    ; Index = 0
    
find_highest:
    ; Parcourir tous les programmes
    ; Vérifier si en RAM (statut 1 ou 3)
    ; Trouver l'adresse de fin la plus haute
    
check_space:
    ; Vérifier si assez d'espace entre r13 et fin RAM
    ; Retourner l'adresse si espace suffisant, sinon 0
```

#### 5. program.asm - Gestion des programmes

**Rôle :** Gère le cycle de vie des programmes (chargement, exécution, fermeture, swapping).

**Fonctions principales :**
- `LoadProgram` : Charge un programme depuis un fichier en RAM
- `CloseProgram` : Ferme un programme et libère sa mémoire
- `UseProgram` : Active un programme (swap-in si nécessaire)
- `PerformSwapping` : Swappe un programme de RAM vers mémoire virtuelle (swap-out)
- `SwapInProgram` : Ramène un programme de mémoire virtuelle vers RAM (swap-in)
- `RequestMemoryAccess` : Vérifie si une adresse est accessible au programme actif
- `ListAllPrograms` : Affiche tous les programmes avec leur statut

**Workflow de chargement :**

1. Demander le nom du fichier
2. Lire le fichier
3. Parser la taille (première ligne)
4. Chercher un espace libre en RAM
5. Si pas d'espace, swapper un programme existant
6. Allouer l'espace et charger le programme
7. Créer une entrée dans le tableau `programs[]`

**Algorithme de swapping (FIFO) :**

```231:337:program.asm
PerformSwapping PROC
    ; Parcourir tous les programmes
    ; Trouver le premier programme en RAM (statut 1) et non actif
    ; Trouver un espace libre en mémoire virtuelle
    ; Copier les données de RAM vers VM (rep movsb)
    ; Mettre à jour les adresses et le statut
```

#### 6. main.asm - Point d'entrée

**Rôle :** Orchestre tous les modules et gère la boucle événementielle principale.

**Fonctions :**
- `main` : Point d'entrée, initialisation, boucle menu
- `DisplayMenu` : Affiche le menu principal

**Workflow principal :**

```31:106:main.asm
main PROC
    ; 1. Initialiser les handles console
    ; 2. Initialiser la mémoire (RAM + VM)
    ; 3. Boucle principale :
    ;    - Afficher le menu
    ;    - Lire le choix utilisateur
    ;    - Router vers la bonne fonction
    ;    - Retourner au menu
    ; 4. Libérer la mémoire et quitter
```

---

## Fonctionnement détaillé {#fonctionnement-detaille}

### Cycle de vie d'un programme

#### 1. Chargement (LoadProgram)

**Étapes :**

1. **Demande du fichier**
   ```assembly
   lea rcx, msgEnterFile
   call PrintString
   ; Lecture du nom via ReadConsoleA
   ```

2. **Ouverture et lecture du fichier**
   ```assembly
   call CreateFileA          ; Ouvrir le fichier
   call ReadFile             ; Lire le contenu
   call CloseHandle          ; Fermer le fichier
   ```

3. **Parsing de la taille**
   - Première ligne du fichier contient la taille en octets
   - Exemple : "512" signifie 512 octets
   - Fonction `ParseSize` convertit la chaîne en nombre

4. **Recherche d'espace libre**
   ```assembly
   mov rcx, programSize
   call FindFreeSpace        ; Chercher espace en RAM
   test rax, rax
   jnz space_found           ; Si trouvé, continuer
   ```

5. **Swapping si nécessaire**
   ```assembly
   ; Si pas d'espace
   call PerformSwapping      ; Swapper un programme existant
   call FindFreeSpace        ; Réessayer
   ```

6. **Allocation et chargement**
   ```assembly
   mov r12, rax              ; Adresse allouée
   call CreateProgramEntry   ; Créer l'entrée dans programs[]
   ; Copier le nom, sauvegarder taille et adresses
   ; Copier les instructions en mémoire (rep movsb)
   mov DWORD PTR [rbx].progStatus, 1  ; Statut = En RAM
   ```

#### 2. Activation (UseProgram)

**Étapes :**

1. **Sélection du programme**
   - Afficher la liste des programmes
   - L'utilisateur choisit un numéro

2. **Vérification du statut**
   ```assembly
   mov eax, [rbx].progStatus
   cmp eax, 2                ; Swappé ?
   je need_swap_in           ; Oui → swap-in nécessaire
   ```

3. **Swap-in si nécessaire**
   ```assembly
   mov rcx, rbx              ; Pointeur vers ProgramInfo
   call SwapInProgram        ; Ramener en RAM
   ```

4. **Activation**
   ```assembly
   mov DWORD PTR [rbx].progStatus, 3  ; Statut = Actif
   mov activeProgramIndex, r12d        ; Sauvegarder l'index
   ```

5. **Affichage des instructions**
   - Afficher toutes les instructions du programme depuis sa mémoire

#### 3. Vérification d'accès (RequestMemoryAccess)

**Étapes :**

1. **Vérifier qu'un programme est actif**
   ```assembly
   mov eax, activeProgramIndex
   cmp eax, -1
   je no_active              ; Pas de programme actif
   ```

2. **Demander l'adresse**
   ```assembly
   lea rcx, msgEnterAddr
   call PrintString
   ; Lecture de l'adresse en hexadécimal
   lea rsi, inputBuffer
   call ParseHex             ; Convertir en nombre
   ```

3. **Vérifier les limites**
   ```assembly
   mov rdx, [rbx].startAddr
   cmp r12, rdx              ; Adresse < début ?
   jl access_denied
   
   mov rdx, [rbx].endAddr
   cmp r12, rdx              ; Adresse >= fin ?
   jge access_denied
   ```

4. **Afficher le résultat**
   - Si autorisé : "Acces AUTORISE"
   - Si refusé : "Acces REFUSE! Adresse hors limites."

#### 4. Fermeture (CloseProgram)

**Étapes :**

1. **Sélection du programme**
   - Afficher la liste
   - L'utilisateur choisit un numéro

2. **Marquer comme fermé**
   ```assembly
   mov DWORD PTR [rbx].progStatus, 0  ; Statut = Fermé
   ```

3. **Désactiver si actif**
   ```assembly
   cmp rsi, rbx              ; Est-ce le programme actif ?
   jne not_active
   mov DWORD PTR activeProgramIndex, -1
   ```

**Note :** La mémoire n'est pas réellement libérée (pour simplicité), mais le statut passe à "Fermé" et l'espace peut être réutilisé.

---

## Mécanismes implémentés {#mecanismes}

### 1. Allocation mémoire

**Méthode :** Allocation contiguë (First Fit)

**Algorithme :**
1. Parcourir tous les programmes chargés
2. Identifier l'adresse de fin la plus élevée
3. Vérifier si l'espace restant est suffisant
4. Si oui, retourner l'adresse de début de l'espace libre
5. Si non, retourner 0 (pas d'espace)

**Avantages :**
- Simple à implémenter
- Rapide pour de petits nombres de programmes
- Pas de fragmentation interne (allocation exacte)

**Inconvénients :**
- Peut créer de la fragmentation externe
- Ne réutilise pas les espaces libérés (programmes fermés)

### 2. Swapping (Swap-out)

**Quand :** Quand la RAM est pleine et qu'on veut charger un nouveau programme

**Algorithme FIFO (First-In-First-Out) :**
1. Parcourir tous les programmes
2. Trouver le premier programme en RAM (statut 1) et non actif (statut ≠ 3)
3. Trouver un espace libre en mémoire virtuelle
4. Copier les données de RAM vers mémoire virtuelle (`rep movsb`)
5. Mettre à jour les adresses (`startAddr`, `endAddr`)
6. Changer le statut à 2 (Swappé)

**Code clé :**

```297:308:program.asm
; Copier les données de RAM vers mémoire virtuelle
mov rsi, [rbx].ProgramInfo.startAddr    ; Source (RAM)
mov rdi, r13                            ; Destination (VM)
mov rcx, [rbx].ProgramInfo.progSize     ; Taille
rep movsb                               ; Copier byte par byte

; Mettre à jour les informations du programme
mov [rbx].ProgramInfo.startAddr, r13
mov rax, r13
add rax, [rbx].ProgramInfo.progSize
mov [rbx].ProgramInfo.endAddr, rax
mov DWORD PTR [rbx].ProgramInfo.progStatus, 2  ; Statut = Swappé
```

**Pourquoi FIFO ?**
- Simple à implémenter
- Équitable (premier arrivé, premier sorti)
- Pas besoin de métriques supplémentaires

### 3. Swap-in

**Quand :** Quand on veut activer un programme qui est swappé

**Algorithme :**
1. Vérifier que le programme est bien swappé (statut 2)
2. Chercher un espace libre en RAM
3. Si pas d'espace, swapper un autre programme (swap-out)
4. Copier les données de mémoire virtuelle vers RAM
5. Mettre à jour les adresses
6. Changer le statut à 1 (En RAM) puis 3 (Actif)

**Code clé :**

```378:392:program.asm
swapin_space_found:
    mov r12, rax                    ; R12 = nouvelle adresse en RAM
    
    ; Copier depuis VM vers RAM
    mov rsi, [rbx].ProgramInfo.startAddr    ; Source (VM)
    mov rdi, r12                            ; Destination (RAM)
    mov rcx, [rbx].ProgramInfo.progSize
    rep movsb                               ; Copier
    
    ; Mettre à jour les informations
    mov [rbx].ProgramInfo.startAddr, r12
    mov rax, r12
    add rax, [rbx].ProgramInfo.progSize
    mov [rbx].ProgramInfo.endAddr, rax
    mov DWORD PTR [rbx].ProgramInfo.progStatus, 1  ; Statut = En RAM
```

### 4. Protection d'accès

**Principe :** Chaque programme ne peut accéder qu'à sa propre plage d'adresses

**Vérification :**
```assembly
; R12 = adresse demandée
; RBX = pointeur vers ProgramInfo du programme actif

mov rdx, [rbx].startAddr
cmp r12, rdx              ; Adresse < début ?
jl access_denied          ; Refusé

mov rdx, [rbx].endAddr
cmp r12, rdx              ; Adresse >= fin ?
jge access_denied         ; Refusé

; Si on arrive ici, l'accès est autorisé
```

**Limites vérifiées :**
- Adresse doit être ≥ `startAddr`
- Adresse doit être < `endAddr`

---

## Choix de conception {#choix-conception}

### 1. Taille de la RAM (4 Ko)

**Choix :** 4 Ko au lieu d'une taille plus grande

**Raison :**
- Permet de **démontrer facilement** le swapping
- Avec des programmes de 512 octets à 1 Ko, la RAM se remplit rapidement
- Facilite les tests et la compréhension
- Taille réaliste pour un système embarqué

**Impact :**
- Le swapping se déclenche fréquemment
- Démontre bien le mécanisme
- Tests plus rapides

### 2. Mémoire virtuelle (10 Mo)

**Choix :** 10 Mo, beaucoup plus grand que la RAM

**Raison :**
- Simule un disque dur (beaucoup plus grand que la RAM)
- Permet de charger beaucoup de programmes
- Démontre l'avantage de la mémoire virtuelle (plus d'espace)

### 3. Allocation contiguë

**Choix :** First Fit plutôt que Best Fit ou Worst Fit

**Raison :**
- **Simplicité** : Plus facile à implémenter
- **Performance** : Plus rapide (premier espace trouvé)
- **Suffisant** pour la démonstration pédagogique

**Alternative considérée :** Best Fit (trouver le plus petit espace suffisant)
- Rejeté car plus complexe sans gain significatif pour le projet

### 4. Politique FIFO pour le swapping

**Choix :** First-In-First-Out plutôt que LRU (Least Recently Used)

**Raison :**
- **Simplicité** : Pas besoin de timestamp ou compteur
- **Équitable** : Premier arrivé, premier sorti
- **Suffisant** pour démontrer le concept

**Alternative considérée :** LRU
- Rejeté car nécessite de suivre l'ordre d'accès
- Plus complexe à implémenter
- Gain négligeable pour la démonstration

### 5. Langage : Assembleur x64

**Choix :** Assembleur plutôt que C/C++

**Raison :**
- **Contrôle total** : Manipulation directe de la mémoire
- **Pédagogique** : Compréhension profonde du fonctionnement
- **Exigence du cours** : Démonstration de compétence en programmation système

**Avantages :**
- Pas de dépendances (pas de bibliothèque C runtime)
- Performance maximale
- Compréhension bas niveau

**Inconvénients :**
- Code plus verbeux
- Développement plus long
- Maintenance plus difficile

### 6. Architecture modulaire

**Choix :** Plusieurs fichiers .asm plutôt qu'un seul

**Raison :**
- **Séparation des responsabilités** : Chaque module a un rôle clair
- **Développement en équipe** : Chaque membre peut travailler sur un module
- **Maintenance** : Plus facile de localiser et corriger les erreurs
- **Réutilisabilité** : Fonctions utilisables partout

### 7. Format des fichiers programmes

**Choix :** Fichiers texte avec taille en première ligne

**Format :**
```
512
MOV AX, BX
ADD AX, 4
SUB CX, AX
...
```

**Raison :**
- **Simplicité** : Facile à créer et modifier
- **Lisible** : Les instructions sont visibles en texte
- **Flexible** : Facile d'ajouter de nouveaux programmes de test

**Alternative considérée :** Format binaire
- Rejeté car plus complexe et moins lisible

### 8. États des programmes

**Choix :** 4 états (Fermé, En RAM, Swappé, Actif)

**Raison :**
- **Clarté** : État explicite de chaque programme
- **Fonctionnalité** : Permet de distinguer programme actif vs inactif
- **Débogage** : Facile de voir où se trouve chaque programme

**États définis :**
- 0 = Fermé (libéré)
- 1 = En RAM (chargé mais inactif)
- 2 = Swappé (en mémoire virtuelle)
- 3 = Actif (en RAM et en cours d'utilisation)

---

## Flux d'exécution {#flux-execution}

### Démarrage du programme

```
1. main.asm (main)
   ↓
2. Obtenir handles console (GetStdHandle)
   → hStdIn, hStdOut stockés dans data.asm
   ↓
3. Initialiser mémoire (InitializeMemory)
   → Allouer RAM (4 Ko) via VirtualAlloc
   → Allouer mémoire virtuelle (10 Mo) via VirtualAlloc
   → ramBase et virtualMemBase stockés dans data.asm
   ↓
4. Entrer dans la boucle menu
```

### Boucle principale

```
┌─────────────────────────────────────┐
│  1. DisplayMenu                     │
│     → PrintString (menuTitle)       │
│     → PrintString (menuOptions)     │
│                                     │
│  2. GetUserChoice                   │
│     → ReadConsoleA                  │
│     → Conversion ASCII → nombre     │
│                                     │
│  3. Routage selon choix             │
│     1 → LoadProgram                 │
│     2 → DisplayMemoryState          │
│     3 → CloseProgram                │
│     4 → UseProgram                  │
│     5 → RequestMemoryAccess         │
│     6 → ListAllPrograms             │
│     7 → exit_program                │
│                                     │
│  4. Retour au menu (sauf option 7)  │
└─────────────────────────────────────┘
```

### Exemple : Chargement d'un programme

```
Utilisateur choisit option 1
    ↓
main.asm → option_load_program
    ↓
program.asm → LoadProgram
    ↓
1. PrintString ("Entrez le nom...")
2. ReadConsoleA (nom du fichier)
3. CreateFileA (ouvrir fichier)
4. ReadFile (lire contenu)
5. ParseSize (parser taille)
    ↓
6. FindFreeSpace (chercher espace RAM)
    ↓
   ┌─ Espace trouvé ?
   │
   ├─ Oui → Continuer
   │
   └─ Non → PerformSwapping
            → FindFreeSpaceInVirtual
            → Copier RAM → VM
            → FindFreeSpace (réessayer)
    ↓
7. CreateProgramEntry (créer entrée)
8. Copier nom, taille, adresses
9. rep movsb (copier instructions en RAM)
10. PrintString ("Programme chargé!")
    ↓
Retour au menu
```

### Exemple : Activation d'un programme swappé

```
Utilisateur choisit option 4
    ↓
main.asm → option_use_program
    ↓
program.asm → UseProgram
    ↓
1. ListAllPrograms (afficher liste)
2. GetUserChoice (choisir programme)
    ↓
3. Vérifier statut
    ↓
   ┌─ Statut = 2 (Swappé) ?
   │
   ├─ Oui → SwapInProgram
   │        → FindFreeSpace (RAM)
   │        → Si pas d'espace :
   │           → PerformSwapping
   │        → Copier VM → RAM
   │        → Mettre statut = 1
   │
   └─ Non → Continuer
    ↓
4. Mettre statut = 3 (Actif)
5. Afficher instructions
    ↓
Retour au menu
```

---

## Structures de données {#structures-donnees}

### Structure ProgramInfo

**Définition :**

```26:34:constants.inc
ProgramInfo STRUCT
    progName        BYTE MAX_PATH_LEN DUP(0)    ; Nom du fichier (260 bytes)
    progSize        QWORD ?                      ; Taille en octets (8 bytes)
    startAddr       QWORD ?                      ; Adresse de début (8 bytes)
    endAddr         QWORD ?                      ; Adresse de fin (8 bytes)
    progStatus      DWORD ?                      ; État (4 bytes)
    instructCount   DWORD ?                      ; Nb instructions (4 bytes)
    instructions    QWORD ?                      ; Pointeur instructions (8 bytes)
ProgramInfo ENDS
```

**Taille totale :** ~296 bytes par programme

**Champs expliqués :**

1. **progName** : Nom du fichier programme (ex: "prog2.txt")
2. **progSize** : Taille mémoire occupée (ex: 512 octets)
3. **startAddr** : Adresse de début en mémoire (RAM ou VM)
4. **endAddr** : Adresse de fin (startAddr + progSize)
5. **progStatus** : État actuel (0=Fermé, 1=En RAM, 2=Swappé, 3=Actif)
6. **instructCount** : Nombre d'instructions (non utilisé actuellement)
7. **instructions** : Pointeur vers les instructions (non utilisé, données directement en mémoire)

### Tableau des programmes

**Définition :**

```9:11:data.asm
programs            ProgramInfo MAX_PROGRAMS DUP(<>)  ; Tableau de 20 programmes
programCount        DWORD 0                      ; Nombre actuel de programmes
activeProgramIndex  DWORD -1                     ; Index du programme actif (-1 = aucun)
```

**Accès à un programme :**

```assembly
; Obtenir le programme à l'index EBX
mov eax, ebx
imul rax, rax, SIZEOF ProgramInfo    ; Calculer offset
lea rsi, programs
add rsi, rax                          ; RSI pointe vers le programme

; Accéder aux champs
mov rax, [rsi].ProgramInfo.progSize
mov rcx, [rsi].ProgramInfo.startAddr
mov edx, [rsi].ProgramInfo.progStatus
```

### Variables globales mémoire

```6:8:data.asm
ramBase             QWORD ?                      ; Adresse de base de la RAM
virtualMemBase      QWORD ?                      ; Adresse de base mémoire virtuelle
```

**Initialisation :**

```40:65:memory.asm
; Allouer la RAM simulée (4 Ko)
xor rcx, rcx                    ; lpAddress = NULL
mov rdx, RAM_SIZE               ; dwSize = 4 Ko
mov r8d, MEM_COMMIT or MEM_RESERVE
mov r9d, PAGE_READWRITE
call VirtualAlloc
mov ramBase, rax                ; Sauvegarder l'adresse

; Allouer la mémoire virtuelle (10 Mo)
xor rcx, rcx
mov rdx, VIRTUAL_MEM_SIZE       ; dwSize = 10 Mo
mov r8d, MEM_COMMIT or MEM_RESERVE
mov r9d, PAGE_READWRITE
call VirtualAlloc
mov virtualMemBase, rax         ; Sauvegarder l'adresse
```

---

## Algorithmes clés {#algorithmes-cles}

### 1. Recherche d'espace libre (First Fit)

**Algorithme :**

```
FindFreeSpace(taille_requise):
    si aucun programme chargé:
        retourner ramBase
    
    plus_haute_fin = ramBase
    
    pour chaque programme:
        si programme en RAM (statut 1 ou 3):
            si programme.endAddr > plus_haute_fin:
                plus_haute_fin = programme.endAddr
    
    espace_disponible = (ramBase + RAM_SIZE) - plus_haute_fin
    
    si espace_disponible >= taille_requise:
        retourner plus_haute_fin
    sinon:
        retourner 0 (pas d'espace)
```

**Complexité :** O(n) où n = nombre de programmes

**Exemple :**
- RAM : 0x1000 à 0x2000 (4 Ko)
- Programme 1 : 0x1000 à 0x1200 (512 octets)
- Programme 2 : 0x1200 à 0x1600 (1024 octets)
- Plus haute fin : 0x1600
- Espace disponible : 0x2000 - 0x1600 = 1024 octets
- Nouveau programme (512 octets) → Alloué à 0x1600

### 2. Swapping FIFO

**Algorithme :**

```
PerformSwapping():
    pour chaque programme (ordre chronologique):
        si programme.statut == 1 (En RAM) et pas actif:
            victime = programme
            sortir de la boucle
    
    si aucune victime trouvée:
        retourner échec
    
    trouver espace libre en mémoire virtuelle
    si pas d'espace:
        retourner échec
    
    copier victime de RAM vers VM
    mettre à jour victime.startAddr et victime.endAddr
    victime.statut = 2 (Swappé)
    
    retourner succès
```

**Complexité :** O(n) pour trouver la victime + O(taille) pour copier

### 3. Vérification d'accès mémoire

**Algorithme :**

```
RequestMemoryAccess(adresse_demandee):
    si pas de programme actif:
        retourner erreur
    
    programme = programmes[activeProgramIndex]
    
    si adresse_demandee < programme.startAddr:
        retourner accès refusé
    
    si adresse_demandee >= programme.endAddr:
        retourner accès refusé
    
    retourner accès autorisé
```

**Complexité :** O(1)

---

## Conclusion

Ce projet démontre de manière concrète les mécanismes fondamentaux de gestion de mémoire dans un système d'exploitation :

1. **Allocation dynamique** : Réservation d'espace pour les programmes
2. **Swapping** : Transfert entre RAM et mémoire virtuelle
3. **Protection** : Vérification des limites d'accès
4. **Gestion d'état** : Suivi de la localisation de chaque programme

L'architecture modulaire permet une compréhension claire de chaque composant et facilite la maintenance. L'utilisation de l'assembleur x64 offre un contrôle total sur la mémoire et démontre une compréhension profonde des mécanismes système.

**Points forts du projet :**
- ✅ Architecture modulaire claire
- ✅ Implémentation complète des fonctionnalités demandées
- ✅ Interface utilisateur intuitive
- ✅ Code commenté et lisible
- ✅ Démonstration pédagogique efficace

**Compétences démontrées :**
- Maîtrise de l'assembleur x64
- Compréhension de la gestion mémoire
- Utilisation des API Windows
- Conception d'algorithmes
- Architecture logicielle

---

**Auteurs :** OLYMPIO Marien, Léa BIKASSA, Joël KAYEMBA  
**Date :** Décembre 2025  
**Cours :** INF22107-MS - Infrastructure de programmation  
**Université :** Université du Québec à Rimouski (UQAR)

