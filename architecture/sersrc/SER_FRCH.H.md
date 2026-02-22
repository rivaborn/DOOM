# File Overview

`SER_FRCH.H` is the French-language string table header for the DOOM serial network driver (`SERSETUP`). It is the active string table in the released source code — `SERSETUP.C`, `PORT.C`, and `DOOMNET.C` all include this file (the English equivalent `SERSTR.H` is commented out in each).

The file defines the same set of string constant macros as `SERSTR.H`, but with French translations of all user-visible messages. It additionally defines several string constants (`STR_PORTLOOK`, `STR_UART8250`, `STR_UART16550`, `STR_CLEARPEND`) that are absent from `SERSTR.H`, making `SER_FRCH.H` the more complete of the two headers — `PORT.C` could not compile against the English header alone.

The French text contains characters encoded in the OEM/DOS code page (Code Page 850 or 437), where accented Latin characters are represented by bytes above 0x7F. These appear as mojibake in UTF-8 environments.

This file contains no code, functions, data structures, or variables — only `#define` string constants.

---

## Global Variables

None.

---

## Functions

None.

---

## Data Structures

None.

---

## Preprocessor String Constants

Each macro expands to a French-language string literal. The table below shows both the French text (as stored) and the intended English meaning.

| Macro | French Value | English Meaning | Used In |
|-------|-------------|-----------------|---------|
| `STR_DROPDTR` | `"Abandon de DTR"` | "Dropping DTR" | `SERSETUP.C` / `Error()` before modem hangup |
| `STR_CLEANEXIT` | `"Sortie normale de SERSETUP"` | "Clean exit from SERSETUP" | `SERSETUP.C` / `Error(NULL)` |
| `STR_ATTEMPT` | `"Tentative de connexion en s\xe9rie, appuyez sur ESC pour annuler."` | "Attempting to connect serially, press ESC to cancel." | `SERSETUP.C` / `Connect()` |
| `STR_NETABORT` | `"Synchronisation de jeu sur r\xe9seau annul\xe9e."` | "Network game synchronization aborted." | `SERSETUP.C` / `Connect()` |
| `STR_DUPLICATE` | `"Cha\xeene id en double. R\xe9essayez ou v\xe9rifiez la cha\xeene d'initialistion du modem."` | "Duplicate id string. Retry or check modem init string." | `SERSETUP.C` / `Connect()` |
| `STR_MODEMCMD` | `"Commande du modem: "` | "Modem command: " | `SERSETUP.C` / `ModemCommand()` |
| `STR_MODEMRESP` | `"R\xe9ponse du modem: "` | "Modem response: " | `SERSETUP.C` / `ModemResponse()` |
| `STR_RESPABORT` | `"R\xe9ponse du modem annul\xe9e."` | "Modem response aborted." | `SERSETUP.C` / `ModemResponse()` |
| `STR_CANTREAD` | `"Lecture de MODEM.CFG impossible"` | "Couldn't read MODEM.CFG" | `SERSETUP.C` / `ReadModemCfg()` |
| `STR_DIALING` | `"Composition du num\xe9ro..."` | "Dialing..." | `SERSETUP.C` / `Dial()` |
| `STR_CONNECT` | `"CONNECTION"` | "CONNECT" (modem response string to wait for) | `SERSETUP.C` / `Dial()`, `Answer()` |
| `STR_WAITRING` | `"Attente d'appel..."` | "Waiting for ring..." | `SERSETUP.C` / `Answer()` |
| `STR_RING` | `"APPEL"` | "RING" (modem response string to wait for) | `SERSETUP.C` / `Answer()` |
| `STR_NORESP` | `"Ce fichier de r\xe9ponse n'existe pas!"` | "No such response file!" | `SERSETUP.C` / `FindResponseFile()` |
| `STR_DOOMSERIAL` | `"GESTIONNAIRE DE LIAISON SERIE DOOM II v1.4"` | "DOOM II SERIAL DEVICE DRIVER v1.4" | `SERSETUP.C` / `main()` |
| `STR_WARNING` | (multi-line, see below) | Warning about no free interrupt vector | `DOOMNET.C` / `LaunchDOOM()` |
| `STR_COMM` | `"Communication avec le vecteur d'interruption 0x%x"` | "Communicating with interrupt vector 0x%x" | `DOOMNET.C` / `LaunchDOOM()` |
| `STR_RETURNED` | `"Retour de DOOM II"` | "Returned from DOOM II" | `DOOMNET.C` / `LaunchDOOM()` |
| `STR_PORTLOOK` | `"Recherche de l'UART sur le port"` | "Looking for UART on port" | `PORT.C` / `GetUart()` |
| `STR_UART8250` | `"UART = 8250"` | "UART = 8250" | `PORT.C` / `InitPort()` |
| `STR_UART16550` | `"UART = 16550"` | "UART = 16550" | `PORT.C` / `InitPort()` |
| `STR_CLEARPEND` | `"Riinitilisation des interruptions en attente.\n"` | "Resetting pending interrupts." (note: French text has typo "Riinitilisation") | `PORT.C` / `InitPort()` |
| `STR_PORTSET` | `"R\xe9glage du port \xe0 %lu baud"` | "Setting port to %lu baud" | `PORT.C` / `InitPort()` |

### Multi-line Warning Macro

```c
#define STR_WARNING \
"Attention: pas de vecteurs d'interruption NULL ou iret trouv\xe9s entre 0x60 et 0x66.\n"\
"Vous pouvez sp\xe9cifier un vecteur avec le param\xe8tre -vector 0x<num\xe9ro>."
```

English: "Warning: no NULL or iret interrupt vectors found between 0x60 and 0x66. You can specify a vector with the -vector 0x\<number\> parameter."

### Notable Differences from `SERSTR.H`

1. **`STR_CONNECT`** is `"CONNECTION"` in French vs `"CONNECT"` in English. This is significant: `ModemResponse` in `SERSETUP.C` waits for this string as the modem connection response. Standard Hayes-compatible modems respond with `"CONNECT"` (or `"CONNECT 9600"`, etc.), so the French version may fail to detect the connection if the modem does not send `"CONNECTION"`. This appears to be a localization bug.

2. **`STR_RING`** is `"APPEL"` in French vs `"RING"` in English. Standard modems send `"RING"`, not `"APPEL"`, so the French `Answer()` function may also fail to detect incoming calls properly unless the modem is configured to send French response codes.

3. **`STR_PORTLOOK`**, **`STR_UART8250`**, **`STR_UART16550`**, **`STR_CLEARPEND`** are only defined in `SER_FRCH.H` and not in `SERSTR.H`, making the English header incomplete.

4. **`STR_CLEARPEND`** contains a typo: `"Riinitilisation"` instead of `"Réinitialisation"`.

---

## Dependencies

None. This is a standalone header file containing only preprocessor macro definitions.

Included by:
- `/mnt/c/Coding/ID/DOOM/sersrc/SERSETUP.C`
- `/mnt/c/Coding/ID/DOOM/sersrc/PORT.C`
- `/mnt/c/Coding/ID/DOOM/sersrc/DOOMNET.C`
