[![Open_inStreamlit](https://img.shields.io/badge/Open%20In-Streamlit-red?logo=Streamlit)](https://llamaindexdocschat.streamlit.app/)
[![Python](https://img.shields.io/badge/python-%203.12-blue.svg)](https://www.python.org/)
[![CodeFactor](https://www.codefactor.io/repository/github/goldzulu/llamaindexdocschat/badge)](https://www.codefactor.io/repository/github/goldzulu/llamaindexdocschat)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](https://github.com/goldzulu/llamaindexdocschat/blob/main/LICENSE)

# Chat with 🦙 LlamaIndex Docs 🗂️

<p align="center">
  <img src="./assets/llamaindexchat.png">
</p>

Chat With LLamaIndex GitHub Repository Docs using [LlamaIndex](https://www.llamaindex.ai/) to supplement OpenAI GPT-3.5 Large Language Model (LLM) with the [LlamaIndex Documentation](https://gpt-index.readthedocs.io/en/latest/index.html). Main features:

- **Transparency and Evaluation**: by customizing the metadata field of documents (and nodes), the App is able to provide links to the sources of the responses, along with the author and relevance score of each source node. This ensures the answers can be cross-referenced with the original content to check for accuracy.
- **Estimating Inference Costs**: tracks 'LLM Prompt Tokens' and 'LLM Completion Tokens' to help keep inference costs under control.
- **Reducing Costs**: persists storage including embedding vectors, and caches the questions / responses to reduce the number of calls to the LLM.
- **Usability**: includes suggestions for questions, and basic functionality to clear chat history.

## 🦙 What's LlamaIndex?

> *LlamaIndex is a simple, flexible data framework for connecting custom data sources to large language models. [...] It helps in preparing a knowledge base by ingesting data from different sources and formats using data connectors. The data is then represented as documents and nodes, where a node is the atomic unit of data in LlamaIndex. Once the data is ingested, LlamaIndex indexes the data into a format that is easy to retrieve. It uses different indexes such as the VectorStoreIndex, Summary Index, Tree Index, and Keyword Table Index. In the querying stage, LlamaIndex retrieves the most relevant context given a user query and synthesizes a response using a response synthesizer. [Response from our Chatbot to the query 'What's LlamaIndex?']*

## 📋 How does it work?

LlamaIndex enriches LLMs (for simplicity, we default the [Settings](https://docs.llamaindex.ai/en/stable/module_guides/supporting_modules/settings/) to OpenAI GPT-3.5 which is then used for indexing and querying) with a custom knowledge base through a process called [Retrieval Augmented Generation (RAG)](https://research.ibm.com/blog/retrieval-augmented-generation-RAG) that involves the following steps:

- **Connecting to a External Datasource**: We use the [Github Repository Loader](https://llamahub.ai/l/readers/llama-index-readers-github) available at [LlamaHub](https://llamahub.ai/) (an open-source repository for data loaders) to connect to the Github repository containing the markdown files of the LlamaIndex Docs:

```python
from llama_index.readers.github import GithubRepositoryReader, GithubClient

def load_data(loader: GithubRepositoryReader) -> []:
    """Load Knowledge Base from GitHub Repository"""

    logging.info("Loading data from Github: %s/%s", loader._owner, loader._repo)
    docs = loader.load_data(branch="main")
    for doc in docs:
        logging.info(doc.extra_info)
        doc.metadata = {'filename': doc.extra_info['file_name'], 'author': "LlamaIndex"}
        
    return docs
```

- **Constructing Documents**: The markdown files of the Github repository are ingested and automatically converted to Document objects. In addition, we add the dictionary {'filename': '', 'author': ''} to the metadata of each document (which will be inhereited by the nodes). This will allow us to retrieve and display the data sources and scores in the chatbot responses to make our App more transparent:

```python
def load_data(loader: GithubRepositoryReader) -> []:
    """Load Knowledge Base from GitHub Repository"""

    logging.info("Loading data from Github: %s/%s", loader._owner, loader._repo)
    docs = loader.load_data(branch="main")
    for doc in docs:
        logging.info(doc.extra_info)
        doc.metadata = {'filename': doc.extra_info['file_name'], 'author': "LlamaIndex"}
        
    return docs
```

- **Parsing Nodes**: Nodes represent a *chunk* of a source Document, we have defined a chunk size of '1024' with an overlap of '32'. Similar to Documents, Nodes contain metadata and relationship information with other nodes.
```python
def index_data(docs: []) -> VectorStoreIndex:
    """Index Documents"""
    
    logging.info("Parsing documents into nodes...")
    parser = SimpleNodeParser.from_defaults(chunk_size=1024, chunk_overlap=32)
    nodes = parser.get_nodes_from_documents(docs)
```

- **Indexing**: An Index is a data structure that allows to quickly retrieve relevant context for a user query. For LlamaIndex, it's the core foundation for retrieval-augmented generation (RAG) use-cases. LlamaIndex provides different types of indices, such as the [VectorStoreIndex](https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_index/), which makes LLM calls to compute embeddings:

```python
    [...]

    logging.info("Indexing nodes...")
    index = VectorStoreIndex(nodes)

    logging.info("Persisting index on ./storage...")
    index.storage_context.persist(persist_dir="./storage")
        
    logging.info("Data-Knowledge ingestion process is completed (OK)")
```

- **Querying (with cache)**: Once the index is constructed, querying a vector store index involves fetching the top-k most similar Nodes (by default 2), and passing those into the Response Synthesis module. The top Nodes are then appended to the user's prompt and passed to the LLM. We rely on the [Streamlit caching mechanism](https://docs.streamlit.io/library/advanced-features/caching) to optimize the performance and reduce the number of calls to the LLM:

```python
@st.cache_data(max_entries=1024, show_spinner=False)
def query_chatengine_cache(prompt, _chat_engine, settings):
    return _chat_engine.chat(prompt)
```

- **Parsing Response**: The App parses the response source nodes to extract the filename, author and score of the top-k similar Nodes (from which the answer was retrieved):

```python
def get_metadata(response):
    sources = []
    for item in response.source_nodes:
        if hasattr(item, "metadata"):
            filename = item.metadata.get('filename').replace('\\', '/')
            author = item.metadata.get('author')
            score = float("{:.3f}".format(item.score))
            sources.append({'filename': filename, 'author': author, 'score': score})
    
    return sources
```

- **Transparent Results with Source Citation**: The use of metadata enables to display links to the sources along with the author and relevance scores from which the answer was retrieved:

<p align="center">
  <img src="./assets/sourcecitation.png">
</p>


- **Estimating Inference Cost**: By using [TokenCountingHandler](https://docs.llamaindex.ai/en/stable/examples/callbacks/TokenCountingHandler.html), the App tracks the number of 'LLM Prompt Tokens' and 'LLM Completion Tokens' to estimate the overall [GTP-3.5 inference costs](https://openai.com/pricing). 

```python
        token_counter = TokenCountingHandler(
            tokenizer=tiktoken.encoding_for_model("gpt-3.5-turbo").encode,
            verbose=False
        )
        
        Settings.llm = OpenAI(model="gpt-3.5-turbo", temperature=0.2)
        Settings.callback_manager = CallbackManager([token_counter])
        
        # Replace the deprecated ServiceContext.from_defaults with Settings
        index = load_index_from_storage(StorageContext.from_defaults(persist_dir="./storage"))
```


## 🚀 Quickstart

1. Clone the repository:
```
git clone git@github.com:goldzulu/llamaindexdocschat.git
```

2. Create and Activate a Virtual Environment:

```
Windows:

py -m venv .venv
.venv\scripts\activate

macOS/Linux

python3 -m venv .venv
source .venv/bin/activate
```

3. Install dependencies:

```
pip install -r requirements.txt
```

4. Copy .env.example to .env and set your OpenAI API Key and Github Token.


5. Ingest Knowledge Base
```
python ingest_knowledge.py
```

6. Launch Web Application

```
streamlit run ./app.py
```

## 📚 References

- [LLamaIndex Doc Reference](https://gpt-index.readthedocs.io/en/latest/index.html)
- [Get Started with Streamlit Cloud](https://docs.streamlit.io/streamlit-community-cloud/get-started)

## Inspired by

Code adapted originally @dcarpintero : https://github.com/dcarpintero/llamaindexchat

With updates and modifications by @goldzulu :
- Refreshed and upgraded to the latest Python 3.12 and LlamaIndex 0.10.56

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.