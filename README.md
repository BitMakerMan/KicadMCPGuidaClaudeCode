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

Su Linux/macOS l'installazione è spesso immediata; su Windows emergono due problemi specifici:

1. **Python sbagliato** — il server MCP ha bisogno che `PYTHONPATH` punti alla cartella Python di KiCad (quella che contiene la libreria `pcbnew`). Usare il Python di sistema causa un crash all'avvio.
2. **Variabili d'ambiente rotte** — passare variabili complesse via PowerShell al comando `claude mcp add` spesso fallisce o viene interpretato in modo errato.

**La soluzione** è un wrapper batch (`start.cmd`) che prepara l'ambiente correttamente prima di avviare il server.

> **Alternativa rapida:** il repo ufficiale include uno script PowerShell automatico `setup-windows.ps1` che esegue tutti i passaggi di installazione in un colpo solo (vedi [Step 1](#-step-1--clonare-e-compilare-il-server)).

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

Il server MCP ha bisogno che `PYTHONPATH` punti alla cartella che contiene `pcbnew.py`.

> ⚠️ **Attenzione:** il percorso cambia tra versioni di KiCad e tra Windows e Linux.  
> Su Windows **non** usare `dist-packages` (percorso Linux) — la cartella corretta è `site-packages`.

### Trovare `python.exe` di KiCad

```powershell
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "python.exe" -ErrorAction SilentlyContinue
```

### Trovare la cartella di `pcbnew.py`

```powershell
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "pcbnew.py" -ErrorAction SilentlyContinue
```

### Percorsi verificati per versione

| Versione KiCad | `python.exe` | `PYTHONPATH` |
|---|---|---|
| **10.0** (installazione utente) | `C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe` | `C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\Lib\site-packages` |
| **9.0** (installazione globale) | `C:\Program Files\KiCad\9.0\bin\python.exe` | `C:\Program Files\KiCad\9.0\lib\python3\dist-packages` |

> 💡 Sostituisci `TUO_UTENTE` con il tuo nome utente Windows.

### ✅ Verifica obbligatoria — test `pcbnew`

Prima di procedere, esegui questo comando per confermare che `pcbnew` sia raggiungibile. Se il test fallisce, il server MCP non funzionerà:

```powershell
# KiCad 10.0
& "C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe" -c "import pcbnew; print(pcbnew.GetBuildVersion())"
```

Output atteso:
```
10.0.0
```

---

## 📝 Step 3 — Creare lo script di avvio `start.cmd`

Questo file batch imposta le variabili d'ambiente necessarie e avvia il server.

> ⚠️ **Attenzione alla variabile:** quella corretta è **`PYTHONPATH`** (standard Python), **non** `PYTHON_PATH`. Usare il nome sbagliato fa sì che il runtime Python ignori completamente la variabile.

1. Apri il **Blocco Note** di Windows.
2. Incolla il codice seguente sostituendo `TUO_UTENTE` con il tuo nome utente:

**Per KiCad 10.0 (installazione utente):**

```batch
@echo off
set PYTHONPATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\Lib\site-packages
set KICAD_BIN_PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin
node "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js"
```

**Per KiCad 9.0 (installazione globale):**

```batch
@echo off
set PYTHONPATH=C:\Program Files\KiCad\9.0\lib\python3\dist-packages
set KICAD_BIN_PATH=C:\Program Files\KiCad\9.0\bin
node "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js"
```

> 💡 Se dopo la build il file generato si chiama `kicad-server.js` invece di `index.js`, aggiorna l'ultima riga di conseguenza. Verificalo con `Get-ChildItem .\dist\`.

3. Vai su **File → Salva con nome…**
4. Salva il file **dentro la cartella del server** (es. `C:\Users\TUO_UTENTE\KiCAD-MCP-Server\`).
5. Nel menu a tendina scegli **Tutti i file (`*.*`)** e nomina il file esattamente `start.cmd`.

> ⚠️ Assicurati che l'estensione sia `.cmd` e **non** `.cmd.txt`.

### Abilitare il debug (opzionale)

Per registrare l'intera sessione MCP nella cartella `logs/` del progetto, aggiungi questa riga prima della riga `node`:

```batch
set KICAD_MCP_DEV=1
```

> 🔒 **Privacy:** il log contiene la cronologia completa dei tool call, inclusi percorsi file e dettagli del design. Elimina la cartella `logs/` prima di pubblicare o condividere il progetto.

---

## 🔌 Step 4 — Collegare il server a Claude Code

> **Importante:** Claude Code salva la configurazione MCP *per progetto* nel file nascosto `.claude.json`.  
> Il comando seguente va eseguito **dalla cartella del tuo progetto KiCad** (quella dove si trovano i file `.kicad_pcb`).

Apri PowerShell nella cartella del progetto ed esegui:

```powershell
claude mcp add kicad cmd /c "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd"
```

Output atteso:

```
Added stdio MCP server kicad with command: cmd /c "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\start.cmd"
File modified: ...\.claude.json [project: C:\Tua\Cartella\Progetto]
```

Il file `.claude.json` generato avrà questa struttura:

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

### Configurazione alternativa diretta (senza wrapper)

Se preferisci non usare il wrapper `.cmd`, puoi configurare le variabili d'ambiente direttamente nel `.claude.json`:

```json
{
  "mcpServers": {
    "kicad": {
      "command": "node",
      "args": ["C:/Users/TUO_UTENTE/KiCAD-MCP-Server/dist/index.js"],
      "env": {
        "PYTHONPATH": "C:/Users/TUO_UTENTE/AppData/Local/Programs/KiCad/10.0/bin/Lib/site-packages",
        "LOG_LEVEL": "info"
      }
    }
  }
}
```

---

## ✅ Step 5 — Verifica e utilizzo

Dalla stessa cartella del progetto, avvia Claude Code:

```powershell
claude
```

Una volta nell'interfaccia, digita il comando di diagnostica:

```
/mcp
```

Se tutto è configurato correttamente vedrai:

```
● kicad  ✓ connected
  Tools: 65+ strumenti organizzati in categorie
```

---

## 🎯 Esempi di utilizzo

Una volta connesso, puoi interagire con i tuoi file KiCad in linguaggio naturale:

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
Ci sono componenti senza footprint assegnato?
```

```
Esporta i file Gerber per la produzione.
```

---

## 🔧 Risoluzione dei problemi

### Il server risulta disconnesso (`✗ failed`)

- Esegui `start.cmd` direttamente da PowerShell per leggere il messaggio d'errore completo.
- Verifica che `pip install -r requirements.txt` sia stato eseguito.
- Controlla che `npm run build` abbia prodotto almeno un file `.js` nella cartella `dist/`.

### `ModuleNotFoundError: No module named 'pcbnew'`

`PYTHONPATH` punta alla cartella sbagliata. Esegui il test di verifica dello Step 2 e cerca `pcbnew.py` con:

```powershell
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Programs\KiCad" -Recurse -Filter "pcbnew.py" -ErrorAction SilentlyContinue
```

Poi aggiorna `PYTHONPATH` in `start.cmd` con la cartella trovata.

### La configurazione non viene trovata

`.claude.json` è locale al progetto. Se apri Claude Code da un'altra cartella il server MCP non sarà disponibile. Ripeti il comando `claude mcp add` per ogni nuovo progetto KiCad.

### Errori con spazi nel percorso

Assicurati che tutti i percorsi in `start.cmd` siano racchiusi tra virgolette doppie `"..."`.

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
