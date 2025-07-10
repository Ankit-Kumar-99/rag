# ✅ End-to-End Flow: PDF Upload + Question Answering with RAG
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/a591ca08-6f72-4641-ac63-ff2334e67f6e" />

# 🔁 Step-by-Step Breakdown:

## 1. Upload PDF

Endpoint:

@app.post("/upload/")
What happens:

Reads uploaded PDF file
Saves it locally using:
with open(file_path, 'wb') as f:
    f.write(contents)
Loads document using AzureAIDocumentIntelligenceLoader:
loader = AzureAIDocumentIntelligenceLoader(file_path=..., api_key=..., ...)
docs = loader.load()
🔧 Model used: prebuilt-layout — for extracting structured content from PDF

## 2. Split Document into Chunks

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]
text_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
splits = text_splitter.split_text(docs_string)
Why?
To make it retrievable later — LLMs perform better on smaller, logically separated chunks.

## 3. Store Embeddings in Azure Search Vector DB

vector_store = AzureSearch(
    azure_search_endpoint=...,
    azure_search_key=...,
    index_name=index_name,
    embedding_function=aoai_embeddings.embed_query
)
vector_store.add_documents(documents=splits)
A new index is created if not already present
Embeddings are created using Azure OpenAI Embedding model
Stored in Azure Cognitive Search (vector store)
## 4. Store Metadata in MongoDB

documents.insert_one({"id": user_id, "index_name": index_name, "filename": file.filename})
Used later to retrieve document for querying

✅ Now Comes the Querying Part
## 5. Ask a Question on Uploaded PDF

Endpoint:

@app.post("/document/")
request = {
  "question": "What are the key findings in the executive summary?",
  "filename": "sample_report.pdf"
}
## 6. Retrieve Index and Load VectorStore

index_name = documents.find_one(...).get("index_name")
vector_store = AzureSearch(...)
retriever = vector_store.as_retriever(search_type="similarity", search_kwargs={"k": 3})
Finds matching chunks from the document using similarity search (top 3 most relevant)
## 7. Construct RAG Chain for QA

prompt = hub.pull("rlm/rag-prompt")  # From LangChain hub

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | doc_llm  # AzureChatOpenAI
    | StrOutputParser()
)
answer = rag_chain.invoke(request.question)
The prompt template formats the chunks + question
AzureChatOpenAI is used to generate an answer from retrieved chunks
## 8. Store Answer in MongoDB

documents_history.insert_one({
  "id": user_id,
  "question": request.question,
  "answer": answer,
  "index_name": index_name
})
✅ Final Output Returned:

{
  "question": "What are the key findings in the executive summary?",
  "answer": "The key findings include increased revenue YoY, higher customer satisfaction, and improved delivery timelines..."
}
🧠 Summary of Tech Components Used

Component	Purpose
FastAPI	API framework
Azure AI Document Intelligence	Document parsing
LangChain	Text splitting, chaining, LLM orchestration
AzureSearch	Vector store for similarity search
Azure OpenAI	Embeddings + LLM for response
MongoDB	Metadata & history storage
🧪 Sample Prompt Template (LangChain Hub)

Use the following context to answer the question. If the answer is not in the context, say "I don’t know".

Context: 
{{ context }}

Question: 
{{ question }}

Answer:
