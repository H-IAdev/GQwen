# 🧠 Long-Term Memory (MEMORY.md)

## User Profile & Hardware Context
- **OS**: Windows (Windows 10 LTSC mentioned in hardware profile).
- **CPU**: Intel Core i3-3110M (2 cores, 4 threads, 2.4 GHz, no AVX2 instructions).
- **RAM**: 8 GB (approx. 5.5-6 GB available after OS overhead).
- **GPU**: Integrated Intel HD 4000 (no CUDA support).
- **Workspaces**:
  - `D:\mind\SpaceCode`
  - `D:\mind\kadath`
  - `D:\mind\TheresiaWorkspace`
  - `c:\Users\User\Mi unidad (elhombrefunesto@gmail.com)\Wiki STUDY`

## Technical Decisions
- **MCP Configuration**: The workspace uses `opencode` which runs `browsermcp` (`@browsermcp/mcp` via `npx`). We can define a second local MCP server (e.g., `custom-system-mcp` using Node.js) to allow local system access without interfering with the existing one.
- **RAG & Inference Architecture**: Due to severe CPU/RAM limitations and lack of AVX2 support, running 3B+ parameter local LLMs is not feasible for a smooth workflow. The recommended architecture is a **hybrid approach** (local lightweight vector search with FAISS/ChromaDB + cloud APIs like Google Gemini 3.5 Flash for reasoning/vision, and Groq for fast execution).

## Study Engine (RAG + Notas + Flashcards) Commands & Keywords
- **Project Location**: `D:\mind\SpaceCode\Proyectos\02_ai_research\study-engine`
- **Activation Keywords**: `start-rag`, `inicia-rag`, `lanza el asistente RAG`, `/start-rag`, `arranca el RAG`
- **Start UI (Gradio)**: `.\venv\Scripts\python main.py --ui` (Runs on `http://127.0.0.1:7860`)
- **Start API (FastAPI)**: `.\venv\Scripts\python main.py --api` (Runs on `http://127.0.0.1:8003`, backend del frontend React en :8004)
- **Index command**: `.\venv\Scripts\python main.py --index` (Builds/updates vector store)
- **Proactive Suggestions**: Suggest running this study engine whenever the user needs to query, summarize, or cross-reference multiple PDFs, study guides, or long texts.
- **Study Engine Summary**: Motor unificado (fusión de wiki-study-builder + hybrid-rag-assistant + aletheia_learning_engine). Chat RAG con búsqueda semántica local (ChromaDB/ONNX en CPU) y razonamiento Gemini 3.5 Flash / Groq Llama 3.3, voz (Groq Whisper + edge-tts ES/EN/JA), notas Zettelkasten/Feynman para Obsidian, y flashcards/escolios exportados a `04_notes/learning`.

## Wiki Note Compiler Commands & Keywords (dentro de study-engine)
- **Project Location**: `D:\mind\SpaceCode\Proyectos\02_ai_research\study-engine`
- **MCP Tool**: `run_wiki_builder` (integrated into `custom-system-mcp` inside `d:\mind\opencode.json`)
- **CLI Compilation Command**: `.\venv\Scripts\python main.py --file raw_inputs/<file> --type <zettel|feynman|cards> --provider <Gemini|Groq>`
- **CSS Stylesheet Location**: `c:/Users/User/Mi unidad (elhombrefunesto@gmail.com)/Wiki STUDY/.obsidian/snippets/study-style.css`
- **Builder Summary**: Compiles structural study notes (Zettelkasten / Feynman style) into the Wiki STUDY vault, generates flashcards/scholiums (Aletheia), enriches with DuckDuckGo searches, generates Obsidian backlinks/tags, and applies custom reading typography. Includes rate-limit retries and Windows console UTF-8 safety configurations.

## Game Development Architecture & Lessons Learned (LUMEN ST — Box-ia)
- **GLTF Model Instancing & Orientation**: Official Three.js GLTF assets (like `RobotExpressive.glb`) may have internal root bone offsets or 180° rotations (`robotClone.rotation.y = 0` required for forward facing).
- **GLTF Death Animation Root Elevation**: Skeletal death animations often elevate or shift the root bone during fall. To prevent fallen models from floating in mid-air, adjust Y elevation (`g.position.y = getFloorY(x, z) - 0.28`) during downed states with `clampWhenFinished = true`.
- **Hybrid 3D Loading Pattern**: Always initialize 3D scenes with instant 0ms procedural geometry placeholders first, then asynchronously fetch and swap HD GLTF models with `AnimationMixer` in the background. This ensures 0ms load screens and 100% offline fallback safety.
- **WebGL Surface Material Selection**: Avoid using heavy `MeshPhysicalMaterial` clearcoat water sheens on large floor surfaces as they degrade tile clarity. Use clean `MeshStandardMaterial` (`roughness: 0.55`) for crisp, high-contrast industrial floor environments.
- **Bot AI Autonomy vs Tethering**: Never tether bot target positions directly to `camera.position` every frame to prevent mirror-movement desync. Bots must use independent patrol waypoints (`b.patrolPos`) and only regroup when the map is free of enemies.
