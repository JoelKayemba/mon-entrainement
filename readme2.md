# Justification et Intégration des Composants - Contribution de Joël KAYEMBA

**Projet :** Simulateur de Gestion de Mémoire Virtuelle  
**Cours :** INF22107-MS - Infrastructure de programmation  
**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025

---

## Introduction

Ce document explique **pourquoi** chaque composant que j'ai développé a été créé et **comment** il s'intègre dans le projet global. Il détaille le rôle de chaque fonction, son utilisation par les autres modules, et les bénéfices qu'elle apporte au simulateur de mémoire virtuelle.

---

## Table des Matières

1. [Module utils.asm - Pourquoi cette bibliothèque est essentielle](#utils-essential)
2. [Fonction PrintString - Le fondement de tous les affichages](#printstring-foundation)
3. [Fonction PrintHexQWord - Visualisation des adresses mémoire](#printhex-visualisation)
4. [Fonction PrintDecimal - Affichage des tailles et statistiques](#printdecimal-statistiques)
5. [Fonction GetUserChoice - Interface utilisateur du menu](#getuserchoice-interface)
6. [Fonction ParseHex - Protection d'accès mémoire](#parsehex-protection)
7. [Module main.asm - L'orchestrateur central](#main-orchestrateur)
8. [Script compile.bat - Productivité et fiabilité](#compile-productivite)
9. [Chaîne d'intégration complète](#chaine-integration)

---

## Module utils.asm - Pourquoi cette bibliothèque est essentielle {#utils-essential}

### Le problème à résoudre

Dans un projet en assembleur pur, nous n'avons **pas accès aux fonctions de la bibliothèque C standard** comme `printf`, `scanf`, `atoi`, etc. Chaque opération d'affichage, de lecture ou de conversion doit être implémentée manuellement en utilisant les API Windows directement.

### La solution : Une bibliothèque utilitaire centralisée

J'ai créé le module `utils.asm` comme une **bibliothèque de fonctions réutilisables** qui fournit toutes les opérations d'entrée/sortie et de conversion nécessaires au projet. Cette approche apporte plusieurs avantages cruciaux :

**1. Réduction de la duplication de code**
- Sans cette bibliothèque, chaque module devrait implémenter ses propres fonctions d'affichage
- Résultat : Code dupliqué dans `memory.asm`, `program.asm`, `main.asm`
- Avec `utils.asm` : Une seule implémentation, utilisée partout

**2. Cohérence de l'affichage**
- Tous les modules utilisent les mêmes fonctions, garantissant un format d'affichage uniforme
- Les adresses mémoire sont toujours affichées en hexadécimal de la même manière
- Les nombres décimaux suivent toujours le même format

**3. Maintenance facilitée**
- Si on veut changer le format d'affichage (ex: ajouter des couleurs), on modifie une seule fonction
- Tous les modules bénéficient automatiquement de la modification

**4. Séparation des responsabilités**
- Les modules métier (`memory.asm`, `program.asm`) se concentrent sur leur logique
- Ils délèguent l'affichage à `utils.asm`, suivant le principe de responsabilité unique

### Impact sur le projet

**Sans `utils.asm`**, le projet serait **impossible à utiliser** car :
- Aucun message ne pourrait s'afficher à l'utilisateur
- Aucune entrée utilisateur ne pourrait être lue
- Les adresses mémoire (64 bits) ne pourraient pas être affichées
- Les tailles de programmes ne pourraient pas être visualisées

**Avec `utils.asm`**, le projet devient **complètement fonctionnel** avec une interface utilisateur claire et professionnelle.

---

## Fonction PrintString - Le fondement de tous les affichages {#printstring-foundation}

### Pourquoi cette fonction existe

La fonction `PrintString` (```29:56:utils.asm```) est la **fonction de base la plus utilisée** dans tout le projet. Elle résout un problème fondamental : **comment afficher du texte sur la console en assembleur**.

### Le problème technique

L'API Windows `WriteConsoleA` nécessite :
- Un handle de console (obtenu via `GetStdHandle`)
- Un pointeur vers la chaîne
- La longueur exacte de la chaîne
- Gestion correcte de la convention d'appel Windows x64

Sans `PrintString`, chaque module devrait :
1. Appeler `lstrlenA` pour obtenir la longueur
2. Préparer les paramètres pour `WriteConsoleA`
3. Gérer le shadow space et l'alignement de la pile
4. Gérer les erreurs

**Résultat :** Code répétitif et propice aux erreurs dans chaque module.

### La solution : Une fonction wrapper simple

```assembly
PrintString PROC
    ; ... préparation de la pile ...
    mov rbx, rcx                    ; Sauvegarder le pointeur
    call lstrlenA                   ; Obtenir la longueur
    ; ... appel à WriteConsoleA ...
    ret
PrintString ENDP
```

Cette fonction encapsule toute la complexité et expose une interface simple : **passez un pointeur vers une chaîne, elle s'affiche**.

### Utilisation dans le projet

**1. Affichage du menu principal**

Dans `main.asm`, la fonction `DisplayMenu` utilise `PrintString` pour afficher le menu :

```115:119:main.asm
lea rcx, menuTitle
call PrintString

lea rcx, menuOptions
call PrintString
```

**Résultat :** L'utilisateur voit le menu avec toutes les options disponibles, permettant la navigation dans le simulateur.

**2. Affichage des états de mémoire**

Dans `memory.asm`, la fonction `DisplayMemoryState` utilise `PrintString` pour afficher les informations sur la RAM et la mémoire virtuelle :

```279:295:memory.asm
lea rcx, msgNewLine
call PrintString
lea rcx, msgRAMState
call PrintString

mov rcx, ramBase
call PrintHexQWord
lea rcx, msgNewLine
call PrintString

lea rcx, msgRAMEnd
call PrintString
```

**Résultat :** L'utilisateur peut visualiser l'état de la mémoire à tout moment, ce qui est essentiel pour comprendre le fonctionnement du swapping.

**3. Messages d'erreur et de succès**

Dans `program.asm`, `PrintString` est utilisée pour informer l'utilisateur :

```169:170:program.asm
lea rcx, msgProgLoaded
call PrintString
```

**Résultat :** Feedback immédiat pour chaque action (chargement réussi, erreur, etc.).

### Bénéfices apportés

- **87 utilisations** dans tout le projet
- **Cohérence** : Tous les messages utilisent le même mécanisme
- **Maintenabilité** : Un seul endroit à modifier pour changer le comportement
- **Simplicité** : Les autres modules appellent simplement `PrintString` sans se soucier des détails

---

## Fonction PrintHexQWord - Visualisation des adresses mémoire {#printhex-visualisation}

### Le problème spécifique du projet

Dans un simulateur de mémoire virtuelle, il est **crucial** de pouvoir visualiser les **adresses mémoire en hexadécimal**. Les adresses sont des valeurs 64 bits (QWORD) qui doivent être affichées de manière lisible.

**Exemple d'adresse :** `0x0000020E61770900` (une adresse réelle de la RAM simulée)

### Pourquoi l'hexadécimal ?

1. **Format standard** : Les adresses mémoire sont traditionnellement affichées en hexadécimal
2. **Lisibilité** : `0x20E61770900` est plus lisible que `225179981376256` en décimal
3. **Correspondance binaire** : Chaque chiffre hex représente exactement 4 bits, facilitant le débogage
4. **Convention système** : Tous les outils système (debuggers, etc.) affichent les adresses en hex

### La solution : Conversion manuelle optimisée

La fonction `PrintHexQWord` (```62:103:utils.asm```) convertit un nombre 64 bits en représentation hexadécimale ASCII.

**Algorithme utilisé :**
1. Extraction des nibbles (4 bits) de droite à gauche
2. Conversion de chaque nibble en caractère ASCII ('0'-'9' ou 'A'-'F')
3. Construction de la chaîne de droite à gauche
4. Affichage via `PrintString`

### Utilisation dans le projet

**1. Affichage des adresses de la RAM**

Dans `memory.asm`, `DisplayMemoryState` utilise `PrintHexQWord` pour montrer où commence et finit la RAM :

```284:293:memory.asm
mov rcx, ramBase
call PrintHexQWord
lea rcx, msgNewLine
call PrintString

lea rcx, msgRAMEnd
call PrintString
mov rcx, ramBase
add rcx, RAM_SIZE
call PrintHexQWord
```

**Résultat :** L'utilisateur voit :
```
Adresse debut: 0x0000020E61770900
Adresse fin: 0x0000020E61771900
```

Cela permet de comprendre **où se trouve physiquement la RAM simulée** dans l'espace mémoire du système.

**2. Affichage des adresses des programmes**

Dans `program.asm`, `ListAllPrograms` affiche l'adresse de début de chaque programme :

```805:809:program.asm
lea rcx, msgProgAddr
call PrintString
mov rcx, [rsi].ProgramInfo.startAddr
call PrintHexQWord
```

**Résultat :** L'utilisateur peut voir où chaque programme est chargé en mémoire, ce qui est essentiel pour :
- Comprendre la localisation des programmes
- Vérifier si un programme est en RAM ou en mémoire virtuelle
- Déboguer les problèmes d'allocation

**3. Vérification d'accès mémoire**

Dans `program.asm`, `RequestMemoryAccess` affiche l'adresse à laquelle l'accès a été autorisé :

```718:723:program.asm
lea rcx, msgAccessOK
call PrintString
mov rcx, r12
call PrintHexQWord
lea rcx, msgNewLine
call PrintString
```

**Résultat :** Quand l'utilisateur demande l'accès à une adresse mémoire, le système confirme avec l'adresse exacte en hexadécimal, permettant de vérifier que la protection d'accès fonctionne correctement.

### Bénéfices apportés

- **6 utilisations** dans des contextes critiques
- **Visualisation claire** : Les adresses sont immédiatement reconnaissables
- **Débogage facilité** : Les adresses hexadécimales permettent de tracer les problèmes
- **Conformité** : Suit les conventions d'affichage des outils système

---

## Fonction PrintDecimal - Affichage des tailles et statistiques {#printdecimal-statistiques}

### Le besoin métier du simulateur

Le simulateur doit afficher de nombreuses valeurs numériques en **décimal** pour être compréhensible par l'utilisateur :
- Tailles de programmes (ex: 512 octets, 1024 octets)
- Utilisation de la RAM (ex: 1536 octets utilisés sur 4096)
- Pourcentages d'utilisation (ex: 37%)
- Numéros de programmes dans les listes

### Pourquoi pas l'hexadécimal pour tout ?

Bien que l'hexadécimal soit idéal pour les adresses, le **décimal est plus naturel** pour :
- Les tailles : "1024 octets" est plus compréhensible que "0x400 octets"
- Les pourcentages : "75%" est standard, pas "0x4B%"
- Les compteurs : "Programme 3" est plus clair que "Programme 0x3"

### La solution : Conversion décimale par division répétée

La fonction `PrintDecimal` (```238:271:utils.asm```) convertit un nombre en représentation décimale ASCII.

**Algorithme :**
1. Division répétée par 10
2. Extraction du reste (chiffre de droite)
3. Conversion en ASCII ('0' + reste)
4. Construction de droite à gauche
5. Affichage via `PrintString`

### Utilisation dans le projet

**1. Affichage de l'utilisation de la RAM**

Dans `memory.asm`, `DisplayMemoryState` calcule et affiche combien d'octets sont utilisés :

```326:338:memory.asm
lea rcx, msgRAMUsed
call PrintString
mov rcx, r12
call PrintDecimal
lea rcx, msgBytes
call PrintString

lea rcx, msgRAMTotal
call PrintString
mov rcx, RAM_SIZE
call PrintDecimal
lea rcx, msgBytes
call PrintString
```

**Résultat :** L'utilisateur voit :
```
Utilise: 1536 octets / 4096 octets (37%)
```

Cela permet de **comprendre visuellement** l'état de la RAM et de voir quand le swapping va se déclencher (quand la RAM est pleine).

**2. Affichage des tailles de programmes**

Dans `program.asm`, `ListAllPrograms` affiche la taille de chaque programme :

```797:802:program.asm
lea rcx, msgProgSize
call PrintString
mov rcx, [rsi].ProgramInfo.progSize
call PrintDecimal
lea rcx, msgNewLine
call PrintString
```

**Résultat :** L'utilisateur peut voir la taille de chaque programme, ce qui permet de :
- Comprendre pourquoi certains programmes nécessitent du swapping
- Vérifier que les programmes ont été chargés correctement
- Planifier l'allocation mémoire

**3. Affichage des pourcentages**

Dans `memory.asm`, le pourcentage d'utilisation est calculé et affiché :

```347:352:memory.asm
lea rcx, msgRAMPercent
call PrintString
mov rcx, rax
call PrintDecimal
lea rcx, msgPercentSign
call PrintString
```

**Résultat :** Un pourcentage facilement compréhensible (ex: "37%") plutôt qu'une fraction.

### Bénéfices apportés

- **6 utilisations** pour des informations critiques
- **Lisibilité** : Les nombres décimaux sont immédiatement compréhensibles
- **Expérience utilisateur** : Interface plus professionnelle et intuitive
- **Débogage** : Les tailles en décimal permettent de vérifier rapidement les calculs

---

## Fonction GetUserChoice - Interface utilisateur du menu {#getuserchoice-interface}

### Le besoin d'interaction utilisateur

Le simulateur doit permettre à l'utilisateur de **choisir parmi 7 options** du menu principal. Sans cette fonction, le programme serait statique et inutilisable.

### Pourquoi une fonction dédiée ?

On pourrait utiliser `GetUserInput` (fonction générique) mais `GetUserChoice` est **spécialisée** pour le menu :

**Avantages de la spécialisation :**
1. **Simplicité** : Retourne directement un nombre (1-7), pas besoin de parsing
2. **Performance** : Lit seulement le premier caractère, plus rapide
3. **Clarté** : Le nom de la fonction indique clairement son usage
4. **Robustesse** : Moins de code = moins de possibilités d'erreur

### La solution : Lecture et conversion simple

La fonction `GetUserChoice` (```142:163:utils.asm```) :
1. Lit l'entrée utilisateur via `ReadConsoleA`
2. Prend le premier caractère
3. Le convertit de ASCII vers nombre ('0'→0, '1'→1, etc.)
4. Retourne le résultat

### Utilisation dans le projet

**Dans main.asm, la boucle événementielle principale :**

```52:71:main.asm
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

**Résultat :** Cette fonction est **appelée à chaque itération de la boucle principale**. Elle permet :
- La navigation dans le menu
- L'exécution des différentes fonctionnalités
- La sortie du programme (option 7)

**Sans cette fonction**, le simulateur serait **impossible à utiliser** car il n'y aurait aucun moyen d'interagir avec lui.

### Intégration avec les autres composants

1. **Appelée par `main.asm`** : Cœur de l'interface utilisateur
2. **Utilise `inputBuffer`** : Buffer défini dans `data.asm`
3. **Utilise `hStdIn`** : Handle initialisé dans `main.asm`
4. **Utilise `ReadConsoleA`** : API Windows pour la lecture

**Chaîne d'appel :**
```
main (obtient hStdIn) 
  → GetUserChoice (lit depuis hStdIn)
    → ReadConsoleA (API Windows)
```

### Bénéfices apportés

- **Interface utilisateur fonctionnelle** : Sans elle, pas de navigation
- **Simplicité** : Les autres modules n'ont pas à gérer la lecture du menu
- **Centralisation** : Toute la logique de lecture du menu est au même endroit
- **Extensibilité** : Facile d'ajouter de nouvelles options (jusqu'à 9 avec un seul chiffre)

---

## Fonction ParseHex - Protection d'accès mémoire {#parsehex-protection}

### Le besoin de sécurité du simulateur

L'une des fonctionnalités clés du projet est la **protection d'accès mémoire** (Étape 7 du cahier des charges). L'utilisateur doit pouvoir demander l'accès à une adresse mémoire, et le système doit vérifier si cette adresse appartient au programme actif.

### Le problème technique

Les adresses mémoire sont des nombres 64 bits, mais l'utilisateur les entre **sous forme de chaîne hexadécimale** (ex: "20E61770900"). Il faut donc :
1. Lire la chaîne entrée par l'utilisateur
2. La convertir en nombre 64 bits
3. Vérifier si cette adresse est dans les limites du programme

### Pourquoi ParseHex est nécessaire

Sans `ParseHex`, il faudrait implémenter la conversion hexadécimale dans `RequestMemoryAccess`, ce qui :
- Dupliquerait du code (mauvaise pratique)
- Complexifierait la fonction de vérification d'accès
- Rendreait le code moins maintenable

### La solution : Parsing robuste

La fonction `ParseHex` (```170:232:utils.asm```) :
1. Parse caractère par caractère
2. Supporte les majuscules ET minuscules (A-F et a-f)
3. S'arrête sur caractères invalides (espaces, retours à la ligne)
4. Construit le nombre de gauche à droite
5. Gère les nombres jusqu'à 64 bits

### Utilisation dans le projet

**Dans program.asm, RequestMemoryAccess :**

```690:706:program.asm
; Demander l'adresse
lea rcx, msgEnterAddr
call PrintString

; Lire l'adresse en hexadécimal
mov rcx, hStdIn
lea rdx, inputBuffer
mov r8d, 256
lea r9, bytesRead
xor rax, rax
mov [rsp+20h], rax
call ReadConsoleA

; Convertir hex en nombre
lea rsi, inputBuffer
call ParseHex
mov r12, rax                    ; R12 = adresse demandée
```

**Ensuite, la vérification des limites :**

```708:715:program.asm
; Vérifier les limites
mov rdx, [rbx].ProgramInfo.startAddr
cmp r12, rdx
jl access_denied

mov rdx, [rbx].ProgramInfo.endAddr
cmp r12, rdx
jge access_denied
```

**Résultat :** 
- L'utilisateur entre une adresse en hexadécimal (ex: "20E61770900")
- `ParseHex` la convertit en nombre (ex: 225179981376256)
- Le système vérifie si cette adresse est dans les limites du programme
- Si oui : "Acces AUTORISE"
- Si non : "Acces REFUSE! Adresse hors limites."

### Bénéfices apportés

- **Fonctionnalité de protection** : Sans elle, la vérification d'accès serait impossible
- **Flexibilité** : Support des majuscules et minuscules (meilleure expérience utilisateur)
- **Robustesse** : Gère automatiquement les espaces et retours à la ligne
- **Réutilisabilité** : Pourrait être utilisée ailleurs si besoin

---

## Module main.asm - L'orchestrateur central {#main-orchestrateur}

### Pourquoi main.asm est crucial

Le module `main.asm` est le **point d'entrée** du programme. Sans lui, le programme ne pourrait même pas démarrer. Mais son rôle va bien au-delà : il **orchestre tous les autres modules** pour créer une application cohérente.

### Le problème architectural

Un projet modulaire nécessite un **coordinateur** qui :
1. Initialise le système
2. Affiche l'interface utilisateur
3. Lit les choix de l'utilisateur
4. Route vers les bonnes fonctions
5. Gère le cycle de vie du programme

Sans `main.asm`, les modules existeraient mais seraient **disconnectés** et **inutilisables**.

### La solution : Une boucle événementielle claire

**1. Initialisation du système**

```37:49:main.asm
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
```

**Pourquoi c'est important :**
- Les handles console sont **essentiels** pour toutes les fonctions d'E/S de `utils.asm`
- Sans `hStdOut`, `PrintString` ne pourrait rien afficher
- Sans `hStdIn`, `GetUserChoice` ne pourrait rien lire
- L'initialisation de la mémoire est **prérequis** à toute opération

**2. Boucle événementielle**

```52:71:main.asm
menu_loop:
    call DisplayMenu
    call GetUserChoice
    
    ; Traiter le choix (rax contient le choix)
    cmp rax, 1
    je option_load_program
    cmp rax, 2
    je option_display_memory
    ...
    jmp menu_loop
```

**Pourquoi cette architecture :**

- **Menu affiché en continu** : L'utilisateur voit toujours les options disponibles
- **Choix utilisateur** : Permet la navigation
- **Routage clair** : Chaque choix déclenche la bonne action
- **Boucle infinie** : Le programme continue jusqu'à l'option 7 (Quitter)

**3. Routage vers les modules**

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
...
```

**Pourquoi ce pattern :**
- **Isolation** : Chaque option est dans son propre label
- **Maintenabilité** : Facile d'ajouter/modifier des options
- **Retour au menu** : Après chaque action, retour au menu principal

### Intégration avec les autres modules

**main.asm utilise :**
- `utils.asm` : `PrintString`, `GetUserChoice` (pour le menu)
- `memory.asm` : `InitializeMemory`, `DisplayMemoryState`, `FreeMemory`
- `program.asm` : `LoadProgram`, `CloseProgram`, `UseProgram`, `RequestMemoryAccess`, `ListAllPrograms`

**Chaîne d'exécution complète :**

```
Démarrage
  ↓
main.asm (obtient handles)
  ↓
main.asm (initialise mémoire via memory.asm)
  ↓
Boucle :
  main.asm (affiche menu via utils.asm)
  main.asm (lit choix via utils.asm)
  main.asm (route vers program.asm ou memory.asm)
  program.asm/memory.asm (utilise utils.asm pour afficher)
  ↓ (retour au menu)
```

### Bénéfices apportés

- **Point d'entrée unique** : Le programme a un démarrage clair
- **Coordination** : Tous les modules sont orchestrés de manière cohérente
- **Interface utilisateur** : Le menu permet la navigation
- **Cycle de vie** : Gestion propre de l'initialisation et de la terminaison
- **Extensibilité** : Facile d'ajouter de nouvelles fonctionnalités

---

## Script compile.bat - Productivité et fiabilité {#compile-productivite}

### Le problème de compilation manuelle

Le projet contient **5 fichiers assembleur** qui doivent être compilés dans un ordre spécifique, puis liés ensemble avec les bonnes bibliothèques. La compilation manuelle serait :

```batch
ml64 /c /Zi data.asm
ml64 /c /Zi utils.asm
ml64 /c /Zi memory.asm
ml64 /c /Zi program.asm
ml64 /c /Zi main.asm
link /SUBSYSTEM:CONSOLE /ENTRY:main /DEBUG data.obj utils.obj memory.obj program.obj main.obj kernel32.lib user32.lib /OUT:memory_simulator.exe
```

**Problèmes :**
- **6 commandes à taper** à chaque compilation
- **Risque d'erreur de frappe** (mauvais nom de fichier, oubli de bibliothèque)
- **Temps perdu** à chaque modification
- **Pas de gestion d'erreur** automatique

### La solution : Automatisation complète

Le script `compile.bat` automatise tout le processus :

```7:25:compile.bat
echo Compilation de data.asm...
ml64 /c /Zi data.asm
if %errorlevel% neq 0 goto error

echo Compilation de utils.asm...
ml64 /c /Zi utils.asm
if %errorlevel% neq 0 goto error
...
```

**Avantages :**

1. **Une seule commande** : `compile.bat` au lieu de 6 commandes
2. **Gestion d'erreur** : Arrêt immédiat si une compilation échoue
3. **Messages clairs** : L'utilisateur voit ce qui se passe
4. **Fiabilité** : Impossible d'oublier un fichier ou une bibliothèque
5. **Reproductibilité** : Toujours la même compilation, peu importe qui l'exécute

### Impact sur le développement

**Sans compile.bat :**
- Développement ralenti (compilation manuelle longue)
- Risque d'erreurs (mauvaise compilation)
- Frustration (répétition de commandes)
- Collaboration difficile (chacun doit connaître les commandes)

**Avec compile.bat :**
- **Compilation en une commande** : Gain de temps considérable
- **Fiabilité** : Moins d'erreurs de compilation
- **Accessibilité** : Même un développeur novice peut compiler
- **Productivité** : Plus de temps pour développer, moins pour compiler

### Intégration dans le workflow

Le script fait partie intégrante du processus de développement :

1. **Modification du code** dans un des fichiers .asm
2. **Exécution de compile.bat** pour recompiler
3. **Test du programme** avec memory_simulator.exe
4. **Retour à l'étape 1** si besoin

Ce cycle itératif est **beaucoup plus rapide** avec le script.

---

## Chaîne d'intégration complète {#chaine-integration}

### Exemple complet : Chargement d'un programme

Voici comment tous mes composants s'intègrent pour réaliser une fonctionnalité complète du projet : **charger un programme**.

**Étape 1 : L'utilisateur démarre le programme**

```
main.asm (main)
  → Obtient hStdIn et hStdOut (GetStdHandle)
  → Initialise la mémoire (InitializeMemory)
  → Entre dans la boucle menu_loop
```

**Résultat :** Le système est prêt, les handles sont disponibles pour `utils.asm`.

**Étape 2 : Affichage du menu**

```
main.asm (DisplayMenu)
  → Appelle PrintString (utils.asm) avec menuTitle
  → Appelle PrintString (utils.asm) avec menuOptions
```

**Résultat :** L'utilisateur voit le menu avec l'option "1. Charger un programme".

**Étape 3 : L'utilisateur choisit l'option 1**

```
main.asm (menu_loop)
  → Appelle GetUserChoice (utils.asm)
    → GetUserChoice lit depuis hStdIn
    → Convertit '1' en nombre 1
    → Retourne 1
  → Compare rax avec 1
  → Saute vers option_load_program
```

**Résultat :** Le choix de l'utilisateur est capturé et routé.

**Étape 4 : Exécution du chargement**

```
main.asm (option_load_program)
  → Appelle LoadProgram (program.asm)
    → LoadProgram appelle PrintString (utils.asm) pour "Entrez le nom du fichier..."
    → LoadProgram lit le nom via ReadConsoleA
    → LoadProgram charge le fichier
    → LoadProgram appelle PrintString (utils.asm) pour "Programme charge avec succes!"
```

**Résultat :** Le programme est chargé et l'utilisateur est informé.

**Étape 5 : Retour au menu**

```
main.asm (option_load_program)
  → jmp menu_loop
  → Retour à l'étape 2 (affichage du menu)
```

**Résultat :** L'utilisateur peut continuer à utiliser le simulateur.

### Flux de données entre les modules

```
┌─────────────┐
│  main.asm   │ ← Point d'entrée, orchestration
└──────┬──────┘
       │ utilise
       ├──────────────────┐
       │                  │
       ▼                  ▼
┌─────────────┐    ┌──────────────┐
│ utils.asm   │    │ program.asm  │
│             │    │              │
│ PrintString │◄───┤ LoadProgram  │
│ GetUserChoice│   │              │
│ PrintHex... │    │              │
└─────────────┘    └──────────────┘
       ▲                  │
       │                  │ utilise
       └──────────────────┘
                  │
                  ▼
         ┌──────────────┐
         │ memory.asm   │
         │              │
         │ DisplayMemory│
         │ Initialize...│
         └──────────────┘
```

### Pourquoi cette architecture fonctionne

1. **Séparation des responsabilités**
   - `utils.asm` : E/S et conversions
   - `main.asm` : Orchestration
   - `program.asm` : Logique métier programmes
   - `memory.asm` : Logique métier mémoire

2. **Réutilisabilité**
   - `PrintString` est utilisée 87 fois sans duplication
   - Chaque fonction a un rôle clair et réutilisable

3. **Maintenabilité**
   - Modifier l'affichage ? Changez `PrintString` une fois
   - Ajouter une option ? Ajoutez un label dans `main.asm`

4. **Testabilité**
   - Chaque module peut être testé indépendamment
   - Les fonctions utilitaires sont faciles à tester

---

## Conclusion : L'impact de ma contribution

### Résumé des composants développés

1. **utils.asm** : 6 fonctions utilitaires, **fondation du projet**
2. **main.asm** : Point d'entrée et orchestration, **cœur du système**
3. **compile.bat** : Script de compilation, **productivité du développement**

### Impact quantitatif

- **87 utilisations** de `PrintString` dans tout le projet
- **6 utilisations** de `PrintHexQWord` pour les adresses critiques
- **6 utilisations** de `PrintDecimal` pour les statistiques
- **1 utilisation** de `GetUserChoice` à chaque itération du menu
- **1 utilisation** de `ParseHex` pour la protection d'accès

### Impact qualitatif

**Sans ma contribution, le projet serait :**
- ❌ **Inutilisable** : Pas d'affichage, pas d'entrée utilisateur
- ❌ **Incompréhensible** : Pas de visualisation des adresses mémoire
- ❌ **Difficile à développer** : Compilation manuelle fastidieuse

**Avec ma contribution, le projet est :**
- ✅ **Fonctionnel** : Interface utilisateur complète et professionnelle
- ✅ **Compréhensible** : Toutes les informations sont affichées clairement
- ✅ **Maintenable** : Code modulaire et réutilisable
- ✅ **Productif** : Compilation en une commande

### Rôle dans les objectifs pédagogiques

Mon travail démontre la compréhension de :
- **Gestion bas niveau** : Manipulation directe des API Windows
- **Architecture modulaire** : Séparation des responsabilités
- **Conventions système** : Windows x64 calling convention
- **Algorithmes** : Conversions numériques, parsing
- **Outils de développement** : Scripts d'automatisation

Ces compétences sont **essentielles** pour comprendre comment fonctionnent les systèmes d'exploitation et la gestion mémoire, qui sont les objectifs principaux du projet.

---

**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025  
**Projet :** Simulateur de Gestion de Mémoire Virtuelle - INF22107-MS

