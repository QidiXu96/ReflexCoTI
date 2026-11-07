# ReflexCoTI
## **Model Summary** 
We developed Reflexive Collaborative Thematic Identification Agents (ReflexCoTI) to support RTA with LLMs. ReflexCoTI integrates several specialized agents. First, multiple coder agents with different identity profiles and reflections collaboratively analyze each transcript to identify excerpts and generate initial codes. The reviewer agent merges similar codes across transcripts to construct a codebook. Based on this codebook, multiple themer agents generate candidate themes, while the repairer agent ensures that all identified codes are appropriately represented in the themes.
<img width="3004" height="2452" alt="new reflextheme (2)" src="https://github.com/user-attachments/assets/11306a5b-2a53-4c48-b987-d3b5b86aa0f3" />

## **API Requirement** 
Users must provide their own API credentials to run ReflexCoTI. 

## **Usage**
1. Clone or download this repository. Related ReflexCoTI coding functions at [ReflexTheme_Utils](./ReflexTheme_Utils/). <br>
2. Prepare your own API credentials and set your azure deployment name at [AzureUtils.py](./ReflexTheme_Utils/AzureUtils.py). <br>
3. Open [ReflexCoTI_Covid.ipynb](./ReflexCoTI_Covid.ipynb/). <br>
4. Follow the notebook to configure the API, load transcripts, and run ReflexCoTI. <br>

## **Quick Start**
```bash
git clone https://github.com/QidiXu96/ReflexCoTI.git
cd ReflexCoTI
pip install -r requirements.txt
```
Then open `ReflexCoTI_Covid.ipynb`, provide your own API credentials, and run the notebook cells in order.

## **Run with a Coding Agent**
You can ask a coding agent to read SKILL.md and follow its instructions to configure and run the ReflexCoTI pipeline.
For example, after cloning this repository, ask:
```bash
Read SKILL.md and follow it to run ReflexCoTI on my transcript chunks in <input-directory>. Save the results to <output-directory>. Check the required configuration before execution and validate the generated outputs.
```
You will need to provide your own Azure OpenAI credentials and deployment configuration.
