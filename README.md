# 🔌 KiCad MCP Server + Claude Code CLI — Guida Configurazione Windows

> Scritto da **Caricek** ([@BitMakerMan](https://github.com/BitMakerMan))  
> Collega Claude Code al tuo KiCad e parla con i tuoi PCB in linguaggio naturale.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![KiCad](https://img.shields.io/badge/KiCad-8.0%20%7C%209.0%20%7C%2010.0-blue)](https://www.kicad.org/)
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

1. **Python sbagliato** — il server MCP ha bisogno della versione di Python *integrata in KiCad* (quella che contiene la libreria `pcbnew`). Usare un altro Python causa un crash all'avvio.
2. **Variabili d'ambiente rotte** — passare variabili complesse via PowerShell al comando `claude mcp add` spesso fallisce o viene interpretato in modo errato.

**La soluzione** è un wrapper batch (`start.cmd`) che prepara l'ambiente correttamente prima di avviare il server.

---

## 📋 Prerequisiti

| Strumento | Versione minima | Note |
|---|---|---|
| [Node.js](https://nodejs.org/) | v18+ | Necessario per il server MCP |
| [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) | ultima | Deve funzionare già standalone |
| [KiCad](https://www.kicad.org/) | 8.0 / 9.0 / 10.0 | La guida usa i percorsi della v10.0 |
| Git | qualsiasi | Per clonare il repository del server |

---

## 🛠️ Step 1 — Clonare e compilare il server

Apri **PowerShell** o **Git Bash** e lancia:

```bash
git clone https://github.com/mixelpixx/KiCAD-MCP-Server.git
cd KiCAD-MCP-Server
npm install
npm run build
```

Verifica che la compilazione termini senza errori e che esista la cartella `dist/` con il file `index.js` al suo interno.

---

## 🔍 Step 2 — Trovare i percorsi di KiCad

Prima di creare lo script devi conoscere i percorsi esatti della tua installazione.

> **Sostituisci `TUO_UTENTE`** con il tuo nome utente Windows in tutti i percorsi seguenti.

### KiCad 10.0 (Nightly / cartella utente)

```
C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin
C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe
```

### KiCad 8.0 / 9.0 (installazione globale)

```
C:\Program Files\KiCad\8.0\bin
C:\Program Files\KiCad\8.0\bin\python.exe
```

Puoi verificare rapidamente l'esistenza del Python di KiCad da PowerShell:

```powershell
Test-Path "C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe"
# deve restituire: True
```

---

## 📝 Step 3 — Creare lo script di avvio `start.cmd`

Questo file batch imposta le variabili d'ambiente necessarie e avvia il server.

1. Apri il **Blocco Note** di Windows.
2. Incolla il codice seguente, adattando i percorsi alla tua installazione:

```batch
@echo off
set PYTHON_PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin\python.exe
set KICAD_BIN_PATH=C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad\10.0\bin
node "C:\Users\TUO_UTENTE\KiCAD-MCP-Server\dist\index.js"
```

3. Vai su **File → Salva con nome…**
4. Salva il file **dentro la cartella del server** (es. `C:\Users\TUO_UTENTE\KiCAD-MCP-Server\`).
5. Nel menu a tendina "Salva come" scegli **Tutti i file (`*.*`)** e nomina il file esattamente `start.cmd`.

> ⚠️ Assicurati che l'estensione sia `.cmd` e **non** `.cmd.txt`.

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

Il file `.claude.json` generato avrà una struttura simile a questa:

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
  Tools: (decine di strumenti riconosciuti)
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

---

## 🔧 Risoluzione dei problemi

### Il server risulta disconnesso (`✗ failed`)

- Verifica che i percorsi in `start.cmd` siano corretti e che `python.exe` esista.
- Prova ad eseguire `start.cmd` direttamente da PowerShell per leggere l'eventuale errore.
- Controlla che `npm run build` abbia prodotto il file `dist/index.js`.

### `python.exe` non trovato

Il Python di KiCad si trova sempre nella cartella `bin` dell'installazione. Cercalo con:

```powershell
Get-ChildItem -Path "C:\Users\TUO_UTENTE\AppData\Local\Programs\KiCad" -Recurse -Filter "python.exe"
```

### La configurazione non viene trovata

`.claude.json` è locale al progetto. Se apri Claude Code da un'altra cartella il server MCP non sarà disponibile. Ripeti il comando `claude mcp add` per ogni nuovo progetto KiCad.

### Errori con spazi nel percorso

Assicurati che tutti i percorsi in `start.cmd` siano racchiusi tra virgolette doppie `"..."`.

---

## 🙏 Credits

- Server MCP originale: [mixelpixx/KiCAD-MCP-Server](https://github.com/mixelpixx/KiCAD-MCP-Server)
- Guida scritta da **Caricek** ([@BitMakerMan](https://github.com/BitMakerMan))

---

## 📄 Licenza

Distribuito sotto licenza [MIT](LICENSE).
