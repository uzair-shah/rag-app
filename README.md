# Juggling Paper RAG

A small retrieval-augmented generation (RAG) app that answers questions about a single PDF, Warrington's *Juggling Probabilities* (2005), and checks how well two LLMs do it.

## How it works

1. **Read** the PDF and pull out its text.
2. **Clean** it by removing boilerplate (licence lines, journal headers), null characters and blank lines.
3. **Chunk** the text into 800-character pieces that overlap by 100 characters, so context isn't cut off mid-thought.
4. **Embed** each chunk with `all-MiniLM-L6-v2` and store it in a local ChromaDB database (`./chroma_db`). This only happens on the first run.
5. **Search**: for each question, find the most similar chunks.
6. **Answer**: send those chunks plus the question to an LLM, told to answer *only* from the context:
   - **Gemini** (`gemini-3-flash-preview`, via the Google API)
   - **DeepSeek-R1** (run locally via Ollama)
7. **Evaluate**: run every question in `test_queries.txt` and score the answers:
   - Questions the paper *can* answer pass if the reply contains an expected keyword.
   - Questions it *can't* answer pass if the model refuses (e.g. "the context does not contain…").

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install pypdf sentence-transformers chromadb google-genai ollama python-dotenv numpy
ollama pull deepseek-r1
```

Create a `.env` file with your Gemini key:

```
GEMINI_API_KEY=your-key-here
```

Put `Warrington-JugglingProbabilities-2005.pdf` and `test_queries.txt` in the project folder.

## Run

```bash
python rag.py
```

It prints each answer's response time and pass/fail, then a summary of the DeepSeek results.

## Test file format

`test_queries.txt` is a JSON list of objects such as:

```json
[
  {
    "question": "What is a juggling process?",
    "category": "definition",
    "should_answer": true,
    "expected_keywords": ["Markov", "state"]
  },
  {
    "question": "Who won the 2022 World Cup?",
    "category": "out_of_scope",
    "should_answer": false,
    "refusal_keywords": ["does not contain", "not mentioned"]
  }
]
```
