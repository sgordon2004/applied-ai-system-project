# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
DocuBot is a documentation assistant that helps answer developer questions about a codebase. Instead of making someone read through every file manually, it tries to find the relevant parts and either return them directly or use a language model to produce a clean answer. The goal is to make project documentation more accessible and easier to query.

**What inputs does DocuBot take?**  
A natural language question typed by the user, a folder of markdown documentation files in the docs/ directory, and optionally a Gemini API key to enable LLM-based modes. The system reads all .md and .txt files from the docs folder at startup.

**What outputs does DocuBot produce?**  
Depending on the mode, DocuBot either returns raw text snippets from the most relevant documentation sections (retrieval only), or a generated answer written by Gemini that is grounded in those snippets (RAG mode). In naive LLM mode, it returns a Gemini response with no retrieval at all.

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Documents are first split into chunks by markdown headers, so each section becomes its own retrievable unit with a filename and heading attached. At query time, each chunk is scored for relevance.

For keyword-based retrieval, scoring works by extracting words from the query, filtering out common stop words, applying a simple stemmer to normalize word forms, and counting how many of those stems appear in the chunk. The text is also preprocessed to split underscore-separated words, which helps match things like function names.

For embedding-based retrieval, if a Gemini API key is available, the system pre-computes an embedding vector for every chunk at startup, then computes cosine similarity between the query embedding and each chunk embedding at query time. The top-k chunks by similarity score are returned.

- Indexing: inverted index from token to list of filenames, plus per-chunk embedding vectors when the LLM client is available
- Scoring: stemmed keyword overlap for keyword mode, cosine similarity for embedding mode
- Selection: top 3 chunks by score

**What tradeoffs did you make?**  
The keyword scorer is fast but brittle. It misses synonyms and semantic meaning entirely. For example, a query about "credentials" won't match a document that only talks about "auth tokens" even though those mean the same thing in context.

Embedding retrieval fixes that problem but requires an API call per query and pre-computes embeddings for all chunks at startup, which adds latency. It also uses up API quota even for retrieval, which matters if you are rate-limited.

I kept the keyword path as a fallback so the system still works without an API key, just less accurately.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  

- Naive LLM mode: Gemini is called once with just the query. No retrieval happens. The model answers from its own training data.
- Retrieval only mode: No LLM call at all. The system runs scoring and returns the raw matched snippets.
- RAG mode: Retrieval runs first. The top 3 chunks are passed to Gemini as context, and Gemini is instructed to answer using only those snippets.

**What instructions do you give the LLM to keep it grounded?**  
The RAG prompt tells Gemini to use only the information in the provided snippets, not to invent function names or configuration values, and to explicitly say "I do not know based on the docs I have." if the snippets contain nothing relevant. It also asks Gemini to name which files it pulled from when it does answer. If there is partial information, the prompt asks the model to share what it found and note what is missing, rather than refusing entirely.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| What environment variables are required for authentication? | Helpful — gave a plausible answer, but listed generic variables not specific to this project | Helpful — returned the AUTH.md environment variables section | Helpful — correctly identified AUTH_SECRET_KEY and TOKEN_LIFETIME_SECONDS, cited AUTH.md | RAG gave the most precise project-specific answer |
| How do I connect to the database? | Helpful — gave generic SQL connection advice unrelated to this project | Helpful — returned the DATABASE.md section | Helpful — accurate, grounded in DATABASE.md | Naive LLM was confidently wrong about the actual setup |
| Which endpoint lists all users? | Helpful — correctly guessed GET /api/users, but only because that is a common REST convention | Helpful — returned the right API_REFERENCE.md chunk | Helpful — cited the exact endpoint and required auth header | RAG was the most precise here |
| How does a client refresh an access token? | Helpful on the surface, but described a JWT flow we do not actually use | Helpful — retrieved AUTH.md refresh section | Helpful — described the POST /api/refresh endpoint correctly | Naive LLM gave a textbook answer, not a project-specific one |

**What patterns did you notice?**  

Naive LLM looks impressive at first because it gives fluent, well-formatted answers. The problem is it pulls from its general training knowledge, not from the actual project docs. For anything that follows a common convention, it might guess right. For anything project-specific, it invents details that sound reasonable but are wrong.

Retrieval only is reliable but awkward. The returned text is accurate but requires the user to read it themselves and draw their own conclusions. It is more like a search engine than an assistant.

