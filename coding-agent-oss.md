# Local AI Development Setup: OpenCode + Ollama

This guide walks through setting up a 100% local, free, and privacy-focused AI coding agent using **OpenCode** and **Qwen 2.5 Coder 7B** via **Ollama**.

---

## Prerequisites

- **Machine:** macOS (Apple Silicon recommended, e.g., M-series Mac) or Linux/Windows.
- **Hardware:** 16GB RAM minimum recommended for running 7B models smoothly.
- **Storage:** ~5 GB free space for model weights.

---

## Step 1: Install & Start Ollama

Ollama serves as the local LLM inference engine on your machine.

1. **Install Ollama:**
   - **macOS / Linux terminal installer:**
     ```
     curl -fsSL https://ollama.com/install.sh | bash
     ```
   - *Alternatively, download the desktop app directly from [ollama.com](https://ollama.com).*

2. **Verify Ollama is Running:**
   Ensure the Ollama application or background daemon is active on your system.

---

## Step 2: Download the Qwen 2.5 Coder Model

Pull the 7B coding model weights to your local machine:

```
ollama pull qwen2.5-coder:7b
```

Note: For systems with 32GB+ RAM, you can optionally pull qwen2.5-coder:14b for more complex reasoning tasks.

## Step 3: Install OpenCode CLI
OpenCode provides an interactive Terminal UI (TUI) optimized for local agentic execution and native Ollama tool-calling.
  
  * Run the Official Installer Script
```
curl -fsSL https://opencode.ai/install | bash
```
  * Add OpenCode to Your Terminal PATH (zsh fix)
If opencode is not recognized after installation, append your profile settings to your .zshrc

```
echo 'source ~/.profile' >> ~/.zshrc
source ~/.zshrc
```

Verify Installation
```
opencode --version
```

## Step 4: Launching OpenCode with Your Local Model
  * Navigate to Your Project Repository
  ``` 
  cd /path/to/your/project
  ```
  
  * Launch OpenCode pointing to your local Ollama instance
  ``` 
  opencode --model ollama/qwen2.5-coder:7b
  ```

## Step 5: That is it - you are now ready to VIBE CODE. Recommended Prompting Strategy for 7B Models
  * To get the best results from a 7B local model without hitting tool-calling loops, structure your initial prompt using this bounded format
  
```
[CONTEXT]
Building a Cloudflare Pages app with Pages Functions + D1 (database) + R2 (object storage).

[CORE GOAL]
Create a clean project boilerplate with simple file uploads, room/location tagging, and basic text search.

[CONSTRAINTS]
- Keep all files modular and under 150 lines of code.
- Do NOT run heavy npm installs automatically.
- Use plain HTML/JS/CSS in public/ or a light setup (like Hono for Functions).

[REQUIRED FILE STRUCTURE]
Create the following minimal layout step-by-step:
├── wrangler.json           (Cloudflare config for D1 and R2 bindings)
├── package.json            (Basic dependencies)
├── functions/
│   └── api/
│       ├── items.js         (D1 database CRUD endpoints)
│       └── upload.js        (R2 image handling)
└── public/
    ├── index.html          (Main app UI)
    ├── app.js              (Frontend fetch logic)
    └── styles.css          (Minimal styling)
```
## Useful OpenCode Commands
* /compact      Cleans up long conversation logs while keeping code context fresh.
* /clear        Resets session context for building new isolated features.
* Ctrl + C      Stops current generation or exits the TUI session.
