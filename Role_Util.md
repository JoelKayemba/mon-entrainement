# Rôle des Fonctions Utilitaires - Module utils.asm

**Projet :** Simulateur de Gestion de Mémoire Virtuelle  
**Fichier :** `utils.asm`  
**Auteur :** Joël KAYEMBA

---

## Vue d'ensemble

Le module `utils.asm` contient **6 fonctions essentielles** utilisées par tous les autres modules du projet. Ces fonctions fournissent les opérations de base nécessaires pour l'affichage, la lecture et les conversions numériques.

---

## 1. PrintString

### À quoi ça sert ?

Affiche une chaîne de caractères sur la console.

### Pourquoi c'est nécessaire ?

**Sans cette fonction :**
- ❌ Aucun message ne pourrait s'afficher à l'utilisateur
- ❌ Le menu serait invisible
- ❌ Les messages d'erreur et de succès ne seraient pas visibles
- ❌ L'interface utilisateur serait complètement muette

**Avec cette fonction :**
- ✅ Tous les messages s'affichent correctement
- ✅ Le menu est visible
- ✅ L'utilisateur reçoit des retours sur ses actions

### Rôle dans le programme

- **Utilisée 87 fois** dans tout le projet
- Fonction **la plus utilisée** du projet
- **Fondation** de toute l'interface utilisateur
- Utilisée par `main.asm`, `memory.asm`, `program.asm`

### Exemples d'utilisation

- Affichage du menu principal
- Messages de chargement de programmes
- Affichage des états de mémoire
- Messages d'erreur et de succès

---

## 2. PrintHexQWord

### À quoi ça sert ?

Affiche un nombre 64 bits en hexadécimal (ex: `0x0000020E61770900`).

### Pourquoi c'est nécessaire ?

**Dans un simulateur de mémoire, il faut afficher les adresses mémoire**, et les adresses sont traditionnellement affichées en hexadécimal.

**Sans cette fonction :**
- ❌ Les adresses mémoire seraient incompréhensibles (nombres décimaux énormes)
- ❌ Impossible de voir où sont chargés les programmes
- ❌ Difficile de déboguer les problèmes d'allocation

**Avec cette fonction :**
- ✅ Les adresses sont lisibles et reconnaissables
- ✅ On peut voir précisément où chaque programme est chargé
- ✅ Format standard utilisé par tous les outils système

### Rôle dans le programme

- **Utilisée 6 fois** pour des affichages critiques :
  - Adresses de début/fin de la RAM
  - Adresses de début/fin de la mémoire virtuelle
  - Adresses des programmes
  - Adresses lors de la vérification d'accès mémoire

### Exemples d'utilisation

- `DisplayMemoryState` : Affiche les adresses de la RAM et mémoire virtuelle
- `ListAllPrograms` : Affiche l'adresse de chaque programme
- `RequestMemoryAccess` : Affiche l'adresse à laquelle l'accès a été autorisé

---

## 3. PrintDecimal

### À quoi ça sert ?

Affiche un nombre 64 bits en décimal (ex: `512`, `1536`, `37`).

### Pourquoi c'est nécessaire ?

Pour les **tailles, statistiques et compteurs**, le décimal est plus naturel et compréhensible que l'hexadécimal.

**Sans cette fonction :**
- ❌ Les tailles de programmes seraient en hexadécimal (ex: `0x200` au lieu de `512`)
- ❌ Les pourcentages seraient incompréhensibles
- ❌ L'interface serait moins intuitive

**Avec cette fonction :**
- ✅ Les tailles sont facilement lisibles ("512 octets" plutôt que "0x200 octets")
- ✅ Les pourcentages sont clairs ("37%" au lieu de "0x25%")
- ✅ L'interface est plus professionnelle

### Rôle dans le programme

- **Utilisée 6 fois** pour des informations importantes :
  - Tailles de programmes
  - Utilisation de la RAM (octets utilisés)
  - Pourcentages d'utilisation
  - Compteurs et numéros

### Exemples d'utilisation

- `DisplayMemoryState` : Affiche "Utilise: 1536 octets / 4096 octets (37%)"
- `ListAllPrograms` : Affiche "Taille: 512 octets"

---

## 4. GetUserInput

### À quoi ça sert ?

Lit une ligne de texte depuis la console (fonction générique).

### Pourquoi c'est nécessaire ?

Pour lire **n'importe quelle entrée utilisateur** (noms de fichiers, adresses, etc.).

**Sans cette fonction :**
- ❌ Impossible de lire les entrées utilisateur
- ❌ Pas de saisie de noms de fichiers
- ❌ Pas d'interaction avec l'utilisateur