RAG is clearly the best option when retrieval works well. The answers are accurate, grounded, and readable. The main risk is when retrieval fails to surface the right chunk, because then the LLM either guesses (if the prompt is too permissive) or refuses (if the prompt is strict). Getting the prompt tight enough to prevent hallucination while still being useful for partial matches took a few iterations.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  

Failure case 1: Query about payment processing.
- Question: "Is there any mention of payment processing in these docs?"
- What the system did: In RAG mode, it retrieved chunks from SETUP.md, API_REFERENCE.md, and DATABASE.md, then returned a response confirming there is no direct mention of payment processing while briefly describing what each file does cover.
- What should have happened: This is actually reasonable behavior. The model correctly reported the absence of payment content and cited its sources rather than guessing. A stricter prompt would produce a flat refusal, but the current behavior is more informative and still honest.

Failure case 2: Keyword retrieval missing semantic matches.
- Question: "What credentials do I need to run the app?"
- What the system did: In keyword mode, it scored low on all chunks because the word "credentials" does not appear in the docs. It returned weak or unrelated results.
- What should have happened: AUTH.md and SETUP.md both cover this implicitly using terms like "API key," "AUTH_SECRET_KEY," and "environment variables." Embedding retrieval handles this correctly; keyword retrieval does not.

**When should DocuBot say "I do not know based on the docs I have"?**  
At minimum, when no snippets are retrieved at all, and when the retrieved snippets contain no relevant information for the query even after Gemini reviews them. A query about a topic not covered anywhere in the docs should always produce a refusal rather than a guess.

**What guardrails did you implement?**  
The RAG prompt explicitly tells Gemini not to invent function names, endpoints, or configuration values. It requires a citation when answering. The retrieval step acts as a filter — if nothing scores above the minimum threshold in keyword mode, no snippets are passed to the LLM at all, and the system returns the default refusal without making an API call. The "partial information" rule in the prompt tries to prevent the model from staying silent when it has something useful to say, while still being honest about gaps.

---

## 6. Limitations and Future Improvements

**Current limitations**  

1. No conversation memory. Every query is independent. There is no way to ask a follow-up question that refers to a previous answer.
2. The keyword scorer does not understand synonyms or paraphrasing. Queries that use different vocabulary than the docs will score poorly even when the intent is a clear match.
3. The docs corpus is small and static. There is no way to add or update documents without restarting the system, and the embedding index gets rebuilt from scratch every time.

**Future improvements**  

1. Multi-turn conversation history. Passing the last few Q&A exchanges as additional context in the RAG prompt would let users ask follow-up questions naturally.
2. Hybrid retrieval scoring. Combining keyword overlap and cosine similarity into a single weighted score would make retrieval more robust than choosing one or the other.
3. Query rewriting before retrieval. A short LLM call to expand or clarify the user's query before running retrieval would improve recall on vague or under-specified questions.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
The biggest risk is misplaced trust. RAG mode sounds authoritative even when it is wrong, and users who do not check the source docs might act on a hallucinated answer. In a real codebase, that could mean misconfiguring authentication, using a non-existent endpoint, or missing a required environment variable. The naive LLM mode is especially risky because it has no grounding at all and will confidently describe patterns from its training data that may have nothing to do with the actual project.

**What instructions would you give real developers who want to use DocuBot safely?**  

- Always verify answers against the actual source files before making changes to configuration or auth logic.
- Prefer RAG mode over naive LLM mode for any project-specific question. Naive mode is only useful for general background knowledge.
- If DocuBot says "I do not know," treat that as accurate. Do not ask the question in a different way hoping to get an answer — it likely means the docs do not cover it.
- Keep the docs/ folder up to date. Stale documentation will produce stale answers, and the system has no way to know when the docs no longer match the code.

---

## 8. How I Used AI

**How did you use AI while building this project?**
- I used AI as a coding partner for this project. Claude Code helped me ideate features to add, suggested relevant tools/libraries, and audited my framework/architecture design throughout the development process.

**What is one suggestion that was genuinely useful?**
- Claude Code suggested embedding the document corpus and user queries and using cosine similarity to judge the connection between the two. This is the feature that created the most discernible positive difference in my system's performance.

**What is one suggestion that was wrong or required correction?**  
- Claude Code attempted to populate my README with fabricated example responses, rather than real output from my project. I had to run the model myself to see what it actually returned, and then make the appropriate edits to the README.