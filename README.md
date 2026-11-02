# Ask-a-PDF: question answering with RAG

A small retrieval-augmented generation (RAG) app that answers questions about one research paper using only the paper's own text, then measures how accurately and how quickly two AI models do it.

📄 **Full write-up with diagrams:** [uzair-shah.github.io/projects/rag-app.html](https://uzair-shah.github.io/projects/rag-app.html)

## What it does

- Answers plain-English questions about Gregory Warrington's 2005 paper *Juggling Probabilities*.
- Finds the most relevant passages first, then asks a model to answer **only** from those passages, and to say so when the paper doesn't contain the answer.
- Compares a cloud model (Google **Gemini**) with a model running locally (**DeepSeek-R1** via Ollama) on accuracy and response time.
- Exposes the same pipeline as a web API (`POST /ask/`) with FastAPI.

## How it works

1. **Read** the PDF and extract its text page by page (`pypdf`).
2. **Clean** it: strip licence lines, journal headers, null characters and blank lines.
3. **Chunk** the text into 800-character pieces that overlap by 100 characters.
4. **Embed and store** each chunk with `all-MiniLM-L6-v2` in a local ChromaDB collection (`chroma_db/`, 50 chunks). This only runs when the collection is empty.
5. **Retrieve** the closest chunks for each question (3 in the evaluation, 5 in the API).
6. **Answer** with Gemini or DeepSeek-R1, using a prompt that contains only the retrieved context.
7. **Score** every answer against `test_queries.txt`.

## Evaluation and results

`test_queries.txt` holds 12 questions: 10 the paper can answer (facts, definitions, a theorem) and 2 trick questions it can't. An answerable question passes if the reply contains an expected keyword; a trick question passes if the model refuses.

| Model | Passed | Average response time |
|---|---|---|
| Gemini (`gemini-3-flash-preview`) | **10 / 12** | 4.76 s |
| DeepSeek-R1 (local, Ollama) | **9 / 12** | 31.37 s |

Both models missed "What are the references?" and "What's the main idea?". DeepSeek-R1 also failed to refuse one trick question. Results and timings are saved in `rag.ipynb`.

**Limitation:** keyword scoring isn't fully reliable. DeepSeek once said an answer wasn't in the context but answered anyway, and the test didn't catch it.

## Files

| File | What it is |
|---|---|
| `rag.py` | Main script: builds the vector store, runs all 12 test questions through both models, prints each answer with its timing and pass/fail |
| `rag.ipynb` | Development notebook that builds the pipeline step by step, with the saved evaluation results |
| `api.py` | FastAPI service: `POST /ask/` returns Gemini's answer plus the source passages |
| `main.py` | Simpler version of the API that returns only the answer |
| `test_queries.txt` | The 12 test questions with categories, expected keywords and refusal phrases |
| `chroma.ipynb` | Scratch notebook trying out ChromaDB on two toy documents |
| `chroma (discarded).py` | Empty placeholder, not used |
| `chroma_db/` | Saved ChromaDB database with the 50 embedded chunks |
| `Warrington-JugglingProbabilities-2005.pdf` | The paper the app answers questions about |
| `Dockerfile` | Packages the app on `python:3.10-slim` and runs `rag.py` |
| `requirements.txt` | Python dependencies |

## Run it

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ollama pull deepseek-r1
```

Create a `.env` file with your Gemini key:

```
GEMINI_API_KEY=your-key-here
```

Run the evaluation:

```bash
python rag.py
```

Or start the API and send it a question:

```bash
uvicorn api:app --reload
curl -X POST localhost:8000/ask/ -H "Content-Type: application/json" -d '{"question": "What is a juggling process?"}'
```

**Docker note:** the image runs `rag.py`, which needs a Gemini key and a running Ollama server. Neither is inside the container, so pass the key in and point it at an Ollama server on your machine.
