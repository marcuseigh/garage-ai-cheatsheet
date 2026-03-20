# A Cowboy's Guide to Setting Up Your Own Garage AI Agent
## Copy-Paste Command Cheatsheet

> Companion to the book by Marcus Eigh — CowboyAI Press, 2026
> All commands tested on macOS with Apple Silicon (M4/M5 Mac Mini)
> Replace `yourname` and paths with your actual username and folder locations

---

## Table of Contents

- [Chapter 2 — macOS Setup](#chapter-2--macos-setup)
- [Chapter 3 — Ollama: Running Local Models](#chapter-3--ollama-running-local-models)
- [Chapter 4 — MCP: Connecting Tools](#chapter-4--mcp-connecting-tools)
- [Chapter 5 — RAG: Document Memory](#chapter-5--rag-document-memory)
- [Chapter 6 — Automation & Agents](#chapter-6--automation--agents)
- [Appendix — Quick Reference](#appendix--quick-reference)

---

## Chapter 2 — macOS Setup

### Install Homebrew (macOS package manager)

```bash
/bin/bash -c "$(curl -fsSL \
  https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Verify Homebrew installed correctly

```bash
brew doctor
```

### Connect to your Mac Mini remotely over SSH

```bash
ssh yourusername@macmini.local
```

> If `macmini.local` doesn't resolve, use the Mac Mini's IP address instead (find it in System Settings → Wi-Fi or Ethernet).

---

## Chapter 3 — Ollama: Running Local Models

### Install Ollama

```bash
brew install ollama
```

### Start the Ollama service (foreground)

```bash
ollama serve
```

> Ollama will listen on `http://localhost:11434`. Leave this running in a terminal tab.

### Pull your first model

```bash
ollama pull llama3.2
```

### Chat with a model interactively

```bash
ollama run llama3.2
```

> Type `/bye` and press Enter to exit the chat. The model stays loaded in memory.

### Model management

```bash
# List all downloaded models
ollama list

# Remove a model you no longer need
ollama rm modelname

# Show model details
ollama show modelname

# Pull a specific version / size
ollama pull llama3.1:70b

# Check what is currently running
ollama ps
```

### Run Ollama as a background service (auto-starts on boot)

```bash
brew services start ollama
```

### Stop or check background service status

```bash
brew services stop ollama
brew services list
```

### Test the Ollama API directly

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "What is the capital of Texas?",
  "stream": false
}'
```

### Allow other devices on your network to access Ollama

```bash
# Set this before running ollama serve
export OLLAMA_HOST=0.0.0.0
ollama serve
```

> Any device on your local network can then reach your model at `http://192.168.x.x:11434` (use your Mac Mini's static IP).

### Recommended models reference

| Model | Best For | RAM Needed |
|---|---|---|
| llama3.2 (3B) | Fast Q&A, quick tasks | ~2 GB |
| llama3.1 (8B) | General purpose | ~5 GB |
| mistral (7B) | Instruction following, chat | ~4 GB |
| codellama (13B) | Code generation, debugging | ~8 GB |
| llama3.1 (70B) | Maximum capability | ~40 GB |
| nomic-embed-text | RAG embeddings (Ch. 5) | <1 GB |
| phi3 (3.8B) | Reasoning tasks | ~2 GB |

---

## Chapter 4 — MCP: Connecting Tools

### Install Node.js (required for MCP servers)

```bash
brew install node
```

### Install core MCP servers

```bash
npm install -g @modelcontextprotocol/server-filesystem
npm install -g @modelcontextprotocol/server-memory
npm install -g @modelcontextprotocol/server-brave-search
```

### Run the filesystem MCP server

```bash
# Replace /Users/yourname/Documents with the folder you want the AI to access
npx @modelcontextprotocol/server-filesystem /Users/yourname/Documents
```

> Leave this running in a terminal window. The AI client connects to it on demand.

### Install Docker (required for Open WebUI)

```bash
brew install --cask docker
```

### Install Open WebUI (browser interface for Ollama)

```bash
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

> Open `http://localhost:3000` in your browser for a ChatGPT-style interface backed by your local models.

### MCP client configuration file (`mcp-config.json`)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Documents"
      ]
    },
    "memory": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-memory"]
    }
  }
}
```

> Save this file in a convenient location. Point your MCP-compatible client (Claude Cowork, Open WebUI, etc.) to this file in its settings.

### Test / debug MCP servers with the Inspector

```bash
npx @modelcontextprotocol/inspector
```

> Opens a browser-based debugging interface where you can call tools manually and inspect responses.

---

## Chapter 5 — RAG: Document Memory

### Pull the embedding model

```bash
ollama pull nomic-embed-text
```

### Install Chroma and Python dependencies

```bash
# Install Python first if needed
brew install python

