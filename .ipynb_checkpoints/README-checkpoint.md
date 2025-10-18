# Coding Challenge: Natural-Compound Extraction from Research PDFs
## Overview
This project implements an **end-to-end Large Language Model (LLM) pipeline** to extract structured compound data from scientific research papers (PDF format).  
The system is designed for **automation, scalability, and self-validation** using **Gemini** for Natural-Compound extraction, AI-based validation and improvement.  

### Objectives
- Extract natural compound information from unstructured research papers.
- Validate and refine extracted data using LLM-based self-evaluation.
- Provide transparent metrics on speed and accuracy.

## High-Level Stage Description
1. **PDF Ingestion:** Load and extract text from research papers, saving in .md files, utilizing recent SOTA for PDF parser, i.e., [Dolphin](https://github.com/bytedance/Dolphin).
2. **Embedding & Vector Store (RAG)**: Convert document chunks into dense vector embeddings using a language model (google/embeddinggemma-300m), i.e., chunk size: 800 tokens with 25% overlap. Store vectors in a retrieval database (ChromaB). Enables semantic search and contextual recall.
3. **Chunk Retrieval**: Given an extraction query (e.g., “find compound-related data”), the retriever fetches top-k relevant text segments. This ensures the LLM sees only the most pertinent context before generation. Because of the paper's complexity, top-k is total segments.
4. **LLM Extraction**: The selected chunks are passed into the Gemini by calling the API (gemini-2.5-pro, gemini-2.5.flash), with a structured prompt template. The LLM outputs a JSON object containing:
	- Compound Name
	- Species
	- Organism
	- Amount
 5. **Post-Processing**: Clean and validate JSON: normalize units and enforce consistent scientific notation. Apply rule-based formatting for missing or partial fields.
 6. **Validation & Improvement**: If we got ground truth, calculate precision, recall, F1. Otherwise, call the Gemini API to evaluate the extraction against the document text via RAG context:
	- Performs semantic comparison between extraction and retrieved chunks.
	- Produces per-field confidence scores and suggests corrections.
	- Can re-query RAG store to verify uncertain fields.

## Experiment and performance
### Accuracy on labeled test
- **Goal:** measure how well the model extracted these fields.
- Given 4 fields: Name, Species, Organism, Amount. Each field is compared between model output and ground truth.
- By implementing matching logic (i.e., fuzzy match, partial match, numeric tolerance match (“71.0 g/kg” ≈ “71.1 g/kg”)), I calculate:
    - True Positives (TP): fields matching under the rules.
    - False Positives (FP): predicted fields incorrect or spurious.
    - False Negatives (FN): missing fields in output.
  Compute per-field precision, recall, F1.
Aggregating across compounds in JSON, we calculate macro-average by computing F1/recall/precision per JSON, then averaging. Treats each JSON equally.
#### Result: (detail in folder output_compound_extraction)
1. **Paper 1:** *Macro Avg. F1 = 0.95*. (output_compound_extraction/acc_coll-et-al-2011-neo-clerodane-diterpenoids-from-ajuga-bracteosa.csv) 
2. **Paper 2:** *Macro Avg. F1 = 0.25*. (output_compound_extraction/acc_J Sci Food Agric - 1999 - Zlatanov - Lipid composition of Bulgarian chokeberry  black currant and rose hip seed oils.csv) 
3. Discussion:
   - Considering performance in the labeled test Paper 2, this low score suggests that the model **struggled significantly** to extract accurate information.  
   - Possible contributing factors:
     - Poor text extraction quality.
     - Unconventional or highly compact layout (e.g., data embedded in tables or footnotes).
     - Different terminology or field ordering than what the prompt expected.
   - Indicates the need for **layout-aware preprocessing** or **retrieval-based chunking** before prompting.
### Latency per stages (detail in notebook.ipynb, i.e, output log)
1. PDF Ingestion: 42s
2. RAG latency: 120ms for indexing and 120s for embedding and calling LLM.
- The latency for calling LLM (Gemini API) occupied 50% or more in e2e pipeline.
### Accuracy on unlabeled test
- I use Gemini for validation commentary.
- i.e., *output_compound_extraction/Gemni_Scoring__00c735fc8eb3dee1b6497b89b04ae7d4-toxins5081392_compounds_extraction.txt*, 
    - Interpretation of Scores
    - **All field-level scores = 1.0**, confirming complete agreement between the extracted JSON and the source document text.
    - Gemini explicitly verified the correctness of:
      - **Compound names:** Accurately transcribed with correct stereochemical notation (e.g., *all-E-lutein*).  
      - **Species:** Both *Dalbergia latifolia* and *Streptomyces* were correctly linked to their respective compounds.  
      - **Organism/source parts:** “Leaves” and “culture filtrate” correctly identified as biological origins.  
      - **Amount fields:** Gemini validated that *3 mg* was extracted where specified and that `'N/A'` was a **semantically correct placeholder** where quantitative data was missing.
    - The feedback demonstrates **high contextual awareness**: Gemini correctly recognized that *all-E-lutein* had no measurable amount reported and treated `'N/A'` as accurate rather than missing.
    - The model also confirmed **cross-references between entities**, verifying that compound–species pairings were not mismatched.
    - This suggests the **validation pipeline can detect nuanced cases** (e.g., missing but contextually justified values) rather than relying only on literal text matching.
#### Overall: 
- Future validation efforts can use similar Gemini feedback as a **reference for automated confidence scaling** or **training data curation** (i.e., using high-confidence samples like this for model fine-tuning).

## Future Work & Improvement Directions
1. Enhanced Prompt Engineering
2. Smarter Retrieval (RAG Optimization), i.e, Hybrid Retrieval Models, Context Re-ranking, Adaptive Chunking
3. Model Fine-Tuning & Adaptation
- Train lightweight adapters (LoRA / PEFT) on validated extraction pairs to improve domain recall (natural-compound extraction).  
- Knowledge Distillation: Distill large model behavior (Gemini or GPT-4) into smaller local models for cost-efficient deployment.
4. Active Learning & Continuous Improvement
- **Low-Confidence Sampling:** Automatically collect low-score samples for manual review or targeted fine-tuning.  
- **Incremental Vector Indexing:** Continuously add newly processed documents and verified extractions into the RAG database.  
- **Feedback Dashboard:** Track model drift, latency, and confidence distribution over time to identify degradation early.  

