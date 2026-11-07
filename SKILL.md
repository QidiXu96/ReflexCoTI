---
name: run-reflexcoti
description: Run ReflexCoTI to do transcript coding, evolving codebook, multi-agent theme generation in this repository.
---

# Run ReflexCoTI

## Establish the requested scope

Run the repository's Python implementation. Use `ReflexCoTI_Covid.ipynb` as an orchestration example, with the corrections below. 
Do not replace the pipeline with your own thematic analysis in chat.

Select the requested starting point:

- Full run: DOCX chunks → iterative coding → codebook → themes → consensus.
- Stage 1 only: DOCX chunks → iterative coding → codebook.
- Stage 2 only: an existing codebook → themes → consensus.
- Inspect existing results: read saved artifacts without making API calls.

Resolve the input location, research question, output destination, and Azure deployment from the request and available configuration. 
Ask for genuinely missing required information; do not invent it. 
Use the COVID research question and personas only for the COVID example. For other studies, establish suitable research context and persona assumptions before running.

Before running the pipeline, verify that these instructions match the current repository code. If they differ, follow the current implementation and report the discrepancy.

## Locate the implementation

| Component | File / entry point |
| --- | --- |
| Azure client and completion calls | `ReflexCoTI_Utils/AzureUtils.py`: `init_client`, `get_completion` |
| Stage 1 orchestration and codebook | `ReflexCoTI_Utils/RTA_workflow.py`: `run_multi_transcript_dynamic_rounds` |
| Coder prompts and dispatch | `ReflexCoTI_Utils/coder_agent.py` |
| Collaborative reflections | `ReflexCoTI_Utils/coder_collaboration_reflection.py` |
| Chunk merging and deduplication | `ReflexCoTI_Utils/coding_utils.py`, `dedup.py` |
| Remove codebook embeddings | `ReflexCoTI_Utils/codebook_analysis.py`: `remove_embeddings_from_all_final_codebooks` |
| Stage 2 generation and repair | `ReflexCoTI_Utils/themer_agents.py`: `run_multiple_theme_agents_single_round_pipeline` |
| Final consensus | `ReflexCoTI_Utils/consensus_cluster_distinctiveness.py`: `run_final_theme_consensus` |

## Prepare an isolated execution environment

1. Record the repository commit, input files, transcript order, parameters, deployment name, and package versions in a run manifest; omit credentials.
2. Create a Python virtual environment and install `requirements.txt` into it.
   The notebook UI is optional; a Python driver can call the same functions.
3. Set the absolute `ReflexCoTI_Utils` directory on `sys.path` **before** importing utilities. 
   They use bare imports such as `from AzureUtils import ...`.
4. Verify access to the embedding model `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`, downloading it if needed. 
   Verify the installed scikit-learn supports `AgglomerativeClustering(metric=...)`.
5. Use a fresh run directory as the process working directory. 
   The Stage 1 implementation writes to paths relative to that directory. 
   Never run a fresh analysis into an earlier run's directories: reflection history and codebooks are stateful, and rerunning can duplicate evidence.

### Configure Azure before importing downstream modules

Use environment variables or an existing secret store for credentials. 
Suggested variable names are `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_API_VERSION`, and `AZURE_OPENAI_DEPLOYMENT`.

`AzureUtils.get_completion` currently defaults to the literal `your_azure_deployment_name`; setting an environment variable alone does not change it. Many callers import this function directly and omit `model`.

For execution without changing tracked pipeline code, copy the utility directory into the fresh run directory. 
In that runtime copy of `AzureUtils.py`, add `import os` and replace the placeholder default with `model=os.environ["AZURE_OPENAI_DEPLOYMENT"]`. 
Put this runtime utility directory first on `sys.path`, then initialize the shared client:

```python
import os
import AzureUtils

AzureUtils.init_client(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
)
```

Record this runtime adaptation. 
If a newer checkout provides native deployment configuration, use that instead. 
Never print or write secrets into notebooks, logs, source files, or the run manifest. 
Do not silently change model parameters to accommodate an incompatible deployment; report the incompatibility.

