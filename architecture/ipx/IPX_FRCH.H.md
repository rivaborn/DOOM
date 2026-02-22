# File Overview

**File:** `ipx/IPX_FRCH.H`
**Target Platform:** MS-DOS (any; purely preprocessor definitions)

`IPX_FRCH.H` is the French-language string table for the DOOM IPX network driver. It defines the same set of preprocessor string macros as `IPXSTR.H` (the English version), but with French translations.

This file is the active string table in the shipped source: both `DOOMNET.C` and `IPXSETUP.C` have the `#include "ipxstr.h"` line commented out and `#include "ipx_frch.h"` active. The inclusion of this French version in the official release (rather than the English `IPXSTR.H`) suggests this particular snapshot was prepared for a French release or localization effort.

The strings contain characters that appear as question marks or accented characters — these are encoded in an 8-bit DOS code page (likely CP850 or CP437), which is why some characters may display incorrectly in modern UTF-8 environments.

---

## Global Variables

None. This is a header of preprocessor `#define` macros only.

---

## Functions

None.

---

## Data Structures

None.

---

## Preprocessor Definitions

All string macros defined in this file (with English equivalents for reference):

| Macro | French Text | English Equivalent |
|-------|-------------|-------------------|
| `STR_NETABORT` | `"Synchronisation du jeu sur réseau annulée."` | "Network game synchronization aborted." |
| `STR_UNKNOWN` | `"Paquet de jeu inconnu durant la configuration"` | "Got an unknown game packet during setup" |
| `STR_FOUND` | `"Noeud détecté!"` | "Found a node!" |
| `STR_LOOKING` | `"Recherche d'un noeud"` | "Looking for a node" |
| `STR_MORETHAN` | `"Plus de %i joueurs spécifiés!"` | "More than %i players specified!" |
| `STR_NONESPEC` | `"Pas de joueurs spécifiés pour le jeu!"` | "No players specified for game!" |
| `STR_CONSOLEIS` | `"Console: joueur %i sur %i"` | "Console is player %i of %i" |
| `STR_NORESP` | `"Ce fichier de réponse n'existe pas!"` | "No such response file!" |
| `STR_FOUNDRESP` | `"Fichier de réponse trouvé"` | "Found response file" |
| `STR_DOOMNETDRV` | `"GESTIONNAIRE DE RESEAU DOOM II"` | "DOOM II NETWORK DEVICE DRIVER" |
| `STR_VECTSPEC` | `"Le vecteur spécifié (0x%02x) était déjà connecté."` | "The specified vector (0x%02x) was already hooked." |
| `STR_NONULL` | Multi-line: "Attention: pas de vecteurs d'interruption NULL ou iret trouvés entre 0x60 et 0x66.\nVous pouvez spécifier un vecteur avec le paramètre -vector 0x<numéro>." | Warning about no free interrupt vectors |
| `STR_COMMVECT` | `"Communication avec le vecteur d'interruption 0x%x"` | "Communicating with interrupt vector 0x%x" |
| `STR_USEALT` | `"Utilisation du port alternatif %i pour le réseau"` | "Using alternate port %i for network" |
| `STR_RETURNED` | `"Retour de DOOM II"` | "Returned from DOOM II" |
| `STR_ATTEMPT` | Multi-line: `"Tentatative de recherche de tous les joueurs pour le jeu en riseau \`%i jouers\nAppuyez sur ECHAP pour quitter.\n"` | "Attempting to find all players for %i player net play. Press ESC to exit.\n" |

**Note:** The `STR_ATTEMPT` macro contains what appears to be a typo in the French text ("Tentatative" instead of "Tentative", "riseau" instead of "réseau", "jouers" instead of "joueurs") and uses a backtick character before `%i`. This suggests the French strings were a quick localization rather than a fully proofread translation.

---

## Dependencies

| File | Reason |
|------|--------|
| None | This file has no `#include` directives |

**Included by:**
- `ipx/DOOMNET.C` (active include, replacing `ipxstr.h`)
- `ipx/IPXSETUP.C` (active include, replacing `ipxstr.h`)