# Install Chroma
pip3 install chromadb

# Install supporting libraries
pip3 install ollama langchain langchain-community
```

### Start the Chroma vector database server

```bash
chroma run --path /Users/yourname/chroma-data
```

> Chroma listens on port 8000 by default.

### Index your documents — `index_docs.py`

```python
import ollama
import chromadb
import os

# Connect to Chroma
client = chromadb.HttpClient(host="localhost", port=8000)
collection = client.get_or_create_collection("my_documents")

def chunk_text(text, chunk_size=500, overlap=50):
    """Split text into overlapping chunks."""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks

def index_file(filepath):
    with open(filepath, "r", encoding="utf-8") as f:
        text = f.read()
    chunks = chunk_text(text)
    for i, chunk in enumerate(chunks):
        response = ollama.embeddings(
            model="nomic-embed-text",
            prompt=chunk
        )
        collection.add(
            ids=[f"{filepath}_{i}"],
            embeddings=[response["embedding"]],
            documents=[chunk],
            metadatas=[{"source": filepath, "chunk": i}]
        )
    print(f"Indexed {len(chunks)} chunks from {filepath}")

# Index all .txt and .md files in a folder
folder = "/Users/yourname/Documents/my-knowledge-base"
for filename in os.listdir(folder):
    if filename.endswith(".txt") or filename.endswith(".md"):
        index_file(os.path.join(folder, filename))

print("Indexing complete!")
```

### Run the indexer

```bash
python3 index_docs.py
```

### Query your RAG system — `ask_rag.py`

```python
import ollama
import chromadb

client = chromadb.HttpClient(host="localhost", port=8000)
collection = client.get_or_create_collection("my_documents")

def ask(question, model="llama3.1", top_k=5):
    # Get embedding for the question
    q_embedding = ollama.embeddings(
        model="nomic-embed-text",
        prompt=question
    )["embedding"]

    # Find the most relevant chunks
    results = collection.query(
        query_embeddings=[q_embedding],
        n_results=top_k
    )

    # Build context from retrieved chunks
    context = "\n\n".join(results["documents"][0])

    # Ask the language model using the retrieved context
    prompt = f"""You are a helpful assistant. Use the following context
to answer the question. If the answer is not in the context, say so.

Context:
{context}

Question: {question}

Answer:"""

    response = ollama.generate(model=model, prompt=prompt)
    return response["response"]

# Interactive loop
while True:
    question = input("\nAsk a question (or type quit): ")
    if question.lower() == "quit":
        break
    answer = ask(question)
    print(f"\nAnswer: {answer}")
```

### Run the RAG query script

```bash
python3 ask_rag.py
```

### Add PDF and Word document support

```bash
pip3 install pypdf python-docx
```

```python
# For PDF files
from pypdf import PdfReader

def read_pdf(filepath):
    reader = PdfReader(filepath)
    return " ".join(page.extract_text() for page in reader.pages)

# For Word documents
from docx import Document

def read_docx(filepath):
    doc = Document(filepath)
    return " ".join(para.text for para in doc.paragraphs)
```

### Install watchdog for automatic re-indexing

```bash
pip3 install watchdog
```

---

## Chapter 6 — Automation & Agents

### Install Open Interpreter

```bash
pip3 install open-interpreter

# Run with your local Ollama model
interpreter --model ollama/llama3.1
```

```python
# Or configure in a Python script
from interpreter import interpreter
interpreter.llm.model = "ollama/llama3.1"
interpreter.llm.api_base = "http://localhost:11434/v1"
interpreter.chat("Organize all the PDF files in my Downloads folder by date")
```

### Install LangChain + LangGraph

```bash
pip3 install langchain langchain-community langgraph
```

```python
from langchain_community.llms import Ollama
from langchain.agents import create_react_agent, AgentExecutor