**Avec cette fonction :**
- ✅ On peut lire les entrées utilisateur
- ✅ On peut demander des noms de fichiers
- ✅ L'interaction est possible

### Rôle dans le programme

- Fonction **générique** pour toutes les lectures
- Utilisée indirectement via d'autres fonctions
- Fondation pour les fonctions spécialisées

### Note

Cette fonction est définie mais peu utilisée directement. La plupart des lectures utilisent `ReadConsoleA` directement ou `GetUserChoice` pour le menu.

---

## 5. GetUserChoice

### À quoi ça sert ?

Lit le choix de l'utilisateur dans le menu (1-7) et le retourne comme nombre.

### Pourquoi c'est nécessaire ?

**C'est le cœur de la navigation dans le programme.** Sans elle, l'utilisateur ne pourrait pas choisir les options du menu.

**Sans cette fonction :**
- ❌ Aucune navigation possible
- ❌ Le menu serait inutilisable
- ❌ Impossible de charger, fermer ou utiliser des programmes
- ❌ Le programme serait complètement statique

**Avec cette fonction :**
- ✅ L'utilisateur peut naviguer dans le menu
- ✅ Chaque choix déclenche la bonne action
- ✅ L'interface est interactive

### Rôle dans le programme

- **Appelée à chaque itération** de la boucle menu dans `main.asm`
- **Fonction critique** : Sans elle, pas de navigation
- Convertis simplement le caractère ASCII ('1'-'7') en nombre (1-7)

### Exemples d'utilisation

- Dans `main.asm`, boucle principale : Après l'affichage du menu, cette fonction lit le choix
- Le choix est ensuite routé vers la bonne fonction (LoadProgram, CloseProgram, etc.)

---

## 6. ParseHex

### À quoi ça sert ?

Convertit une chaîne hexadécimale (ex: "1A2B") en nombre 64 bits.

### Pourquoi c'est nécessaire ?

Pour la **fonctionnalité de protection d'accès mémoire**. L'utilisateur entre une adresse en hexadécimal, et le système doit la convertir en nombre pour vérifier les limites.

**Sans cette fonction :**
- ❌ Impossible de tester la protection d'accès
- ❌ L'utilisateur ne pourrait pas entrer d'adresses mémoire
- ❌ La fonctionnalité de vérification d'accès serait inutilisable

**Avec cette fonction :**
- ✅ L'utilisateur peut entrer des adresses en hexadécimal
- ✅ Le système peut vérifier si l'adresse est dans les limites du programme
- ✅ La protection d'accès fonctionne correctement

### Rôle dans le programme

- **Utilisée dans `RequestMemoryAccess`** (program.asm)
- Permet la conversion des adresses entrées par l'utilisateur
- Supporte les majuscules et minuscules (A-F et a-f)

### Exemples d'utilisation

- L'utilisateur choisit l'option 5 (Demander accès mémoire)
- Le système demande : "Entrez l'adresse memoire (hex): "
- L'utilisateur entre : "20E61770900"
- `ParseHex` convertit "20E61770900" en nombre
- Le système vérifie si cette adresse est dans les limites du programme actif

---

## Résumé : Pourquoi utils.asm est essentiel

### Impact global

**Sans utils.asm :**
- ❌ **Aucun affichage** : Pas de messages, pas de menu
- ❌ **Aucune entrée** : Pas de lecture utilisateur
- ❌ **Pas de conversions** : Pas d'affichage d'adresses ou de nombres
- ❌ **Le projet serait inutilisable**

**Avec utils.asm :**
- ✅ **Interface complète** : Menu, messages, affichages
- ✅ **Interactions** : Lecture des choix et entrées
- ✅ **Visualisation** : Adresses et nombres affichés correctement
- ✅ **Projet fonctionnel**

### Statistiques d'utilisation

| Fonction | Utilisations | Importance |
|----------|--------------|------------|
| PrintString | 87 fois | Critique |
| PrintHexQWord | 6 fois | Critique (adresses) |
| PrintDecimal | 6 fois | Important (statistiques) |
| GetUserChoice | Chaque itération menu | Critique (navigation) |
| ParseHex | 1 fois | Nécessaire (protection) |
| GetUserInput | Rare | Générique |

### Conclusion

Le module `utils.asm` est la **fondation** du projet. Toutes les autres fonctionnalités dépendent de ces fonctions utilitaires. Sans elles, le simulateur serait un programme vide et inutilisable.

**Rôle principal :** Fournir la **couche d'abstraction** entre les autres modules et les API Windows, permettant un code propre, réutilisable et maintenable.

---

**Auteur :** Joël KAYEMBA  
**Date :** Décembre 2025  
**Projet :** Simulateur de Gestion de Mémoire Virtuelle - INF22107-MS

