🔥 Key Areas an Interviewer May Ask About

(Along with strong, smart answers 👇)

# 🔹 1. How does your system convert a question into SQL?
✅ Smart Answer:

I use LangChain’s create_sql_query_chain() which internally uses an LLMChain with SQL-aware prompt templates. It takes in a natural language question and uses my Azure OpenAI LLM to generate SQL by understanding the connected schema from the SQLDatabase object (Chinook.db in my case).
I also enhance it by appending selected table constraints in the prompt to reduce ambiguity and control the SQL generation scope.

# 🔹 2. Why use LangChain for SQL instead of doing this manually?
✅ Smart Answer:

LangChain provides ready-made abstractions like create_sql_query_chain, schema introspection, prompt templates, and toolchains like QuerySQLDataBaseTool, saving significant boilerplate work. It also ensures the LLM gets contextual awareness of table structures, leading to fewer hallucinations and safer queries.

# 🔹 3. What happens if the LLM generates incorrect or unsafe SQL?
✅ Smart Answer:

I add constraints via prompt engineering — e.g., explicitly asking the model to use only selected tables and avoid DDL/DML statements like DROP, DELETE, etc. However, a safer production version could use a SQL parser to validate queries before execution or use SQLGuard-like sandboxing.

# 🔹 4. What if the SQL output doesn't match the schema or fails execution?
✅ Smart Answer:

The execute_query.invoke() method is wrapped in a try/except block to catch and raise HTTP exceptions. On failure, I could enhance it further by adding a feedback loop — i.e., feeding the error back to the LLM to regenerate a corrected query, or asking the user for clarification.

# 🔹 5. How are the results processed post SQL execution?
✅ Smart Answer:

Once results are fetched, I use the LLM again to generate insights and extract column names. This provides both structured output and human-readable summaries. I round numeric fields to two decimal places and ensure the column names align with result tuples before storing them in MongoDB.

# 🔹 6. Why use Azure OpenAI instead of OpenAI API?
✅ Smart Answer:

Azure OpenAI fits enterprise needs due to:
Better governance/compliance
Private endpoints & VNet support
Regional hosting
Also, my organization prefers Azure for cost tracking and integrated security.

# 🔹 7. Why store history in MongoDB?
✅ Smart Answer:

MongoDB offers schema flexibility — ideal for user-generated content like queries, answers, errors, etc. It also allows fast retrieval of a user’s past SQL questions, queries, and insights with simple filters like {id: user_id}.

# 🔹 8. How do you authenticate users and maintain session state?
✅ Smart Answer:

I use JWTs for authentication and store user sessions using FastAPI’s SessionMiddleware. For Google login, I decode the JWT from the client, register new users in MongoDB if needed, and generate my own JWT token for session validation. All protected endpoints use Depends(get_current_user).

# 🔹 9. How does the LLM know the database schema?
✅ Smart Answer:

LangChain's SQLDatabase class introspects the schema automatically and includes that in the prompt. So when the LLM generates SQL, it already has awareness of available tables and columns. It avoids guessing schema details blindly.

# 🔹 10. What are the limitations or future improvements?
✅ Smart Answer:

Some ideas:
Add query validation or SQL parser checks
Use SQL-LLM fine-tuned models for complex queries
Add explanations for SQL (e.g., "This query fetches top artists by count")
Add auto-correction if a query fails
Let users rate LLM-generated queries for feedback tuning


# 🔹 11. What happens when a user uploads a PDF?
Answer:
When a PDF is uploaded:

It's saved temporarily.
AzureAIDocumentIntelligenceLoader parses its layout/content.
MarkdownHeaderTextSplitter splits content into structured chunks.
Each chunk is embedded using AzureOpenAIEmbeddings.
The chunks and their embeddings are stored in Azure Cognitive Search (as a vector store) under an index named after the file.
# 🔹 12. Why are the documents split into chunks?
Answer:
Chunking improves retrieval quality and context window fit. It allows:

Finer semantic search granularity.
Efficient vector similarity matching.
Reduces token overflow during question-answering.
# 🔹 13. How is similarity search implemented on the uploaded PDF content?
Answer:
Using AzureSearch.as_retriever(search_type="similarity", search_kwargs={"k": 3}), we:

Embed the user’s query.
Search for the top-3 semantically closest document chunks.
Pass them as context to the RAG pipeline.
# 🔹 14. What is the role of hub.pull("rlm/rag-prompt") in the document query endpoint?
Answer:
It retrieves a predefined LangChain RAG prompt template that:

Structures input (context + question) for the LLM.
Guides how to frame answers using retrieved context.
Ensures relevance and grounded responses from the model.
# 🔹 15. How does your application prevent hallucinations in document-based answers?
Answer:
By using RAG:

Answers are generated only from retrieved, relevant chunks.
The LLM is conditioned on specific context (retrieved chunks).
Prompt engineering also reinforces grounded answering (no fabrication).
# 🔹 16. How do you ensure index uniqueness in the vector store for each document?
Answer:
The index name is dynamically derived from the filename:

index_name = file.filename.lower().replace('.', '-').replace('-pdf','').replace(' ','')
This ensures uniqueness, and get_index() is used to check for existence before recreating.

# 🔹 17. What if the document has already been indexed—how do you avoid duplication?
Answer:
The code:

index_result = documents.find_one({"index_name": index_name, "id": user_id})
prevents re-indexing by checking MongoDB. If already processed, it skips re-insertion.

# 🔹 18. Can multiple users upload the same document? How is user isolation maintained?
Answer:
Yes, users can upload identical PDFs. Isolation is maintained by:

Storing embeddings per user in MongoDB.
Including the user_id with every index record.
Access and queries are scoped per authenticated user via JWT.
# 🔹 19. What embedding model do you use and why?
Answer:
We use AzureOpenAIEmbeddings, which:

Leverages OpenAI’s text-embedding-ada-002 under Azure.
Offers robust semantic understanding.
Integrates natively with AzureSearch for vector indexing.
# 🔹 20. Why did you choose Azure Cognitive Search as the vector store?
Answer:
Because:

It integrates well with enterprise Azure environments.
Scales vector search efficiently.
Supports hybrid search (text + semantic).
Has tight support in LangChain.
# 🔹 21. Can you walk me through the RAG chain used for document Q&A?
Answer:
Yes. The RAG pipeline:

Retrieves relevant chunks via similarity search.
Passes them into a prompt (retrieved from LangChain Hub).
Invokes AzureChatOpenAI to generate the final answer.
Parses it with StrOutputParser() and returns it to the user.
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | doc_llm
    | StrOutputParser()
)
# 🔹 22. How do you handle document history and traceability?
Answer:
We store every question and answer in the documents_history collection in MongoDB, along with:

user_id
filename
question
answer
This allows auditability and history tracking per user.
