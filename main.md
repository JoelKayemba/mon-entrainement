# Documentation Détaillée - Module main.asm

**Projet :** Simulateur de Gestion de Mémoire Virtuelle  
**Fichier :** `main.asm`  
**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025

---

## Table des Matières

1. [Vue d'ensemble du module](#vue-ensemble)
2. [Structure et dépendances](#structure-dependances)
3. [Fonction main - Point d'entrée](#fonction-main)
4. [Fonction DisplayMenu](#fonction-displaymenu)
5. [Flux d'exécution complet](#flux-execution)
6. [Initialisation du système](#initialisation)
7. [Boucle événementielle](#boucle-evenementielle)
8. [Routage des choix](#routage-choix)
9. [Gestion de la terminaison](#terminaison)

---

## Vue d'ensemble du module {#vue-ensemble}

### Rôle du module

Le module `main.asm` est le **point d'entrée** du programme et l'**orchestrateur central** qui coordonne tous les autres modules. Il gère :
- L'initialisation du système
- L'affichage du menu principal
- La boucle événementielle
- Le routage des choix utilisateur
- La terminaison propre du programme

### Pourquoi ce module est crucial

**Sans main.asm :**
- ❌ Le programme ne pourrait pas démarrer (pas de point d'entrée)
- ❌ Aucune coordination entre les modules
- ❌ Pas d'interface utilisateur (pas de menu)
- ❌ Les autres modules existeraient mais seraient inutilisables

**Avec main.asm :**
- ✅ Point d'entrée clair (`main`)
- ✅ Initialisation complète du système
- ✅ Interface utilisateur fonctionnelle
- ✅ Orchestration de tous les modules
- ✅ Cycle de vie complet (démarrage → exécution → arrêt)

### Fonctions exportées

Le module exporte une seule fonction publique :
- **`main`** : Point d'entrée principal du programme

Le module définit également une fonction interne :
- **`DisplayMenu`** : Affiche le menu principal (utilisée uniquement par `main`)

---

## Structure et dépendances {#structure-dependances}

### Déclarations externes

Le module dépend de nombreuses fonctions et variables définies dans les autres modules :

#### Variables externes (data.asm)

```5:6:main.asm
EXTERN hStdIn:QWORD, hStdOut:QWORD
EXTERN menuTitle:BYTE, menuOptions:BYTE
```

**Explication :**
- `hStdIn` / `hStdOut` : Handles de la console (initialisés dans `main`)
- `menuTitle` / `menuOptions` : Chaînes du menu (définies dans `data.asm`)

#### API Windows

```9:11:main.asm
EXTERN GetStdHandle:PROC
EXTERN ExitProcess:PROC
```

**Explication :**
- `GetStdHandle` : Obtient les handles de la console standard
- `ExitProcess` : Termine le processus proprement

#### Fonctions utilitaires (utils.asm)

```13:15:main.asm
EXTERN PrintString:PROC
EXTERN GetUserChoice:PROC
```

**Explication :**
- `PrintString` : Pour afficher le menu et les messages
- `GetUserChoice` : Pour lire le choix de l'utilisateur (1-7)

#### Fonctions mémoire (memory.asm)

```17:20:main.asm
EXTERN InitializeMemory:PROC
EXTERN FreeMemory:PROC
EXTERN DisplayMemoryState:PROC
```

**Explication :**
- `InitializeMemory` : Alloue la RAM et la mémoire virtuelle
- `FreeMemory` : Libère la mémoire allouée
- `DisplayMemoryState` : Affiche l'état de la mémoire (option 2)

#### Fonctions programmes (program.asm)

```22:27:main.asm
EXTERN LoadProgram:PROC
EXTERN CloseProgram:PROC
EXTERN UseProgram:PROC
EXTERN RequestMemoryAccess:PROC
EXTERN ListAllPrograms:PROC
```

**Explication :**
- `LoadProgram` : Charge un programme en mémoire (option 1)
- `CloseProgram` : Ferme un programme (option 3)
- `UseProgram` : Active un programme (option 4)
- `RequestMemoryAccess` : Vérifie l'accès mémoire (option 5)
- `ListAllPrograms` : Liste tous les programmes (option 6)

### Fonction publique

```30:30:main.asm
PUBLIC main
```

**Explication :**
- `main` est la fonction publique qui sert de point d'entrée
- Le linker cherche cette fonction grâce à `/ENTRY:main` dans `compile.bat`

---

## Fonction main - Point d'entrée {#fonction-main}

### Spécification complète

```31:106:main.asm
PUBLIC main
main PROC
    ; Aligner la pile (Windows x64 calling convention)
    push rbp
    mov rbp, rsp
    sub rsp, 40h                ; Allouer shadow space + alignement
    
    ; Obtenir les handles console
    mov rcx, STD_OUTPUT_HANDLE
    call GetStdHandle
    mov hStdOut, rax
    
    mov rcx, STD_INPUT_HANDLE
    call GetStdHandle
    mov hStdIn, rax
    
    ; Initialiser la RAM et la mémoire virtuelle
    call InitializeMemory
    test rax, rax
    jz exit_program
    
    ; Boucle principale du menu
menu_loop:
    call DisplayMenu
    call GetUserChoice
    
    ; Traiter le choix (rax contient le choix)
    cmp rax, 1
    je option_load_program
    cmp rax, 2
    je option_display_memory
    cmp rax, 3
    je option_close_program
    cmp rax, 4
    je option_use_program
    cmp rax, 5
    je option_memory_access
    cmp rax, 6
    je option_list_programs
    cmp rax, 7
    je exit_program
    jmp menu_loop

option_load_program:
    call LoadProgram
    jmp menu_loop

option_display_memory:
    call DisplayMemoryState
    jmp menu_loop

option_close_program:
    call CloseProgram
    jmp menu_loop

option_use_program:
    call UseProgram
    jmp menu_loop

option_memory_access:
    call RequestMemoryAccess
    jmp menu_loop

option_list_programs:
    call ListAllPrograms
    jmp menu_loop

exit_program:
    ; Libérer la mémoire
    call FreeMemory
    
    ; Terminer le programme
    mov rsp, rbp
    pop rbp
    xor rcx, rcx
    call ExitProcess
main ENDP
```

### Paramètres d'entrée

- **Aucun** (fonction sans paramètres)
- Point d'entrée du programme, appelé par le système d'exploitation

### Valeur de retour

- **Aucune** (la fonction ne retourne jamais)
- Le programme se termine via `ExitProcess`

### Phases d'exécution

La fonction `main` s'exécute en 6 phases distinctes :

#### Phase 1 : Initialisation de la pile

```33:35:main.asm
push rbp
mov rbp, rsp
sub rsp, 40h                ; Allouer shadow space + alignement
```

**Explication :**
- Conforme à la convention d'appel Windows x64
- `sub rsp, 40h` : Réserve 32 bytes (shadow space) + 16 bytes (alignement)
- Nécessaire pour les appels d'API Windows

**Pourquoi :**
- Les API Windows s'attendent à ce que la pile soit alignée sur 16 bytes
- Le shadow space est réservé même si non utilisé (requis par la convention)

#### Phase 2 : Obtention des handles console

```37:44:main.asm
; Obtenir les handles console
mov rcx, STD_OUTPUT_HANDLE
call GetStdHandle
mov hStdOut, rax

mov rcx, STD_INPUT_HANDLE
call GetStdHandle
mov hStdIn, rax
```

**Constantes utilisées :**
- `STD_OUTPUT_HANDLE` = -11 : Handle pour la sortie console
- `STD_INPUT_HANDLE` = -10 : Handle pour l'entrée console

**Explication :**
- `GetStdHandle` retourne un handle dans RAX
- Les handles sont stockés dans les variables globales `hStdOut` et `hStdIn`
- **Critique** : Ces handles sont nécessaires pour toutes les fonctions d'E/S de `utils.asm`

**Pourquoi ici :**
- Les handles doivent être obtenus **avant** toute opération d'E/S
- Stockés globalement pour être accessibles à tous les modules
- Initialisés une seule fois au démarrage

**Si échec :**
- `GetStdHandle` retourne INVALID_HANDLE_VALUE (-1)
- Le programme continuerait mais les E/S échoueraient
- Dans un vrai programme, on devrait vérifier les handles

#### Phase 3 : Initialisation de la mémoire

```46:49:main.asm
; Initialiser la RAM et la mémoire virtuelle
call InitializeMemory
test rax, rax
jz exit_program
```

**Explication :**
- `InitializeMemory` alloue :
  - RAM simulée : 4 Ko via `VirtualAlloc`
  - Mémoire virtuelle : 10 Mo via `VirtualAlloc`
- Retourne 1 si succès, 0 si échec

**Vérification :**
- `test rax, rax` : Vérifie si RAX = 0 (échec)
- `jz exit_program` : Si échec, quitter immédiatement

**Pourquoi cette vérification :**
- **Sans mémoire, le simulateur ne peut pas fonctionner**
- Mieux vaut quitter proprement que continuer avec des pointeurs invalides
- Évite les crashes ultérieurs

**Si succès :**
- `ramBase` et `virtualMemBase` sont initialisés dans `data.asm`
- Le système est prêt à charger des programmes

#### Phase 4 : Boucle événementielle principale

```51:71:main.asm
; Boucle principale du menu
menu_loop:
    call DisplayMenu
    call GetUserChoice
    
    ; Traiter le choix (rax contient le choix)
    cmp rax, 1
    je option_load_program
    cmp rax, 2
    je option_display_memory
    cmp rax, 3
    je option_close_program
    cmp rax, 4
    je option_use_program
    cmp rax, 5
    je option_memory_access
    cmp rax, 6
    je option_list_programs
    cmp rax, 7
    je exit_program
    jmp menu_loop
```

**Caractéristiques :**
- **Boucle infinie** : Continue jusqu'à l'option 7 (Quitter)
- **Menu affiché en continu** : L'utilisateur voit toujours les options
- **Choix capturé** : `GetUserChoice` lit l'entrée utilisateur
- **Routage** : Chaque choix déclenche l'action correspondante

**Architecture de routage :**
- Série de comparaisons (`cmp`) suivies de sauts conditionnels (`je`)
- Simple, lisible, facile à maintenir
- Extensible : Facile d'ajouter de nouvelles options

**Gestion des choix invalides :**
- Si le choix n'est pas 1-7, aucune condition n'est vraie
- `jmp menu_loop` : Retour au menu (réaffiche et redemande)

#### Phase 5 : Exécution des actions

```73:95:main.asm
option_load_program:
    call LoadProgram
    jmp menu_loop

option_display_memory:
    call DisplayMemoryState
    jmp menu_loop

option_close_program:
    call CloseProgram
    jmp menu_loop

option_use_program:
    call UseProgram
    jmp menu_loop

option_memory_access:
    call RequestMemoryAccess
    jmp menu_loop

option_list_programs:
    call ListAllPrograms
    jmp menu_loop
```

**Pattern utilisé :**
- Chaque option est isolée dans son propre label
- Appel de la fonction correspondante
- `jmp menu_loop` : Retour au menu après l'action

**Pourquoi ce pattern :**
- **Modularité** : Chaque action est indépendante
- **Maintenance** : Facile de modifier une option sans affecter les autres
- **Débogage** : Facile de localiser les problèmes
- **Clarté** : Code très lisible

**Ordre des options :**
1. Charger un programme
2. Afficher état mémoire
3. Fermer un programme
4. Utiliser un programme
5. Demander accès mémoire
6. Lister tous les programmes
7. Quitter

#### Phase 6 : Terminaison

```97:105:main.asm
exit_program:
    ; Libérer la mémoire
    call FreeMemory
    
    ; Terminer le programme
    mov rsp, rbp
    pop rbp
    xor rcx, rcx
    call ExitProcess
```

**Étapes de terminaison :**

1. **Libération de la mémoire**
   ```assembly
   call FreeMemory
   ```
   - Libère la RAM simulée via `VirtualFree`
   - Libère la mémoire virtuelle via `VirtualFree`
   - **Bonnes pratiques** : Libération explicite même si le système le ferait automatiquement

2. **Restauration de la pile**
   ```assembly
   mov rsp, rbp
   pop rbp
   ```
   - Restaure la pile à son état initial
   - Nécessaire pour la propreté du code (même si on va quitter)

3. **Terminaison du processus**
   ```assembly
   xor rcx, rcx        ; Code de sortie = 0 (succès)
   call ExitProcess
   ```
   - `ExitProcess` termine le processus
   - Code de sortie 0 = succès (convention Windows)
   - Le processus se termine proprement

---

## Fonction DisplayMenu {#fonction-displaymenu}

### Spécification

```110:124:main.asm
DisplayMenu PROC
    push rbp
    mov rbp, rsp
    sub rsp, 20h
    
    lea rcx, menuTitle
    call PrintString
    
    lea rcx, menuOptions
    call PrintString
    
    mov rsp, rbp
    pop rbp
    ret
DisplayMenu ENDP
```

### Paramètres d'entrée

- **Aucun** (fonction sans paramètres)

### Valeur de retour

- **Aucune** (procédure void)
- La fonction affiche le menu sur la console

### Comment ça marche ?

**Étape 1 : Préparation de la pile**
```assembly
push rbp
mov rbp, rsp
sub rsp, 20h      ; Shadow space (20h = 32 bytes)
```

**Étape 2 : Affichage du titre**
```assembly
lea rcx, menuTitle
call PrintString
```

**menuTitle** (défini dans `data.asm`) :
```
"=== SYSTEME DE GESTION MEMOIRE VIRTUELLE ==="
```

**Étape 3 : Affichage des options**
```assembly
lea rcx, menuOptions
call PrintString
```

**menuOptions** (défini dans `data.asm`) :
```
"1. Charger un programme"
"2. Afficher etat memoire"
"3. Fermer un programme"
"4. Utiliser un programme"
"5. Demander acces memoire"
"6. Lister tous les programmes"
"7. Quitter"
"Choix: "
```

**Étape 4 : Restauration et retour**
```assembly
mov rsp, rbp
pop rbp
ret
```

### Utilisation

**Appelée dans la boucle principale :**
```52:53:main.asm
menu_loop:
    call DisplayMenu
```

**Résultat :** Le menu s'affiche à chaque itération de la boucle, permettant à l'utilisateur de voir les options disponibles.

### Pourquoi une fonction séparée ?

1. **Séparation des responsabilités** : Code du menu isolé
2. **Réutilisabilité** : Peut être appelée plusieurs fois
3. **Maintenabilité** : Facile de modifier l'affichage du menu
4. **Clarté** : Le code de `main` reste lisible

---

## Flux d'exécution complet {#flux-execution}

### Démarrage du programme

```
1. Système d'exploitation charge l'exécutable
   ↓
2. Système appelle main() (point d'entrée)
   ↓
3. Initialisation de la pile
   ↓
4. Obtention des handles console
   → hStdOut et hStdIn stockés dans data.asm
   ↓
5. Initialisation de la mémoire
   → RAM (4 Ko) et mémoire virtuelle (10 Mo) allouées
   → ramBase et virtualMemBase stockés dans data.asm
   ↓
6. Entrée dans la boucle menu
```

### Boucle principale (répétée jusqu'à option 7)

```
┌──────────────────────────────────────────┐
│  menu_loop:                              │
│                                          │
│  1. DisplayMenu()                        │
│     → Affiche le titre et les options   │
│                                          │
│  2. GetUserChoice()                      │
│     → Lit l'entrée utilisateur (1-7)    │
│     → Retourne le choix dans RAX        │
│                                          │
│  3. Routage selon choix:                 │
│     Choix 1 → option_load_program        │
│     Choix 2 → option_display_memory      │
│     Choix 3 → option_close_program       │
│     Choix 4 → option_use_program         │
│     Choix 5 → option_memory_access       │
│     Choix 6 → option_list_programs       │
│     Choix 7 → exit_program               │
│     Autre → menu_loop (réafficher)       │
│                                          │
│  4. Exécution de l'action                │
│     → Appel de la fonction correspondante│
│                                          │
│  5. Retour au menu (jmp menu_loop)       │
│     (sauf si option 7)                   │
└──────────────────────────────────────────┘
```

### Exemple : Chargement d'un programme

```
Utilisateur choisit option 1
    ↓
main.asm : cmp rax, 1
    ↓
main.asm : je option_load_program
    ↓
main.asm : call LoadProgram
    ↓
program.asm : LoadProgram
    → Demande le nom du fichier
    → Lit le fichier
    → Parse la taille
    → Cherche un espace libre en RAM
    → Si pas d'espace : PerformSwapping
    → Charge le programme en RAM
    → Crée une entrée dans programs[]
    ↓
main.asm : jmp menu_loop
    ↓
Retour au menu (afficher et attendre choix suivant)
```

### Terminaison

```
Utilisateur choisit option 7
    ↓
main.asm : je exit_program
    ↓
main.asm : call FreeMemory
    → VirtualFree(ramBase)
    → VirtualFree(virtualMemBase)
    ↓
main.asm : call ExitProcess(0)
    ↓
Programme se termine (code de sortie 0 = succès)
```

---

## Initialisation du système {#initialisation}

### Ordre d'initialisation

L'initialisation suit un ordre précis :

```
1. Pile (shadow space + alignement)
   ↓
2. Handles console
   → hStdOut (nécessaire pour tous les affichages)
   → hStdIn (nécessaire pour toutes les lectures)
   ↓
3. Mémoire
   → RAM simulée (4 Ko)
   → Mémoire virtuelle (10 Mo)
```

### Pourquoi cet ordre ?

**Handles avant mémoire :**
- Les handles sont nécessaires pour afficher les messages d'initialisation
- Si `InitializeMemory` échoue, on peut afficher un message d'erreur (nécessite hStdOut)

**Mémoire avant boucle :**
- Sans mémoire, aucun programme ne peut être chargé
- La boucle menu serait inutile si la mémoire n'est pas initialisée
- Vérification : Si échec, on quitte immédiatement

### Gestion des erreurs

**Si GetStdHandle échoue :**
- Retourne INVALID_HANDLE_VALUE (-1)
- **Actuellement** : Pas de vérification (amélioration possible)
- **Conséquence** : Les E/S échoueraient silencieusement

**Si InitializeMemory échoue :**
- Retourne 0
- **Vérifié** : `test rax, rax` puis `jz exit_program`
- **Conséquence** : Programme quitte proprement

---

## Boucle événementielle {#boucle-evenementielle}

### Type de boucle

**Boucle événementielle** (event loop) :
- Attend une action de l'utilisateur
- Traite l'action
- Retourne à l'attente

### Caractéristiques

1. **Non-bloquante** : Chaque action se termine avant de retourner au menu
2. **Séquentielle** : Une seule action à la fois
3. **Infinie** : Continue jusqu'à l'option 7 (Quitter)

### Alternative : Boucle multi-thread

**Non implémentée** mais possible :
- Thread séparé pour chaque action
- Plus complexe
- Pas nécessaire pour ce projet (actions rapides)

### Avantages de cette approche

1. **Simplicité** : Code facile à comprendre
2. **Débogage** : Facile de tracer l'exécution
3. **Pas de problèmes de synchronisation** : Une seule action à la fois
4. **Suffisant** : Pour les besoins du simulateur

---

## Routage des choix {#routage-choix}

### Méthode utilisée : Comparaisons séquentielles

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

### Avantages

1. **Simplicité** : Code très lisible
2. **Performance** : Sauts conditionnels très rapides
3. **Flexibilité** : Facile d'ajouter/modifier des options
4. **Débogage** : Facile de mettre un breakpoint sur chaque option

### Alternative : Table de sauts (jump table)

**Non utilisée**, mais voici comment cela fonctionnerait :

```assembly
; Exemple conceptuel (non implémenté)
lea rbx, jump_table
mov rax, [rbx + rax*8]      ; Charger adresse depuis table
jmp rax                     ; Sauter

jump_table:
    QWORD option_load_program
    QWORD option_display_memory
    ...
```

**Avantages de la table de sauts :**
- Plus rapide pour beaucoup d'options
- Pas de comparaisons séquentielles

**Pourquoi non utilisée :**
- Plus complexe à maintenir
- Gain négligeable pour 7 options
- Moins lisible

### Gestion des choix invalides

**Si choix < 1 ou > 7 :**
- Aucune condition n'est vraie
- `jmp menu_loop` : Retour au menu
- Le menu se réaffiche

**Résultat :** Choix invalides sont ignorés, l'utilisateur peut réessayer.

---

## Gestion de la terminaison {#terminaison}

### Processus de terminaison

```97:105:main.asm
exit_program:
    ; Libérer la mémoire
    call FreeMemory
    
    ; Terminer le programme
    mov rsp, rbp
    pop rbp
    xor rcx, rcx
    call ExitProcess
```

### Étapes expliquées

**1. Libération de la mémoire**
```assembly
call FreeMemory
```
- Libère la RAM via `VirtualFree(ramBase)`
- Libère la mémoire virtuelle via `VirtualFree(virtualMemBase)`
- **Pourquoi important** : Bonnes pratiques, même si le système libérerait automatiquement

**2. Restauration de la pile**
```assembly
mov rsp, rbp
pop rbp
```
- Restaure RSP à sa valeur initiale
- Restaure RBP (frame pointer)
- **Pourquoi** : Code propre, même si on va quitter

**3. Code de sortie**
```assembly
xor rcx, rcx      ; RCX = 0 (code de succès)
```
- Code de sortie 0 = succès (convention Windows)
- Autres valeurs = erreur (1, 2, etc.)

**4. Terminaison**
```assembly
call ExitProcess
```
- Termine le processus proprement
- Libère toutes les ressources système
- Retourne le code de sortie au système

### Points d'entrée vers exit_program

Il y a **deux** endroits où on peut arriver à `exit_program` :

1. **Option 7 choisie** : Utilisateur veut quitter
2. **Échec d'initialisation** : `jz exit_program` si `InitializeMemory` échoue

### Pourquoi libérer la mémoire explicitement ?

**Le système le ferait automatiquement**, mais :

1. **Bonnes pratiques** : Montre la compréhension de la gestion mémoire
2. **Débogage** : Facilite la détection de fuites mémoire
3. **Clarté** : Code explicite est plus lisible
4. **Conventions** : Respecte les pratiques de programmation système

---

## Résumé

Le module `main.asm` est le **cœur** du projet :

1. **Point d'entrée** : Fonction `main` appelée par le système
2. **Initialisation** : Prépare le système (handles, mémoire)
3. **Orchestration** : Coordonne tous les autres modules
4. **Interface utilisateur** : Gère le menu et la navigation
5. **Cycle de vie** : Démarrage → Exécution → Arrêt propre

**Caractéristiques clés :**
- ✅ Architecture événementielle claire
- ✅ Routage simple et extensible
- ✅ Gestion d'erreurs (vérification initialisation)
- ✅ Terminaison propre (libération mémoire)

**Sans ce module :**
- ❌ Pas de point d'entrée
- ❌ Pas de coordination
- ❌ Pas d'interface utilisateur
- ❌ **Le projet ne fonctionnerait pas**

**Avec ce module :**
- ✅ Programme fonctionnel et utilisable
- ✅ Interface utilisateur intuitive
- ✅ Architecture claire et maintenable

---

**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025  
**Projet :** Simulateur de Gestion de Mémoire Virtuelle - INF22107-MS