## Stage 1: prepare and code transcript chunks

1. Accept already prepared DOCX chunks named
   `Interview_0001_chunk_001.docx`, `Interview_0001_chunk_002.docx`, etc.
   Use four-digit transcript IDs: downstream clue-pool parsing requires them.
   This checkout supplies example chunks, but no raw-transcript semantic chunking entry point. 
   Do not claim to have performed dynamic chunking. If the input is unchunked, establish the preprocessing step before executing Stage 1.
2. Group chunks by transcript and sort transcripts and chunks numerically, using the notebook's `build_files_list` function or equivalent grouping. 
   Retain the padded IDs in filenames. Reject empty input, duplicate chunk IDs, unreadable files, and mixed transcript IDs within a group. Report chunk numberin gaps.
3. Use the implemented agent keys:
   `healthworker`, `policy_maker`, `patient`, `qualitative_researcher`.
   Load `all_agent_initial_reflections` and `agent_personas` from the notebook's source definitions for COVID. Do not execute its API or pipeline cells merely
   to retrieve these dictionaries. 
   Persona descriptions do not dynamically replace coder or themer profiles: inspect their wrapper functions when adapting identities. 
   Changing dispatch keys requires implementation changes.
4. For a different study, update the runtime copy's
   `coder_agent.DEFAULT_RESEARCH_QUESTION` and `DEFAULT_INTERVIEWEE`, inspect the fixed coder/themer/repairer profiles, and align the reflection personas.
5. Preserve the following directory names under the fresh working directory.
   They match hard-coded writers in the current utilities:

```python
from pathlib import Path
import RTA_workflow as rta

dataset_dir = Path("ReflexCoTI in Covid")
dataset_dir.mkdir(parents=True, exist_ok=True)

rta.run_multi_transcript_dynamic_rounds(
    files_list=files_list,
    agents=agents,
    max_rounds=3,
    all_agent_initial_reflections=all_agent_initial_reflections,
    coding_base_dir=str(dataset_dir / "independent_coding"),
    final_coding_base_dir=str(dataset_dir / "final_coding"),
    final_coding_without_score_base_dir=str(dataset_dir / "final_coding_without_score"),
    agent_personas=agent_personas,
    codebook_path=str(dataset_dir / "codebook.json"),
    review_output_dir=str(dataset_dir / "codebook_review_logs"),
    novelty_threshold=0.05,
    k=2,
    clue_pool_dir_for_novelty=None,
)
```

Keep `coder_agent.DEFAULT_OUTPUT_BASE_DIR`,
`coder_collaboration_reflection.REFLECTION_OUTPUT_BASE_DIR`, and the local
`final_base_dir` inside `coding_utils.merge_agent_results` aligned if changing
paths. Passing alternative directories to the orchestrator alone is insufficient.

Preserve the implementation's order: independent coding with prior coding memory;
cleaning and merging; containment/similarity deduplication; novelty evaluation;
collaborative reflection only when another round will run; final-round evidence
filtering; one-code-per-clue selection; sequential codebook updates.

Keep the current defaults unless the request specifies changes:

- Novelty threshold `0.05`, `k=2`, notebook maximum `3` rounds.
- Evidence importance at least `4`; one-code-per-clue similarity `0.80`.
- Codebook retrieval top `3`, reuse threshold `0.80`, compact threshold `0.80`.
- `OPTION_CONFIGS` contains option `1` (skip codebook) and option `3`
  (`code_clue_threshold`). Option 3 is the codebook used for theme generation.

The Stage 1 `codebook_path` argument supplies the **parent location** for option
directories. The actual option-3 result is
`ReflexCoTI in Covid/option_3_code_clue_threshold/codebook.json`.
Do not assume Stage 1 writes the parent `codebook.json`.

Inspect logs and expected per-agent/per-chunk artifacts before marking Stage 1
complete. The orchestrator catches some errors and may continue after failures.
Reviewer failures can add codes as NEW; report these fallback decisions.

## Prepare the codebook for Stage 2

