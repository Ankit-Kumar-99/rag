# rag


🔍 High-Level Overview

Your app is a multi-modal RAG (Retrieval Augmented Generation) system built with FastAPI, integrating:

User Authentication (via Google OAuth and JWT)
SQL-based QA using LangChain + Azure OpenAI
Document-based RAG via Azure Document Intelligence + Azure Cognitive Search
Audio RAG using AssemblyAI for transcription + ChromaDB for retrieval
Video RAG by extracting audio → transcribing → using ChromaDB
History tracking using MongoDB
📦 Key Components Breakdown

✅ Authentication
Uses JWT (HS256) with Google Sign-in (/googlelogin)
Sessions handled with SessionMiddleware
MongoDB stores users
Token-based authorization is applied via Depends(get_current_user)
🗃️ SQL Querying with LLM
Database: Chinook.db
LLM: AzureOpenAI
Uses:
create_sql_query_chain() to generate queries
QuerySQLDataBaseTool to execute queries
After query execution:
Results are processed, rounded
Insights and column names are generated using LLM
All stored in MongoDB (sql_history)
📄 Document RAG Flow
Upload
Endpoint: /upload/
File is read → passed to AzureAIDocumentIntelligenceLoader
Text is split with MarkdownHeaderTextSplitter
Vectorization
Embedding: AzureOpenAIEmbeddings
Stored in Azure Cognitive Search (AzureSearch)
Index management done via SearchIndexClient
Query
Endpoint: /document/
LangChain RAG chain: prompt (from LangChain hub) + retriever + LLM
Stores answer in documents_history
🔊 Audio RAG
Upload
Endpoint: /uploadAudio
Uses AssemblyAIAudioTranscriptLoader for transcription
Split using RecursiveCharacterTextSplitter
Stored in audios collection
Query
Uses Chroma vectorstore (in-memory)
RetrievalQA with stuff chain type
Result and transcript returned
Stored in audio_history
🎥 Video RAG
Upload
Endpoint: /uploadVideo
Uses moviepy to convert to .mp3
Transcript generation same as audio
Query
Similar to audio: ChromaDB + RetrievalQA
Stored in video_history
🧱 Storage Used

Type	Technology Used	Purpose
SQL DB	SQLite (Chinook.db)	Source for SQL queries
Vector DB	Azure Cognitive Search	Document embeddings & retrieval
Vector DB	Chroma	Audio/video embeddings
NoSQL DB	MongoDB	Users + all history collections
LLM	Azure OpenAI	Natural language processing
OCR	Azure Document Intelligence	Parsing PDF & other docs
ASR	AssemblyAI	Transcribing audio/video
🔁 Endpoints Summary

Route	Method	Description
/googlelogin/	POST	JWT-based Google login
/query/	POST	SQL query with natural language
/tables/	GET	List all SQL tables
/upload/	POST	Upload document & store vectors
/document/	POST	Ask question on a specific document
/uploadAudio	POST	Upload & process audio file
/audio	POST	RAG-based audio Q&A
/uploadVideo	POST	Upload & process video file
/video	POST	RAG-based video Q&A
/history/, /documentHistory/, /audioHistory/, /videoHistory/	GET	Get respective histories
/get-user	GET	Get current user details
🧠 LangChain Usage

Use Case	Component Used
SQL	create_sql_query_chain, QuerySQLDataBaseTool
Document RAG	AzureSearch, AzureAIDocumentIntelligenceLoader
Audio/Video RAG	Chroma, RetrievalQA
Prompting	LangChain Hub (rlm/rag-prompt)
LLM	AzureChatOpenAI, AzureOpenAI
Caching	SQLiteCache, optional Redis commented
