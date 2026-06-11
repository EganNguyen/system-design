# ML/AI — Requirements & Best Techniques

| #  | Requirement                      | Best Technique                                        |
|----|----------------------------------|-------------------------------------------------------|
| 1  | Binary classification            | Logistic Regression / XGBoost                         |
| 2  | Multi-class classification       | XGBoost / Neural Network                              |
| 3  | Regression prediction            | XGBoost / Random Forest                               |
| 4  | Regression at scale (large data) | LightGBM                                              |
| 5  | Fraud detection                  | Gradient Boosting + Anomaly Detection                 |
| 6  | Recommendation system            | Collaborative Filtering (small) / Two-Tower (prod)    |
| 7  | Similarity search                | Embeddings + Vector Database                          |
| 8  | Approximate nearest neighbor     | ANN (HNSW / IVF Index)                               |
| 9  | Semantic search                  | RAG + Vector Search                                   |
| 10 | Question answering               | RAG                                                   |
| 11 | Enterprise chatbot               | RAG + LLM                                             |
| 12 | Multi-agent orchestration        | Agent Framework (LangGraph / CrewAI)                  |
| 13 | Knowledge management             | RAG + Knowledge Graph                                 |
| 14 | Document understanding           | OCR + LLM                                             |
| 15 | Contract analysis                | LLM + RAG                                             |
| 16 | Summarization                    | LLM                                                   |
| 17 | Translation                      | Transformer Models                                    |
| 18 | Sentiment analysis               | Fine-tuned BERT                                       |
| 19 | Named entity recognition         | BERT / spaCy                                          |
| 20 | Customer support automation      | LLM Agent                                             |
| 21 | Time-series forecasting          | Prophet / XGBoost / LSTM                              |
| 22 | Time-series anomaly detection    | Isolation Forest / Autoencoder                        |
| 23 | Predictive maintenance           | Time-Series ML                                        |
| 24 | Demand forecasting               | XGBoost + Time Features                               |
| 25 | Churn prediction                 | XGBoost                                               |
| 26 | Lead scoring                     | Gradient Boosting                                     |
| 27 | Dynamic pricing                  | Reinforcement Learning                                |
| 28 | Route optimization               | Reinforcement Learning + OR                           |
| 29 | Image classification             | CNN / Vision Transformer                              |
| 30 | Object detection                 | YOLO                                                  |
| 31 | Image segmentation               | U-Net / SAM                                           |
| 32 | Face recognition                 | Embeddings                                            |
| 33 | Multimodal understanding         | Vision-Language Model (GPT-4V / LLaVA)               |
| 34 | OCR                              | Transformer OCR                                       |
| 35 | Speech-to-text                   | Whisper                                               |
| 36 | Text-to-speech                   | Neural TTS                                            |
| 37 | Code generation                  | LLM                                                   |
| 38 | SQL generation                   | Text-to-SQL + RAG                                     |
| 39 | Workflow automation              | AI Agents                                             |
| 40 | Multi-step reasoning             | Agentic AI + Tool Calling                             |
| 41 | Personal AI assistant            | Agent + Memory + RAG                                  |
| 42 | Enterprise search                | Vector DB + RAG                                       |
| 43 | Billion-document retrieval       | Hybrid Search (BM25 + Vector)                         |
| 44 | Real-time recommendations        | Streaming Features + Online Inference                 |
| 45 | Feature management               | Feature Store                                         |
| 46 | Model deployment                 | MLOps Platform                                        |
| 47 | Continuous training              | Automated Pipelines                                   |
| 48 | Model monitoring                 | Drift Detection                                       |
| 49 | A/B testing ML models            | Shadow Deployment / Canary Deployment                 |
| 50 | Low-latency inference            | ONNX / TensorRT                                       |
| 51 | Edge AI                          | Quantized Models                                      |
| 52 | Cost optimization                | Distillation                                          |
| 53 | Small-device AI                  | TinyML                                                |
| 54 | Guardrails / AI safety           | LLM-as-Judge + Rule Filters                           |
| 55 | Enterprise AI platform           | MLOps + LLMOps                                        |

---

# Enterprise AI Architecture Patterns

| Requirement                   | Architecture                   |
|-------------------------------|--------------------------------|
| ChatGPT for company documents | RAG                            |
| ChatGPT with actions          | Agent                          |
| AI with database access       | Agent + SQL Tool               |
| AI with ERP/CRM integration   | Agent + Function Calling       |
| AI software engineer          | Agent + Code Execution         |
| AI customer support           | Agent + RAG                    |
| AI business analyst           | Agent + BI Tools               |
| AI research assistant         | Deep Research Agent            |
| AI call center                | Speech + LLM + Agent           |
| AI fraud platform             | Streaming ML + Feature Store   |
| Multi-agent system            | Agent Framework (LangGraph / CrewAI) |

---

# Open Source Technologies Commonly Used by Big Tech

### Data & ML Platform
- [Apache Spark](https://spark.apache.org)
- [Apache Kafka](https://kafka.apache.org)
- [Apache Airflow](https://airflow.apache.org)
- [Kubeflow](https://www.kubeflow.org)

### MLOps
- [MLflow](https://mlflow.org)
- [Feast](https://feast.dev)
- [KServe](https://kserve.github.io/website/)

### LLMOps
- [LangChain](https://www.langchain.com)
- [LlamaIndex](https://www.llamaindex.ai)
- [Haystack](https://haystack.deepset.ai)
- [LangGraph](https://www.langchain.com/langgraph)

### Vector Databases
- [Qdrant](https://qdrant.tech)
- [Weaviate](https://weaviate.io)
- [Milvus](https://milvus.io)

### Deep Learning
- [PyTorch](https://pytorch.org)
- [TensorFlow](https://www.tensorflow.org)

---

# AI Engineer Interview Cheat Sheet

| Problem                    | Usually Use                        |
|----------------------------|------------------------------------|
| Tabular business data      | XGBoost                            |
| Large-scale tabular data   | LightGBM                           |
| Fraud detection            | XGBoost + Feature Engineering      |
| Search company documents   | RAG                                |
| Enterprise chatbot         | RAG + LLM                          |
| Need external actions      | Agent                              |
| Need memory                | Agent + Memory                     |
| Need reasoning + tools     | Agentic AI + Tool Calling          |
| Need multiple AI agents    | Agent Framework (LangGraph / CrewAI)|
| Similarity matching        | Embeddings                         |
| Recommender system         | Two-Tower Model                    |
| Billion-scale retrieval    | Hybrid Search                      |
| AI safety / guardrails     | LLM-as-Judge + Rule Filters        |
| Production ML platform     | MLOps                              |
| Production LLM platform    | LLMOps                             |

---

## Practical Decision Rules

> **Structured data (tables)** → Start with **XGBoost** (or LightGBM at scale)
>
> **Unstructured data (documents, emails, PDFs)** → Start with **RAG**
>
> **AI must take actions** (call APIs, update systems, execute workflows) → Use **Agents**
>
> **AI must coordinate multiple agents** → Use an **Agent Framework** (LangGraph / CrewAI)
>
> **AI must be safe and controlled in production** → Add **Guardrails** (LLM-as-Judge + Rule Filters)
>
> **AI must continuously learn and operate at scale** → Invest in **MLOps / LLMOps**