llm = Ollama(model="llama3.1", base_url="http://localhost:11434")
# Define tools, build agent, set it to work
```

### Install CrewAI (multi-agent framework)

```bash
pip3 install crewai
```

```python
from crewai import Agent, Task, Crew
from langchain_community.llms import Ollama

llm = Ollama(model="llama3.1")

researcher = Agent(
    role="Researcher",
    goal="Find key information on the topic",
    llm=llm
)

writer = Agent(
    role="Writer",
    goal="Produce a clear summary",
    llm=llm
)
```

### Daily briefing agent — `daily_briefing.py`

```python
import ollama
import os
from datetime import datetime

WATCH_FOLDER = "/Users/yourname/Documents/inbox"
OUTPUT_FOLDER = "/Users/yourname/Documents/briefings"

def get_new_files():
    """Find files modified in the last 24 hours."""
    import time
    cutoff = time.time() - 86400
    return [
        os.path.join(WATCH_FOLDER, f)
        for f in os.listdir(WATCH_FOLDER)
        if os.path.getmtime(os.path.join(WATCH_FOLDER, f)) > cutoff
    ]

def summarize_file(filepath):
    with open(filepath, "r") as f:
        content = f.read()
    response = ollama.generate(
        model="llama3.1",
        prompt=f"Summarize this document in 3 bullet points:\n\n{content}"
    )
    return response["response"]

def run_daily_briefing():
    files = get_new_files()
    if not files:
        print("No new files today.")
        return

    date_str = datetime.now().strftime("%Y-%m-%d")
    report_lines = [f"Daily Briefing — {date_str}\n", "=" * 40, ""]

    for filepath in files:
        filename = os.path.basename(filepath)
        summary = summarize_file(filepath)
        report_lines.append(f"FILE: {filename}")
        report_lines.append(summary)
        report_lines.append("")

    report = "\n".join(report_lines)
    output_path = os.path.join(OUTPUT_FOLDER, f"briefing_{date_str}.txt")
    with open(output_path, "w") as f:
        f.write(report)
    print(f"Briefing saved to {output_path}")

run_daily_briefing()
```

### Schedule the briefing with a macOS LaunchAgent

Save the following as `~/Library/LaunchAgents/com.garageai.briefing.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.garageai.briefing</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/python3</string>
    <string>/Users/yourname/scripts/daily_briefing.py</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>7</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```

### Load the LaunchAgent (activate the schedule)

```bash
launchctl load ~/Library/LaunchAgents/com.garageai.briefing.plist
```

> The briefing script will now run automatically every morning at 7:00 AM.

---

## Appendix — Quick Reference

### Ollama one-liners

```bash
ollama serve                   # Start Ollama
ollama pull llama3.1           # Download a model
ollama run llama3.1            # Chat with a model
ollama list                    # List local models
ollama rm modelname            # Delete a model
ollama ps                      # Show running models
brew services start ollama     # Run as background service
```

### MCP one-liners

```bash
npx @modelcontextprotocol/server-filesystem /path   # File server
npx @modelcontextprotocol/server-memory             # Memory server
npx @modelcontextprotocol/inspector                 # Debug server
```

### Chroma one-liners

```bash
chroma run --path /path/to/data    # Start vector database
pip3 install chromadb              # Install Chroma
pip3 install ollama langchain      # Install Python dependencies
```

### Open WebUI (full Docker command)

```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

---

### Key URLs

| Resource | URL |
|---|---|
| Ollama | https://ollama.com |
| Ollama model library | https://ollama.com/library |
| MCP documentation | https://modelcontextprotocol.io |
| Open WebUI | https://openwebui.com |
| Chroma | https://trychroma.com |
| Open Interpreter | https://openinterpreter.com |
| LangChain | https://langchain.com |
| CrewAI | https://crewai.com |
| Homebrew | https://brew.sh |
| r/LocalLLaMA | https://reddit.com/r/LocalLLaMA |

---

*A Cowboy's Guide to Setting Up Your Own Garage AI Agent — Marcus Eigh / CowboyAI Press*
*github.com/marcuseigh/garage-ai-cheatsheet*
