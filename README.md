# 🔌 KiCad MCP Server + Claude Code CLI — Guida Configurazione Windows

> Scritto da **Caricek** ([@BitMakerMan](https://github.com/BitMakerMan))  
> Collega Claude Code al tuo KiCad e parla con i tuoi PCB in linguaggio naturale.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![KiCad](https://img.shields.io/badge/KiCad-9.0%20%7C%2010.0-blue)](https://www.kicad.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey)](https://github.com/BitMakerMan/KicadMCPGuidaClaudeCode)

---

## 📑 Indice

- [Panoramica](#-panoramica)
- [Prerequisiti](#-prerequisiti)
- [Step 1 — Clonare e compilare il server](#-step-1--clonare-e-compilare-il-server)
- [Step 2 — Trovare i percorsi di KiCad](#-step-2--trovare-i-percorsi-di-kicad)
- [Step 3 — Creare lo script di avvio `start.cmd`](#-step-3--creare-lo-script-di-avvio-startcmd)
- [Step 4 — Collegare il server a Claude Code](#-step-4--collegare-il-server-a-claude-code)
- [Step 5 — Verifica e utilizzo](#-step-5--verifica-e-utilizzo)
- [Esempi di utilizzo](#-esempi-di-utilizzo)
- [Risoluzione dei problemi](#-risoluzione-dei-problemi)

---

## 🧭 Panoramica

Questa guida spiega come collegare **Claude Code CLI** al **KiCad MCP Server** su sistemi Windows.

Su Linux/macOS l'installazione è spesso immediata; su Windows emergono quattro problemi specifici:

1. **Python sbagliato** — il server MCP ha bisogno che `PYTHONPATH` punti alla cartella Python di KiCad (quella che contiene la libreria `pcbnew`). Usare il Python di sistema causa un crash all'avvio.
2. **`python.exe` non trovato** — anche con `PYTHONPATH` corretto, se la cartella `bin` di KiCad non è nel `PATH` di sistema il server MCP non riesce ad avviare il processo Python e restituisce `Python process for KiCAD scripting is not running`.
3. **Variabili d'ambiente rotte** — passare variabili complesse via PowerShell al comando `claude mcp add` spesso fallisce o viene interpretato in modo errato.
4. **Percorso `site-packages` diverso da Linux** — su Windows con KiCad 10.0 la cartella è `bin\Lib\site-packages`, non `lib/python3/dist-packages` come su Linux/KiCad 9.0.

**La soluzione** è un wrapper batch (`start.cmd`) che imposta correttamente tutte e tre le variabili necessarie: `PYTHONPATH`, `KICAD_BIN_PATH` e `PATH`.

---

## 📋 Prerequisiti

| Strumento | Versione minima | Note |
|---|---|---|
| [Node.js](https://nodejs.org/) | v18+ | Necessario per il server MCP |
| [Python](https://www.python.org/) | 3.10+ | Usare il Python di sistema per `pip`, non quello di KiCad |
| [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) | ultima | Deve funzionare già standalone |
| [KiCad](https://www.kicad.org/) | **9.0+** | Versione minima richiesta dal server MCP |
| Git | qualsiasi | Per clonare il repository del server |

> ⚠️ **KiCad 8.0 non è supportato** dal server MCP ufficiale. Aggiorna almeno alla 9.0.

---

## 🛠️ Step 1 — Clonare e compilare il server

### Opzione A — Setup automatico (consigliato)

Il repo ufficiale include uno script PowerShell che installa tutto automaticamente:

```powershell
git clone https://github.com/mixelpixx/KiCAD-MCP-Server.git
cd KiCAD-MCP-Server
.\setup-windows.ps1
```

Se lo script viene bloccato dall'execution policy, esegui prima:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Opzione B — Setup manuale

```powershell
git clone https://github.com/mixelpixx/KiCAD-MCP-Server.git
cd KiCAD-MCP-Server
npm install
pip install -r requirements.txt
npm run build
```

> ⚠️ **Non saltare `pip install -r requirements.txt`** — le dipendenze Python sono obbligatorie per il funzionamento del server.

Verifica che al termine esista la cartella `dist/` con il file JS al suo interno:

```powershell
Get-ChildItem .\dist\
```

---

## 🔍 Step 2 — Trovare i percorsi di KiCad

Il server MCP ha bisogno che `PYTHONPATH` punti alla cartella che contiene `pcbnew.py` e che `python.exe` di KiCad sia raggiungibile tramite il `PATH`.

> ⚠️ **Attenzione:** il percorso cambia tra versioni di KiCad e tra Windows e Linux.  
> Su Windows **non** usare `dist-packages` (percorso Linux) — la cartella corretta dipende dalla versione installata.

### Trovare automaticamente i percorsi

Esegui questi due comandi PowerShell per trovare i percorsi esatti sul tuo sistema:

```powershell
# Trova python.exe di KiCad
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "python.exe" -ErrorAction SilentlyContinue

# Trova la cartella che contiene pcbnew.py
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "pcbnew.py" -ErrorAction SilentlyContinue
```

### Percorsi verificati per versione

| Versione KiCad | Cartella `bin` | `PYTHONPATH` |
|---|---|---|
| **10.0** (installazione utente — Windows) | `C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin` | `C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\Lib\site-packages` |
| **9.0** (installazione globale — Windows) | `C:\Program Files\KiCad\9.0\bin` | `C:\Program Files\KiCad\9.0\lib\python3\dist-packages` |

> 💡 Sostituisci `TUO_UTENTE` con il tuo nome utente Windows.

### ✅ Verifica obbligatoria — test `pcbnew`

Prima di procedere, esegui questo comando per confermare che `pcbnew` sia raggiungibile. Se il test fallisce, il server MCP non funzionerà:

```powershell
# KiCad 10.0 — adatta il percorso alla tua versione
& "C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe" -c "import pcbnew; print(pcbnew.GetBuildVersion())"
```

Output atteso:
```
10.0.0
```

---

## 📝 Step 3 — Creare lo script di avvio `start.cmd`

Questo file batch imposta tutte le variabili d'ambiente necessarie e avvia il server. Sono necessarie **tre variabili**:

| Variabile | Scopo |
|---|---|
| `PYTHONPATH` | Dice a Python dove trovare `pcbnew` e le altre librerie KiCad |
| `KICAD_BIN_PATH` | Usata dal server MCP per localizzare i tool di KiCad |
| `PATH` | Permette al server MCP di trovare ed eseguire `python.exe` di KiCad |

> ⚠️ **La riga `set PATH=...` è indispensabile.** Senza di essa il server MCP non riesce ad avviare il processo Python e restituisce l'errore `Python process for KiCAD scripting is not running`, anche se `PYTHONPATH` è corretto.

Crea il file direttamente da PowerShell con questo comando (più affidabile del Blocco Note):

**Per KiCad 10.0 (installazione utente):**

```powershell
Set-Content -Path "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd" -Encoding ascii -Value "@echo off", "set PYTHONPATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\Lib\site-packages", "set KICAD_BIN_PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin", "set PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin;%PATH%", "node `"C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js`""
```

**Per KiCad 9.0 (installazione globale):**

```powershell
Set-Content -Path "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd" -Encoding ascii -Value "@echo off", "set PYTHONPATH=C:\Program Files\KiCad\9.0\lib\python3\dist-packages", "set KICAD_BIN_PATH=C:\Program Files\KiCad\9.0\bin", "set PATH=C:\Program Files\KiCad\9.0\bin;%PATH%", "node `"C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js`""
```

Verifica che il contenuto sia corretto:

```powershell
Get-Content .\start.cmd
```

Output atteso (KiCad 10.0):
```batch
@echo off
set PYTHONPATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\Lib\site-packages
set KICAD_BIN_PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin
set PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin;%PATH%
node "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js"
```

> 💡 Se dopo la build il file generato si chiama `kicad-server.js` invece di `index.js`, aggiorna l'ultima riga. Verificalo con `Get-ChildItem .\dist\`.

### Abilitare il debug (opzionale)

Per registrare l'intera sessione MCP nella cartella `logs/` del progetto, aggiungi `set KICAD_MCP_DEV=1` come seconda riga del file:

```batch
@echo off
set KICAD_MCP_DEV=1
set PYTHONPATH=...
set KICAD_BIN_PATH=...
set PATH=...
node "..."
```

> 🔒 **Privacy:** il log contiene la cronologia completa dei tool call, inclusi percorsi file e dettagli del design. Elimina la cartella `logs/` prima di pubblicare o condividere il progetto.

---

## 🔌 Step 4 — Collegare il server a Claude Code

### Come funziona la configurazione su Windows

> ⚠️ **Comportamento specifico di Windows:** a differenza di Linux/macOS dove il `.claude.json` viene creato nella cartella del progetto corrente, su Windows Claude Code salva la configurazione MCP nel file **globale** `C:\Users\TUO_UTENTE\.claude.json`. Questo significa che il server KiCad sarà disponibile in **tutte** le sessioni Claude Code, indipendentemente dalla cartella di lavoro.

### Aggiungere il server

Dalla cartella del server MCP, esegui:

```powershell
claude mcp add kicad cmd /c "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd"
```

Output atteso:

```
Added stdio MCP server kicad with command: cmd /c C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd to local config
File modified: C:\Users\TUO_UTENTE\.claude.json [project: C:\...]
```

Se ricevi `MCP server kicad already exists`, rimuovilo e riaggiungilo:

```powershell
claude mcp remove kicad
claude mcp add kicad cmd /c "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd"
```

### Struttura del `.claude.json` generato

```json
{
  "mcpServers": {
    "kicad": {
      "command": "cmd",
      "args": ["/c", "C:\\Users\\TUO_UTENTE\\KiCAD-MCP-Server\\start.cmd"]
    }
  }
}
```

---

## ✅ Step 5 — Verifica e utilizzo

Avvia Claude Code:

```powershell
claude
```

Digita il comando di diagnostica:

```
/mcp
```

Output atteso:

```
1 server
  Local MCPs (C:\Users\TUO_UTENTE\.claude.json [...])
> kicad · √ connected
```

Fai un primo test funzionale chiedendo a Claude di creare un progetto:

```
Crea un nuovo progetto KiCad chiamato "MioProgetto" in C:\KicadProjects
```

---

## 🎯 Esempi di utilizzo

Una volta connesso, puoi interagire con i tuoi file KiCad in linguaggio naturale:

```
Quanti tool KiCad hai disponibili? Elencami le categorie.
```

```
Crea un nuovo progetto KiCad chiamato "ESP32_uPLC" in C:\Kicad_MCP_Test
```

```
Analizza il file della board in questa cartella. Quanti componenti ci sono?
```

```
Esegui un controllo DRC sulla scheda e mostrami tutti gli errori.
```

```
Genera la distinta base (BOM) estraendola dal file dello schema.
```

```
Elenca tutte le net della board e le relative connessioni.
```

```
Esporta i file Gerber per la produzione.
```

---

## 🔧 Risoluzione dei problemi

### `Python process for KiCAD scripting is not running`

È l'errore più comune su Windows. Significa che il server MCP non riesce a trovare `python.exe` di KiCad. La causa è la mancanza della riga `set PATH=...` nello `start.cmd`.

Assicurati che il tuo `start.cmd` contenga questa riga (adatta il percorso alla tua versione):

```batch
set PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin;%PATH%
```

### `ModuleNotFoundError: No module named 'pcbnew'`

`PYTHONPATH` punta alla cartella sbagliata. Cerca `pcbnew.py` con:

```powershell
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "pcbnew.py" -ErrorAction SilentlyContinue
```

Poi aggiorna `PYTHONPATH` in `start.cmd` con la cartella trovata.

### Il server risulta disconnesso (`✗ failed`)

- Esegui `start.cmd` direttamente da PowerShell per leggere il messaggio d'errore completo.
- Verifica che `pip install -r requirements.txt` sia stato eseguito.
- Controlla che `npm run build` abbia prodotto almeno un file `.js` nella cartella `dist/`.

### `MCP server kicad already exists`

```powershell
claude mcp remove kicad
claude mcp add kicad cmd /c "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd"
```

### Il file `start.cmd` contiene testo sbagliato

Non usare il Blocco Note — usa il comando `Set-Content` di PowerShell come descritto nello Step 3. Il copia-incolla nel terminale può introdurre caratteri inattesi.

### `setup-windows.ps1` bloccato da PowerShell

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## 🙏 Credits

- Server MCP originale: [mixelpixx/KiCAD-MCP-Server](https://github.com/mixelpixx/KiCAD-MCP-Server)
- Guida scritta da **Caricek** ([@BitMakerMan](https://github.com/BitMakerMan))

---

## 📄 Licenza

Distribuito sotto licenza [MIT](LICENSE).