The notebook uses undefined `OPTION_DIRS` and `OPTION_LABELS`. Define them
explicitly rather than executing the notebook unchanged:

```python
from codebook_analysis import remove_embeddings_from_all_final_codebooks

OPTION_DIRS = {3: str(dataset_dir / "option_3_code_clue_threshold")}
OPTION_LABELS = {3: "code_clue_threshold"}
conversion = remove_embeddings_from_all_final_codebooks(
    option_dirs=OPTION_DIRS, option_labels=OPTION_LABELS,
)
if conversion[3]["status"] != "saved":
    raise RuntimeError(conversion[3])
theme_codebook_path = conversion[3]["output_path"]
```

For Stage 2 only, use the explicitly selected existing codebook. The bundled
`ReflexCoTI in Covid/Stage 1 Result/codebook_no_embedding.json` is an example
artifact, not output newly generated by this run.

Require a nonempty `entries` list; each usable entry must contain a nonempty
`initial code` string and a nonempty `clues` list of strings. Preserve
`transcript_ids`. Report entries that the loader would discard.

## Stage 2: generate, repair, and select themes

```python
from themer_agents import run_multiple_theme_agents_single_round_pipeline
from consensus_cluster_distinctiveness import (
    run_final_theme_consensus, extract_final_theme_names,
)

theme_output_dir = dataset_dir / "theme_results"
results = run_multiple_theme_agents_single_round_pipeline(
    codebook_path=str(theme_codebook_path),
    output_dir=str(theme_output_dir),
    research_question=research_question,
    agents=agents,
    no_progress_limit=10,
    verbose=True,
)
```

Inspect each agent's `final_reports`, `failed_init`, and `failed_repair` before
consensus. Require zero missing codes and inspect duplicate/unknown assignments.
Repair can stop without full coverage after the no-progress limit; a successful
function return does not establish complete coverage. If unresolved, report
Stage 2 as incomplete and preserve the artifacts.

Use the returned cluster path, replacing the notebook's hard-coded path:

```python
consensus_results, consensus_path = run_final_theme_consensus(
    final_clusters_path=results["final_clusters_path"],
    final_agent_results=results["final_agent_results"],
    output_dir=str(theme_output_dir / "consensus_distinctive"),
    f1_tolerance=0.10,
    similarity_threshold=0.75,
    lambda_distinctiveness=0.10,
    beam_width=100,
)
final_theme_names = extract_final_theme_names(consensus_results)
```

Clustering embeds code names with supporting clues and uses complete linkage
(distance threshold `0.40`, stop ratio `0.90`, maximum `5` rounds), followed by
small-cluster packing and global code IDs. Consensus code grouping uses **average
linkage**, with rounded mean agent theme count; candidate selection uses F1
eligibility and joint beam search with a redundancy penalty. Preserve these
distinct operations. This checkout ends at cluster-level consensus themes;
do not invent a further global-themer pass.

## Validate outputs and report completion

- Confirm every requested transcript, chunk, agent, and completed round has the
  required artifacts; explicitly identify missing outputs or skipped work.
- Parse codebook, agent results, coverage reports, and consensus JSON. Check code
  IDs against `results["final_clusters_path"]`; do not treat consensus selection
  as proof of complete final code coverage. Report final coverage separately.
- Verify representative quotes against their supplied clues and, where source
  chunks are available, against the transcripts. Report mismatches without
  silently rewriting model outputs. Coverage does not establish quote accuracy.
- Preserve intermediate results, reflection history, review logs, and failed
  responses. Do not blindly restart a partially completed Stage 1 run: it has no
  validated automatic resume contract. Inspect state before any continuation.
- Evaluate similarity to human themes only when actual reference themes are
  supplied; never invent evaluation scores or reference themes.
- Report the scope, input counts, agents, deployment, effective parameters,
  output paths, validation results, failures/fallbacks, and remaining limitations.
  Distinguish checked existing artifacts from newly generated outputs. State
  clearly when credentials or dependencies prevented an API-backed run.

Keep generated analysis available for researcher review and refinement.
