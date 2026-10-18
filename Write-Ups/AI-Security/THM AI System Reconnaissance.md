## Introduction

When organisations add AI capabilities, their networks change in ways that most security teams never see. New services appear on unfamiliar ports. APIs respond in formats that standard scanners do not recognise. ML platforms, model registries, and experiment-tracking servers are deployed with default configurations that prioritise ease of use over security.

### What is AI Reconnaissance

AI reconnaissance is the practice of identifying these components, understanding what they are, and what they expose. You still use tools like Nmap and curl, but the targets are different. Standard scanning wordlists and service detection miss AI infrastructure entirely because they were not built to find it.

### Why This Matters Right Now

In January 2026, security researcher Maor Dayan ran a targeted Shodan scan and found [42,665 exposed AI agent instances](https://www.clawctl.com/blog/42665-exposed-agents-what-we-learned) on the public internet. 93.4% were vulnerable, and many were leaking API keys through unauthenticated connections. Around the same time, GreyNoise captured over 91,000 attack sessions targeting AI deployments in three months. Attackers are already scanning for AI infrastructure at scale.
### Scenario

I am a security engineer at Cyphira, a mid-size fintech company that deployed several AI-powered features over the past year without a security review. Your CISO read a threat report about attackers hijacking exposed AI infrastructure and wants answers: which AI systems do we have, what they expose, and how an attacker could find them?

## The AI Infrastructure Stack

When you scan a traditional network, you know what to expect. Web servers on ports 80 and 443. SSH on 22. Databases on 3306 or 5432. You have seen these a hundred times. AI infrastructure looks nothing like that. It runs on ports most security professionals have never targeted, speaks protocols that standard HTTP scanners misinterpret, and exposes APIs that return data formats you will not find in any conventional web application.

A production AI system is not a single server. It is a collection of specialised services that handle different parts of the machine learning lifecycle. Here is what you will encounter during reconnaissance.

#### Model Serving Endpoints

These are the frameworks that load trained models into memory and expose prediction APIs. They are the front door of any AI deployment.

- NVIDIA Triton Inference Server is one of the most common. It exposes HTTP on port 8000, gRPC on port 8001, and Prometheus metrics on port 8002. Three ports for one service.
- TensorFlow Serving uses gRPC on 8500 and HTTP on 8501.
- TorchServe exposes inference on 8080, a management API on 8081, and metrics on 8082.
- For LLM-specific deployments, Ollama runs on port 11434, and vLLM typically serves on port 8000. Both expose OpenAI-compatible API formats.

Why do many of these expose both HTTP and gRPC at the same time? gRPC is faster for streaming large tensor payloads between internal services. HTTP/REST is easier for client applications to consume. So they run both.

#### Orchestration and Experiment Tracking

These platforms manage the entire ML lifecycle, from experiment design through to model deployment. They are the **highest-value targets** during reconnaissance because they contain everything.

- MLflow Tracking Server runs on port 5000.
- It records every experiment, every hyperparameter configuration, every training metric, and every model artifact the organisation produces.
- If you find an open MLflow instance, you have found their complete ML research history.
- Kubeflow runs on Kubernetes (ports 80/443) and orchestrates training pipelines, notebook servers, and model deployments.
- Ray exposes a dashboard on port 8265 and a server endpoint on port 8000.

The ShadowRay campaign we mentioned in Task 1 targeted exactly this dashboard.

#### Vector Databases

These store embeddings (numerical representations of documents) and power semantic search for RAG pipelines. If an organisation has an AI chatbot or knowledge assistant, there is almost certainly a vector database behind it.

- Qdrant runs on port 6333 for HTTP and 6334 for gRPC.
- Weaviate runs on port 8080 and includes a GraphQL endpoint. Milvus uses port 19530.
- Chroma runs on port 8000.

What makes these interesting from a recon perspective is that their schema endpoints reveal which embedding model the system uses and what kind of data it contains. A collection called "internal-hr-policies" with 768-dimensional vectors tells you quite a lot about what that RAG system is doing.

#### Model Registries

A model registry stores the actual model files. Serialised **.pkl, .pt, .onnx**, and **.mar** files, along with version history, stage transitions (staging, production, archived), artifact URIs pointing to cloud storage, and the identity of who created each version. An unsecured registry is the single highest-value reconnaissance target. It maps the organisation's entire ML product portfolio in one place.

#### Supporting Infrastructure

- Jupyter notebooks (port 8888) often run with `--ip=0.0.0.0` and no authentication. That **gives direct terminal access to anyone who reaches the port**. Data scientists also routinely store cleartext credentials in notebook cells.
- MinIO (ports 9000 and 9001) provides S3-compatible object storage for model artifacts. Bucket listing is frequently enabled.
- Prometheus metrics endpoints on model servers (Triton on 8002, TorchServe on 8082) leak model names, batch sizes, GPU utilisation, and inference latency. You can map the entire deployment topology without ever touching the inference API.

### Port and Protocol Reference

Here is the reference table you will use for the rest of this room. Keep it handy.

|**Component**|**Default Port(s)**|**Protocol(s)**|**Recon Endpoints**|
|---|---|---|---|
|NVIDIA Triton  <br>_(Loads models into memory and serves predictions at scale)_|8000, 8001, 8002|HTTP, gRPC, Prometheus|/v2/health/ready, /v2/models|
|TensorFlow Serving  <br>_(Google's model serving framework for TensorFlow models)_|8500, 8501|gRPC, HTTP|/v1/models/<name>|
|TorchServe  <br>(_PyTorch's official model serving framework_)|8080, 8081, 8082|HTTP|/ping, /models|
|Ollama  <br>(_Local runtime for running LLMs on your own hardware_)|11434|HTTP|/api/tags, /api/show|
|vLLM  <br>_(High-throughput LLM serving engine with OpenAI-compatible API)_|8000|HTTP|/v1/models|
|MLflow  <br>_(Tracks experiments, stores models, and manages the ML lifecycle)_|5000|HTTP|/api/2.0/mlflow/experiments/search|
|Kubeflow  <br>_(Kubernetes-native platform for orchestrating ML pipelines)_|80, 443|HTTP|/pipeline/apis/v1beta1/pipelines|
|Ray  <br>_(Distributed compute framework for scaling AI workloads)_|8265, 8000|HTTP|/api/jobs/, Ray Dashboard|
|Qdrant  <br>(_Vector database for semantic search and RAG pipelines_)|6333, 6334|HTTP, gRPC|/collections|
|Weaviate  <br>_(Vector database with built-in GraphQL and module system)_|8080|HTTP, GraphQL|/v1/schema, /v1/meta|
|Milvus  <br>(_Distributed vector database for large-scale embedding storage_)|19530|gRPC|Port 19530 connection|
|Jupyter Notebook  <br>_(Interactive coding environment used by data scientists)_|8888|HTTP|/api/kernels, /api/contents|
|MinIO  <br>_(S3-compatible object storage often used for model artifacts)_|9000, 9001|HTTP (S3-compatible)|Bucket listing|
|Prometheus metrics  <br>_(Not a standalone AI service; exposed by Triton on 8002, TorchServe on 8082, and other ML servers as a built-in monitoring endpoint)_|8002, 8082|HTTP|/metrics|

That is 14 components across 20+ ports. Compare that to a traditional web application, which might add 3 to 5 ports to a network. Adding an AI stack roughly triples the attack surface at the network layer alone.

### Case Study: What Exposed AI Infrastructure Actually Looks Like

In the introduction, we mentioned that a January 2026 Shodan scan found 42,665 exposed AI agent instances. Let's break down what those instances actually were, because the number alone does not tell you much. The detail does. The most commonly exposed services included MLflow dashboards sitting on port 5000 with no authentication. Unauthenticated Jupyter notebooks on port 8888, some already compromised with cryptomining malware (Monero miners like peer2profit), deployed by earlier attackers who found them first. Open Ray dashboards on port 8265. Triton inference endpoints on port 8000.

These were not isolated findings. They were connected systems. IBM X-Force documented this pattern in their [2025 research on ML training infrastructure attacks](https://www.ibm.com/think/x-force/becoming-the-trainer-attacking-ml-training-infrastructure). In one case, they mapped a complete chain: a Jupyter notebook running on Azure ML contained cleartext MLflow credentials in a notebook cell. Those credentials pointed to an MLflow tracking server. The MLflow server's model registry contained artifact URIs pointing to S3 storage buckets holding the actual model files. Every component trusted the network boundary, so compromising one compromised the rest.

The Shodan dorks that found these services were not complicated:

- `port:5000 "MLflow"` for exposed MLflow instances
- `port:8888 title:"Home Page - Select or create a notebook"` for unauthenticated Jupyter notebooks
- `http.title:"Ray Dashboard"` for exposed Ray cluster dashboards
- `port:8001 "triton"` for Triton gRPC endpoints

Simple queries. Massive results. That is why the port reference table above matters. If you know the ports and the service banners, finding AI infrastructure is not hard. The hard part is knowing to look in the first place.

### Q&A

We will be starting the Cyphira AI udit. The internal subnet is 10.10.45.0/24. We have to find every AI component on this network.

<img width="953" height="863" alt="image" src="https://github.com/user-attachments/assets/5314c1d5-0c3e-483b-9e9c-9924075363cc" />

```
Starting Nmap 7.94SVN ( https://nmap.org ) at [date]
Nmap scan report for web-prod.cyphira.internal (10.10.45.10)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp closed
6334/tcp closed
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp closed
9000/tcp closed
9001/tcp closed

Nmap scan report for db-prod.cyphira.internal (10.10.45.11)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp closed
6334/tcp closed
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp closed
9000/tcp closed
9001/tcp closed

Nmap scan report for mlflow-tracker.cyphira.internal (10.10.45.12)
Host is up (X.XXs latency).
5000/tcp open http Werkzeug/3.0.1 Python/3.11.7
6333/tcp closed
6334/tcp closed
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp closed
9000/tcp closed
9001/tcp closed

Nmap scan report for triton-inference.cyphira.internal (10.10.45.15)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp closed
6334/tcp closed
8000/tcp open http
8001/tcp open grpc
8002/tcp open http Prometheus metrics
8888/tcp closed
9000/tcp closed
9001/tcp closed

Nmap scan report for qdrant-vectordb.cyphira.internal (10.10.45.18)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp open http
6334/tcp open grpc
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp closed
9000/tcp closed
9001/tcp closed

Nmap scan report for jupyter-ds.cyphira.internal (10.10.45.20)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp closed
6334/tcp closed
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp open http Tornado/6.4
9000/tcp closed
9001/tcp closed

Nmap scan report for minio-store.cyphira.internal (10.10.45.22)
Host is up (X.XXs latency).
5000/tcp closed
6333/tcp closed
6334/tcp closed
8000/tcp closed
8001/tcp closed
8002/tcp closed
8888/tcp closed
9000/tcp open http MinIO
9001/tcp open http MinIO Console

Nmap done: 256 IP addresses (7 hosts up) scanned in X.XX seconds
Service detection performed.
```

<img width="958" height="872" alt="image" src="https://github.com/user-attachments/assets/f364665b-de7d-42ed-8984-34c9c39c452d" />

```
Starting Nmap 7.94SVN ( https://nmap.org ) at [date]
Nmap scan report for web-prod.cyphira.internal (10.10.45.10)
Host is up (X.XXs latency).
22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
80/tcp open http nginx 1.24.0
443/tcp open ssl/http nginx 1.24.0
5432/tcp closed

Nmap scan report for db-prod.cyphira.internal (10.10.45.11)
Host is up (X.XXs latency).
22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
80/tcp closed
443/tcp closed
5432/tcp open postgresql PostgreSQL 15.2

Nmap scan report for mlflow-tracker.cyphira.internal (10.10.45.12)
Host is up (X.XXs latency).
22/tcp closed
80/tcp closed
443/tcp closed
5432/tcp closed

Nmap scan report for triton-inference.cyphira.internal (10.10.45.15)
Host is up (X.XXs latency).
22/tcp closed
80/tcp closed
443/tcp closed
5432/tcp closed

Nmap scan report for qdrant-vectordb.cyphira.internal (10.10.45.18)
Host is up (X.XXs latency).
22/tcp closed
80/tcp closed
443/tcp closed
5432/tcp closed

Nmap scan report for jupyter-ds.cyphira.internal (10.10.45.20)
Host is up (X.XXs latency).
22/tcp closed
80/tcp closed
443/tcp closed
5432/tcp closed

Nmap scan report for minio-store.cyphira.internal (10.10.45.22)
Host is up (X.XXs latency).
22/tcp closed
80/tcp closed
443/tcp closed
5432/tcp closed

Nmap done: 256 IP addresses (7 hosts up) scanned in X.XX seconds
Service detection performed.
Standard services found. But have you checked the full AI port range from your reference table?
```
What is the IP address of the host running an HTTP service on port 8888 in your scan results?

`10.10.45.20`

Which port does MLflow Tracking Server run on by default?

`5000`

## Fingerprinting AI Services

Now we need to figure out what is actually running behind each open port. This is where **fingerprinting** comes in. Standard service detection (`Nmap -sV`) often mislabels AI services. It sees HTTP on port `8000` and calls it `"http-alt"` It sees gRPC on port 8001 and does not know what to do with it. If you rely on the default Nmap output alone, you will miss what these services actually are.

Fingerprinting AI services requires a different approach; you need to look at:

- HTTP headers
- JSON response structures
- Error messages
- Endpoint naming conventions

Each AI framework has its own distinct signature, and once you know what to look for, identification becomes quick and reliable.

### HTTP Header Fingerprinting

Response headers are often the fastest way to identify an AI framework. Many inference engines broadcast their identity unless an administrator specifically masks them behind a reverse proxy.

- **TorchServe** returns a `Server: TorchServe/0.x.x` header.
    - That is a direct identification with no ambiguity.
- **Triton Inference Server** includes a `NV-Status` header in its responses.
    - It also responds to a specific request header: if you send `endpoint-load-metrics-format: text`, triton returns hardware telemetry (CPU and GPU utilisation numbers) directly in the response headers.
    - No other framework does this: if you see GPU utilisation data in an HTTP header, you are talking to Triton.
- **FastAPI-based ML** services show `server: uvicorn` in the response.
    - When you combine that with routes like `/predict` or `/embeddings`, you have a strong indicator of a Python ML backend.
- OpenAI-compatible wrappers (vLLM, LiteLLM, Ollama) return `x-request-id` headers and structured JSON with an `"object": "model"` field on their `/v1/models` endpoint.

### API Response Signatures

Each framework returns distinctly structured JSON. Once you have seen these a few times, you can identify the framework from the response body alone.

- TensorFlow Serving returns: `{"model_version_status": [{"version": "1", "state": "AVAILABLE"}]}`
- Triton returns: `{"name": "fraud_detector", "versions": ["1"], "platform": "tensorflow_graphdef"}`
- MLflow error responses include stack traces referencing mlflow.server and mlflow.tracking namespaces.  
    - Even the error tells you what you are talking to.
- **OpenAI-compatible endpoints** return: `{"object": "model", "id": "llama-3.1-8b", "created": 1700000000}`

If you hit an endpoint and get back JSON containing `"object"` : `"model"`, you are talking to something that speaks the OpenAI API format. That narrows it down to vLLM, LiteLLM, Ollama, or a custom wrapper.

### Error Message Fingerprinting

This is one of the most reliable identification techniques, and it works because AI inference APIs are rigid. They expect specific tensor shapes and data types. Send something wrong, and they complain in ways that reveal exactly what they are.

The technique is simple: send a deliberately malformed payload and read the error response.

- Send a flat list of integers to a **TensorFlow Serving endpoint** that expects a complex tensor object, and you get back an error mentioning `tensorinfo_map`. That string only appears in TF Serving errors.
- Send a bad request to an **MLflow server,** and the stack trace references `mlflow.server`, `mlflow.tracking`, or `databricks` namespaces.
- **MLflow path traversal errors** (CVE-2024-1558) go further, exposing full server filesystem paths.
- **Databricks Mosaic AI** returns Java exceptions  `io.jsonwebtoken.IncorrectClaimException` for malformed tokens. That is an instant identifier.

The reason this technique works so well is that AI frameworks produce errors that are much more verbose than those in traditional web applications. Data scientists need detailed debugging information during model development. Those debug-friendly defaults rarely get turned off before production deployment.

### Endpoint Naming Conventions

Traditional REST APIs use resource nouns: `/users`, `/accounts`, `/products`. AI endpoints use computational action terms that stand out immediately during directory brute-forcing.

- Inference endpoints: `/predict`, `/invocations` (the SageMaker convention), `/infer`, `/generate`, `/embeddings`, `/score`
- Model management: `/v1/models`, `/v2/models`
- MLflow internal API: `/api/2.0/mlflow/` (this prefix is distinctive and does not appear in any other framework)
- Kubeflow pipelines: `/pipeline/apis/v1beta1/`

If you are running ffuf or feroxbuster against a target, add these paths to your wordlist. Standard wordlists from SecLists will not include them.

### gRPC Fingerprinting

Many AI services expose gRPC alongside HTTP.

- **Triton** uses gRPC on **port 8001**.
- **TensorFlow Serving** uses it on **port 8500**.

Traditional HTTP scanners miss these entirely because gRPC is a binary protocol that does not respond to standard HTTP probes.

The tool for this is `grpcurl`. If gRPC reflection is enabled (common in development environments that get pushed to production without hardening), you can dump the entire protobuf schema:

- `grpcurl -plaintext target:8001 list`
- `grpcurl -plaintext target:8001 describe inference.GRPCInferenceService`

That output reveals all available RPCs, the exact structure of every input tensor, and the format of every output. Think of it as the gRPC equivalent of finding a `/openapi.json` file on a REST API.

### TLS Fingerprinting (JA3/JA4)

AI deployments have distinctive TLS signatures because internal service-to-service traffic is dominated by Python libraries (requests, urllib, gRPC implementations) rather than web browsers. JA3 and JA4 hash analysis can differentiate automated ML pipeline traffic from human web browsing at the network level.

The GreyNoise research we will cover in the case study below found that 99% of the attack traffic in one campaign shared the same JA4H signature, despite originating from 62 different IP addresses across 27 countries. That kind of uniformity points to shared automation tooling, and it is exactly what JA4 fingerprinting is designed to catch.

### Case Study: GreyNoise LLM Endpoint Reconnaissance

_Between October 2025 and January 2026, GreyNoise captured over 91,000 attack sessions targeting AI deployments through their Ollama honeypot infrastructure. One campaign stood out. Over 11 days starting December 28, attackers generated 80,000+ requests probing conversational endpoints to determine whether each target was compatible with OpenAI, Gemini, or Anthropic API schemas. They targeted over 73 different LLM model endpoints, including GPT-4o variants, Claude models, Llama, Gemini, DeepSeek, Mistral, Qwen, and Grok._

_The fingerprinting technique was simple and effective. The attackers sent innocuous prompts: "hi" appeared 32,716 times. "How many states are there in the United States?" appeared 27,778 times. "How many letter 'r' are in the word strawberry?" was another common test. These prompts were not random. Each one produces a response that differs in structure, format, and content depending on which model is running behind the endpoint. By comparing the response format to known API schemas, the attackers could identify the exact model and proxy configuration without triggering any content moderation filters._

GreyNoise traced the IPs and found overlap with infrastructure previously used to exploit over 200 known CVEs, including React2Shell (CVE-2025-55182). Their assessment: _"Professional threat actor conducting reconnaissance. The infrastructure overlap with established CVE-scanning operations suggests that this enumeration feeds into a larger exploitation pipeline. They're building target lists."_

This is AI fingerprinting at scale. No exploitation. No payloads. Just systematic identification of what is deployed and how it responds.

### Q&A

We will now fingerprint the services that were previously discovered, using the Cyphira Service Prober agent. 

<img width="957" height="875" alt="image" src="https://github.com/user-attachments/assets/4ac98f34-7639-460c-a367-1caa0cdc0d28" />

<img width="954" height="664" alt="image" src="https://github.com/user-attachments/assets/d45196c8-4404-45af-953f-eb4d9b8e4333" />

<img width="945" height="245" alt="image" src="https://github.com/user-attachments/assets/8647a91e-1709-49f5-bc2d-f7330f28921f" />

Which unique HTTP response header does the service on 10.10.45.15:8000 return to identify as an NVIDIA product?

`NV-Status`

When you run `grpcurl` against 10.10.45.15:8001, what is the name of the inference service listed in the reflection output?

`inference.GRPCInferenceService`

## Enumerating AI Systems

This is the part of reconnaissance where findings start getting serious.

- **Fingerprinting** tells you "that is an MLflow server."
- **Enumeration** tells you "that MLflow server contains 4 experiments, 3 production models, artifact URIs pointing to s3://cyphira-ml-models/, and the models were created by J.Chen(AI Enginner) at Cyphira"

One is identification. The other is intelligence.

### MLflow Enumeration

MLflow is the most rewarding service to enumerate because it stores everything in one place and exposes it through a clean REST API. If you find an open MLflow instance, here is the chain you follow.

**Step 1: List all experiments.**

- `` `POST /api/2.0/mlflow/experiments/search` ``

The response returns every experiment on the server with its name and internal ID. Experiment names often reveal project codenames and business objectives: `fraud-detection-v3`, `rag-embeddings-tuning`, `customer-churn-prototype`.

**Step 2: List registered models.**

- `GET /api/2.0/mlflow/registered-models/list`

This returns the full model inventory. Every model the organisation has built and registered. Names, descriptions, and creation timestamps.

**Step 3: Get model version details.**

- `` `GET /api/2.0/mlflow/model-versions/search` ``

This is where it gets serious. The response includes the `source` field, which contains the artifact URI. That URI frequently points to internal cloud storage: `s3://internal-ml-models-corp/experiments/1/artifacts/` or a similar path. It also includes the `user_id` of the data scientist who created each version, creation timestamps, and stage labels showing which models are in production versus staging.

**Step 4: Search training runs.**

- `POST /api/2.0/mlflow/runs/search`

Submit this with an empty body or a filter, and you get back hyperparameters, training metrics, and custom tags. Tags are particularly valuable. Teams use them for internal project codenames, Git commit hashes that link back to proprietary source code, and deployment environment identifiers.

**Step 5: List downloadable artifacts.**

- `` `GET /api/2.0/mlflow/artifacts/list` ``

This lists the actual model files available for download. At this point, you have mapped the organisation's entire ML portfolio through five API calls.

### Inference Server Metadata

**Triton** and **TensorFlow Serving** both expose metadata endpoints that tell you exactly how to talk to the model.

- On Triton `GET /v2/models/<name>/config` the response includes input tensor names, their multi-dimensional shapes, accepted data types (FP32, UINT64, INT8), the maximum batch size, and the backend framework (tensorflow_graphdef, pytorch_libtorch, onnxruntime).
    - This is the equivalent of getting a complete database schema. It tells an attacker exactly how to format a valid inference request.
- On TensorFlow Serving `GET /v1/models/<name>/metadata` returns the same kind of input/output tensor specifications, shape, dtype, and name for every input and output the model expects.

### Vector Database Enumeration

Vector databases reveal what data the AI system is working with and which embedding model processes it.

- On **Weaviate**, `GET /v1/meta` returns the server version and installed modules.
    - `GET /v1/schema` returns every class definition, including property names and the vectoriser module configuration.
    - That vectoriser field tells you which embedding model the system uses.
    - Weaviate also exposes `/v1/graphql` for full schema introspection and data querying on unauthenticated instances.
- On **Qdrant**, `GET /collections` it lists all collection names.
    - `GET /collections/<name>` returns vector dimensions, the distance metric, and the total point count.
    - A collection named `internal-hr-policies` with 768-dimensional vectors and 50,000 points tells you a lot about what that RAG system indexes.
- On **Chroma**, older versions expose `GET /api/v1/collections` without authentication by default.

### Prometheus Metrics as Intelligence

Model servers often expose `/metrics` on a dedicated port (Triton on 8002, TorchServe on 8082). These Prometheus-format endpoints return:

- Model names and version numbers currently loaded
- Inference request counts and latency percentiles
- Batch sizes being processed
- GPU memory utilisation per model

This is passive intelligence. You learn the deployment topology, which models handle the most traffic, how much GPU capacity backs them, and whether the system is under load. All without sending a single request to the inference API itself.

### Debug Interfaces and Information Leakage

FastAPI-based ML services auto-generate `/docs` (Swagger UI) and `/openapi.json`. Both expose the full request/response schema, authentication requirements, and example payloads for every endpoint. Free reconnaissance.

MLflow's GraphQL endpoint  `/graphql` has historically bypassed the REST API's authentication controls. An unauthenticated attacker could query internal resolvers like `mlflowSearchRuns` and `mlflowGetRun`, extracting host machine usernames, source code paths via `mlflow.source.name` tags, and the full project inventory. Appending `?debug=true` or `?verbose=1` to AI gateway endpoints can trigger raw stack traces that reveal filesystem paths, installed library versions, and, occasionally, hardcoded credentials that failed to load from environment variables.

### Jupyter Notebook Enumeration

`GET /api/kernels` on an unauthenticated Jupyter instance returns the names, kernel IDs, and last activity timestamps of running kernels. Enough to infer what code is executing. But the real value is in the notebook cells. Data scientists routinely store cleartext credentials for MLflow (`MLFLOW_TRACKING_USERNAME`, `MLFLOW_TRACKING_PASSWORD`), cloud storage access keys, and Hugging Face tokens directly in their code. This is often the bridge between compromising one service and gaining access to everything else in the stack.

### Case Study: IBM X-Force ML Training Infrastructure Attack Chain

In 2025, [IBM X-Force Red documented](https://securityintelligence.com/x-force/attacking-ml-training-infrastructure/) a complete attack chain that demonstrates exactly what this task teaches.

The attack started with a phishing attempt targeting a data scientist, which gave the attackers access to an Azure ML environment. Inside a Jupyter notebook, they found cleartext MLflow credentials stored in a cell. Using those credentials, they ran MLOKit, a CLI tool built by IBM's X-Force Red team specifically for MLflow reconnaissance.

MLOKit automated the entire enumeration chain. `list-models` to inventory every registered model. `download-model` to pull version metadata and artifact URIs. The tool walked the same API endpoints we covered above, but in an automated sequence. The entire ML portfolio was exfiltrated through standard REST API calls that the MLflow server treated as normal authenticated requests.

The chain went:

- Jupyter notebook (credentials) to MLflow tracking server (model registry) to S3 storage (model artifacts).
- Three components, each trusting the one before it.
- The security failure was not a software vulnerability. It was that each service was accessible, and the credentials to connect them were sitting in a notebook cell.

IBM later extended MLOKit to support SageMaker and other platforms, adding modules for `list-notebooks`, `add-notebook-trigger` (for code execution via lifecycle configurations), and `poison-model` (for overwriting model artifacts). The tool is open-source on [GitHub](https://github.com/xforcered/MLOKit).

### Q&A

Time to extract intelligence from the services I fingerprinted. Open the **Cyphira Data Extractor** agent.

I confirmed that 10.10.45.12 runs MLflow, 10.10.45.15 runs Triton, 10.10.45.18 runs Qdrant, 10.10.45.20 runs Jupyter, and 10.10.45.22 runs MinIO. Now I need to pull everything these services are willing to hand over.

**Step 1: Enumerate MLflow experiments.**

<img width="941" height="357" alt="image" src="https://github.com/user-attachments/assets/f544f8a6-352d-4485-a471-3d32310238c6" />

**Step 2: List registered models.**

<img width="944" height="259" alt="image" src="https://github.com/user-attachments/assets/7629b247-d119-406e-b88c-449fd96c1bae" />

**Step 3: Extract model version details.**

<img width="943" height="348" alt="image" src="https://github.com/user-attachments/assets/d6870aab-cae6-4598-88b3-9cbbe8155551" />

**Step 4: Query Triton model config.**

<img width="936" height="235" alt="image" src="https://github.com/user-attachments/assets/83db20ab-a9f6-49f1-930f-b4319e2973cc" />

**Step 5: Enumerate the vector database.**

<img width="938" height="232" alt="image" src="https://github.com/user-attachments/assets/d25c75e7-8cd5-4d10-a006-5e5cc1df83d5" />

**Step 6: Check the Jupyter notebooks.**

<img width="946" height="632" alt="image" src="https://github.com/user-attachments/assets/dd7a70a5-6381-416e-a2e3-00c16dcc49be" />

What MLflow REST API endpoint would you use to retrieve the artifact storage location for a specific model version?

`/api/2.0/mlflow/model-versions/search`

What is the cleartext password for the MLflow service account stored in the Jupyter notebook on 10.10.45.20?

`cyphira-MLfl0w-2024!`

## Mapping the AI Attack Surface

### How AI Expands the Traditional Attack Surface

Previously, we catalogued 14 AI components across 20+ ports. Compare that to a traditional web application, which adds roughly 5 ports to a network, but the port count alone is not the full picture.

These services constantly talk to each other. The inference server pulls features from the vector database. The orchestration platform pushes model updates to the registry. Jupyter notebooks connect to everything. Prometheus scrapes metrics from every service. If one component binds to 0.0.0.0 instead of 127.0.0.1, the entire internal mesh becomes reachable.

The security perimeter for an AI deployment is not the external firewall. It extends deep into the internal communication pathways between these services. Overly permissive network policies and the absence of mTLS between data-hungry components are the norm, not the exception.

### Platform Misconfigurations That Attackers Map

Every major ML platform has documented misconfigurations that turn routine deployments into reconnaissance targets.

**MLflow** shipped without authentication by default before version 2.x.

- Even after authentication was added, CVE-2026-2635 revealed that MLflow's `basic_auth.ini` file contained hardcoded default credentials.
- Attackers running mass port scans on port 5000 could authenticate using these defaults.
- CVE-2026-2033 went further: a directory traversal flaw in the artifact handler allowed unauthenticated remote code execution.
- Both scored CVSS 9.8.

**Kubeflow** dashboards are frequently deployed without OIDC authentication and exposed via a basic Kubernetes LoadBalancer or NodePort.

- An unauthenticated user can access the full Kubeflow interface and spawn Jupyter notebooks, which are attached to Kubernetes service accounts with cluster-level permissions.
- That is a direct path from an open dashboard to container orchestration access.

**TorchServe** exposes a management API on port 8081 that allows dynamic model registration from arbitrary URLs.

- If that port is accessible, an attacker can instruct the server to download and load a malicious .mar (Model Archive) file from an external server.
- TorchServe executes initialisation code during model loading, so loading a crafted archive achieves remote code execution.

**SageMaker** notebooks with `DirectInternetAccess: Enabled` accept inbound connections from the internet.

- A 2024 cloud security report found that 82% of organisations using SageMaker had at least one notebook configured this way.

### Model Registries: The Highest-Value Target

Let's talk about why an unsecured registry is the single most damaging reconnaissance finding.

A registry does not just store model files. It stores the complete lineage:

- Model names
- Version history
- Stage labels (staging, production, archived)
- Creation timestamps
- Run IDs linking back to full training metadata
- Artifact URIs revealing internal cloud storage paths
- User ID of every contributor

One open registry maps the entire ML product portfolio. IBM X-Force documented the exploitation pattern:

- An attacker finds MLflow credentials in a Jupyter notebook
- They run MLOKit against the registry
- Then exfiltrate every model artifact

The registry is the map that tells the attacker where everything else is stored.

### Supply Chain Reconnaissance

AI systems depend heavily on external resources, and those dependencies are discoverable during reconnaissance.

Hugging Face tokens appear in GitHub repositories via simple dorks: `filename:.env HF_TOKEN`.

- They appear in `.env` files, CI/CD pipeline logs, and Kubernetes secrets.
- A compromised token grants read and write access to the organisation's private models and datasets on the Hugging Face platform.

**Dependency confusion** applies to ML pipelines just as it applies to traditional software.

- ML projects have large `requirements.txt` files with internal package names.
- If an internal package like `company-data-utils` is not registered on PyPI, an attacker can register it there.
- Kubeflow pipelines that build containers at training time pull packages live, so a typosquatted or misconfigured package can execute code inside the training cluster.

**Model download sources** are identifiable during reconnaissance.

- If the organisation pulls models from the Hugging Face Hub or the PyTorch Hub, the download paths are visible in configuration files, notebook cells, and container build logs.
- An attacker who can inject a malicious model into an upstream source (or replace one via a compromised HF token) poisons the entire supply chain.

### MITRE ATLAS Mapping

 ATLAS is modelled after ATT&CK but specifically covers adversarial threats to AI and ML systems. It contains 15 tactics, 66 techniques, and 46 sub-techniques as of late 2025.

Here is how the room's content connects:

| **Room Content**                                                        | **ATLAS Technique**                    |
| ----------------------------------------------------------------------- | -------------------------------------- |
| Port scanning for AI services, probing endpoints                        | AML.T0006 (Active Scanning)            |
| Locating model registries and training artifacts through unsecured APIs | AML.T0007 (Discover ML Artifacts)      |
| Finding exposed HF tokens and poisoned dependencies                     | AML.T0010 (ML Supply Chain Compromise) |
| Enumerating LLM configurations and API compatibility                    | AML.T0014 (Discover ML Model Family)   |
| All of the above, collectively                                          | AML.TA0002 (Reconnaissance tactic)     |

ATLAS techniques are not just labels. They provide a shared vocabulary for communicating findings. When you write a reconnaissance report, mapping each finding to an ATLAS technique ID tells the reader exactly what category of activity you performed and what risk it represents.

### Case Study: ShadowRay Campaign (CVE-2023-48022)

The [ShadowRay campaign](https://www.oligo.security/blog/shadowray-2-0-attackers-turn-ai-against-itself-in-global-campaign-that-hijacks-ai-into-self-propagating-botnet) is the clearest example of how reconnaissance of a single AI component cascades into full infrastructure compromise.

Ray's Job Submission API on port 8265 shipped without authentication by design. Anyscale, the company behind Ray, maintained that Ray should run only within trusted network boundaries and that the lack of authentication was an intentional feature, not a bug. They disputed the CVE. Attackers disagreed.

- Using Shodan, they identified over 230,000 publicly exposed Ray dashboards. Once they found an open dashboard, they submitted malicious jobs through `/api/jobs/` containing multi-stage payloads.
- The first stage performed reconnaissance on the compromised host: reading `/etc/passwd` to enumerate users and running `printenv` to dump environment variables and harvest AWS IAM tokens and other cloud credentials. Using those stolen credentials, the attackers pivoted laterally across cloud infrastructure.
- The primary objective was resource theft. The attackers hijacked GPU compute nodes and deployed XMRig cryptocurrency miners. They capped CPU usage at 60% and disguised their processes as legitimate Linux kernel workers to avoid detection.

The ShadowRay 2.0 variant, active from late 2025, added significant sophistication.

- Attackers used LLMs to generate adaptable malware payloads and established persistence through hidden cron jobs and systemd services.
- They hosted their payloads on GitLab and, when those repositories were taken down, migrated them to GitHub within days.
- The campaign also deployed sockstress (a TCP exhaustion tool) against production websites, turning the operation into a multi-purpose botnet.

**Why this matters for this room**: _the entire ShadowRay chain started with reconnaissance. Finding an exposed dashboard. Confirming it was Ray. The API was unauthenticated and ready for job submission. One exposed component, discovered through the same techniques we have been practising, led to credential theft, lateral movement, and large-scale resource hijacking._

### Q&A

The Cyphira Jupyter notebook at 10.10.45.20 contains a Hugging Face token (hf_kR7mXpQvL9nJwT2yBcDfAeGh8iKlMnOp). The internal-kb-embedder model on MLflow references sentence-transformers/all-MiniLM-L6-v2 as its base model. What ATLAS technique ID covers the risk of these exposed supply chain dependencies?

`AML.T0010`

You scanned the Cyphira subnet with nmap, probed endpoints with curl, and extracted metadata from MLflow APIs. All of these activities fall under one overarching ATLAS tactic. What is its ID?

`AML.TA0002`

## Structured Reconnaissance Methodology and Detection

### The 5 Phases

Think of this as a checklist. It follows the same order we covered in the room, but structured as a process you can run start to finish.

**Phase 1: Passive Reconnaissance**

Before you touch the target network, see what is already publicly visible.

- Search **Shodan**, **Censys**, and **FOFA** for **AI service banners** on the organisation's IP ranges.
    - They work here:
        - `port:5000 "MLflow"`,
        - `port:8888 title:"Home Page - Select or create a notebook"`,
        - `http.title:"Ray Dashboard"`.
- Search **GitHub** for **leaked credentials**.
    - `filename:.env MLFLOW_TRACKING_URI`,
    - `filename:.env HF_TOKEN`,
    - `filename:config.json model_name site:github.com`.
- These turn up _MLflow connection strings_ and _Hugging Face tokens_ more often than you would expect.

Check **arXiv** and **engineering blogs** for published model architectures.

- Teams regularly _publish papers describing the exact frameworks and infrastructure they use_.
- This maps to ATLAS technique **AML.T0000 (Search for Victim's Publicly Available Research Materials)**.

Check DockerHub and GitHub Container Registry for organisation-named ML images. Public container images frequently contain hardcoded configurations. Look at job postings. A listing for an "MLflow Administrator" or "Kubeflow Platform Engineer" tells you exactly what is deployed.

**Phase 2: Active Scanning**

Target the AI-specific ports from the reference table under **The AI Infrastructure**:

`nmap -p 5000,6333,8000,8001,8002,8080,8265,8500,8501,8888,9000,11434,19530 -sV --script=http-title,http-headers <target>`

That single command covers most ML serving components.

- Pay attention to gRPC services on ports 8001 and 8500 that Nmap may report as generic.
- Follow up with grpcurl for anything that looks like gRPC.
- Check for Prometheus metrics endpoints (`/metrics`) on every discovered service.
- These are often on separate ports (8002 for Triton, 8082 for TorchServe) and leak deployment topology.

**Phase 3: API Fingerprinting**

Run ffuf or feroxbuster with an AI-specific wordlist against every discovered HTTP service. Your wordlist should include:

`/v1/models`  
`/v2/models`  
`/v2/health/ready`  
`/api/2.0/mlflow/experiments/list`  
`/api/2.0/mlflow/registered-models/list`  
`/pipeline/apis/v1beta1/pipelines`  
`/api/serve/deployments/`  
`/v1/schema`  
`/v1/meta`  
`/api/kernels`  
`/api/contents`  
`/openapi.json`  
`/docs`  
`/graphql`  
`/metrics`  
`/api/tags`  
`/api/show`  
`/collections`  
`/healthz`  
`/ping`

For each endpoint that returns a 200, apply the fingerprinting techniques: check response headers, parse JSON structure, and look at error messages from malformed requests.

**Phase 4: Metadata Extraction**

For every confirmed AI service, run the enumeration chain.

- On **MLflow**: Experiments, registered models, model versions (with artifact URIs and user IDs), training runs, and artifact listings. Five API calls that map the entire ML portfolio.
- On **Triton** or **TF Serving**: Model config endpoints for tensor specs and framework identification.
- On **vector databases**: Schema and collection endpoints for data type and embedding model identification.
- On **Jupyter**: Kernel listings and notebook cell contents for cleartext credentials.

**Phase 5: Supply Chain Review**

Identify model download sources visible in configurations, notebook cells, and container build logs. Check whether internal model artifact buckets (S3, GCS, MinIO) are publicly readable. Audit `requirements.txt` and `Pipfile` contents for internal package names that could be squatted on PyPI. Check container registries for image pull access without credentials.

## Tool Reference

Here is a quick reference for the tools that support each phase:

|   |   |   |
|---|---|---|
|**Tool**|**What It Does**|**Phase**|
|Shodan / Censys / FOFA|Internet-wide search for AI service banners|Phase 1|
|GitHub search (dorks)|Find leaked credentials and configs in public repos|Phase 1|
|Nmap (with NSE scripts)|Port discovery and service version detection|Phase 2|
|grpcurl|Interact with gRPC services, dump protobuf schemas if reflection is enabled|Phase 2|
|ffuf / feroxbuster|Directory brute-forcing with AI-specific wordlists|Phase 2, 3|
|curl|Manual HTTP probing, header analysis, error triggering|Phase 3, 4|
|MLOKit (IBM X-Force Red)|Automated MLflow enumeration and model exfiltration|Phase 4|
|Nuclei (ProjectDiscovery)|Template-based scanning for known AI service misconfigurations|Phase 2, 3|
|Agrus Scanner|Purpose-built shadow AI detection with 50+ AI-specific probes across all 65,535 ports|Phase 2|

### What Your Reconnaissance Looks Like From the Other Side

Everything you just learned has a signature in logs. If you understand what your activity looks like to a defender, you become better at both attack and defence.

**Model enumeration pattern**

- A burst of sequential GET requests to `/v2/models` from a single IP. Normal users do not query model listings repeatedly in rapid succession.
- In a SIEM, this appears as 10-50 requests to the same endpoint within seconds, all from a single source.

**Scripted MLflow access**

-  API calls to `/registered-models/list` and `/model-versions/search` without a corresponding UI session.
- The MLflow web interface generates specific session cookies and additional requests.
- Raw API calls without those are a clear indicator of scripted enumeration.
- This is the exact pattern MLOKit produces.

**Prometheus scraping from outside the monitoring stack.**

-  `/metrics` requests from IPs not in the known monitoring CIDR.
- Your Prometheus server has a known IP.
- Anyone else reading those metrics is performing reconnaissance.

**AI-aware port scanning.**

- Port scans that hit 5000, 8000, 8001, 8080, 8265, 8888 in sequence from the same source.
- That is not a random scan.
- That is someone running the same Nmap command from Phase 2 of this methodology.

**Path traversal against MLflow artifacts.**

- Requests containing `../` or `%2e%2e%2f` against MLflow artifact endpoints.
- This signals someone probing for CVE-2026-2033.

**Jupyter access without a session.**

- `/api/kernels` and `/api/contents` requests without a valid session cookie.
- Either an attacker found the notebook server, or an automated tool is enumerating it.

### Quick Wins for Reducing Your Reconnaissance Surface

These are not full hardening guides. The [AI Threat Modelling Assessment](https://tryhackme.com/room/aithreatmodellingassessment)room covers mitigation in depth. These are the changes you can make today that have the greatest impact on reducing what an attacker can discover.

- Enable MLflow authentication. Set `MLFLOW_TRACKING_USERNAME` and `MLFLOW_TRACKING_PASSWORD`, or deploy behind an authenticating reverse proxy
- Disable Jupyter's `--allow-root` and `--ip=0.0.0.0` flags in production. Require token authentication. Never bind to 0.0.0.0 without a VPN or ingress-level auth
- Block AI-specific ports at the network perimeter. Ports 5000, 8000 through 8002, 8080, 8265, 8500/8501, 8888, and 9000 should never be internet-facing without explicit intent
- Disable Triton's model control endpoint with `--model-control-mode none` in production to prevent unauthenticated model loading and unloading
- Restrict Prometheus metrics endpoints to your internal monitoring CIDR. The `/metrics` endpoints on Triton (port 8002) and TorchServe (port 8082) expose model names, GPU utilisation, batch sizes, and deployment topology. They should only be reachable from your monitoring stack, not from the general network.
- Rotate and scope Hugging Face tokens. Use fine-grained tokens with read-only permissions and minimal scope
- Strip debug headers and verbose error messages from ML service responses before they reach untrusted networks
- Audit MinIO and S3 bucket policies hosting model artifacts. Model weights should not be in publicly readable buckets

### Case Study: Hugging Face Spaces Breach (June 2024)

_In June 2024, attackers gained unauthorised access to the [Hugging Face Spaces](https://huggingface.co/blog/space-secrets-disclosure) platform and accessed authentication secrets stored by developers in their hosted applications. These secrets included HF tokens that could access private models, datasets, and configurations._

- The breach demonstrated exactly what Phase 1 and Phase 5 of the methodology target.
- HF tokens stored in application secrets are the same kind of credential that shows up during passive reconnaissance through GitHub dorks, .env files in public repos, and CI/CD pipeline logs.
- Once an attacker has a valid HF token, they have read and write access to whatever that token is scoped to.

Hugging Face's response mapped directly to the "Quick Wins" list above. They revoked compromised tokens, recommended that all users migrate to fine-grained access tokens with minimal scope, and advised against storing tokens in application secrets where platform compromises could expose them.The lesson is straightforward.

The credentials that Phase 1 passive reconnaissance discovers (GitHub dorks, exposed .env files) are the same credentials that get compromised in platform breaches. Rotating and scoping tokens is not just a best practice. It is the direct countermeasure to both reconnaissance and breach exposure.

### Q&A

A SIEM log shows requests to /api/2.0/mlflow/registered-models/list from an IP with no corresponding MLflow UI session. What tool's access pattern does this match?

`MLOKit`

What is the single most effective quick win for preventing unauthenticated access to the MLflow tracking server?

`Enable MLflow Authentication`

## Conclusion

### Mapping to Industry Frameworks

The techniques and findings from this room align with several widely adopted security frameworks. This mapping helps you communicate your reconnaissance results in language that security leaders, auditors, and compliance teams already understand.

**MITRE ATLAS (Adversarial Threat Landscape for AI Systems)**

ATLAS is the primary framework for AI-specific threats. Here is how this room maps:

|Room Content|ATLAS Technique ID|Technique Name|
|---|---|---|
|Shodan and GitHub dorks for AI infrastructure|AML.T0000|Active Scanning|
|Locating model registries and artifacts through unsecured APIs|AML.T0048|Discover ML Artifacts|
|Finding exposed HF tokens and dependency confusion|AML.T0040|ML Supply Chain Compromise|
|Enumerating LLM configs and API schema compatibility|AML.T0069|Discover LLM System Information|
|All reconnaissance activities collectively|AML.TA0002|Reconnaissance (Tactic)|

**MITRE ATT&CK (Enterprise)**

Traditional ATT&CK techniques also apply because AI reconnaissance uses many of the same methods as conventional network assessment:

|Room Content|ATT&CK Technique ID|Technique Name|
|---|---|---|
|Port scanning for AI-specific services|T1046|Network Service Scanning|
|Extracting deployment topology from metrics and metadata|T1592|Gather Victim Host Information|
|Probing for unauthenticated management interfaces|T1595.002|Vulnerability Scanning|
|Collecting AI infrastructure intelligence before engagement|TA0043|Reconnaissance (Tactic)|

**OWASP Top 10 for LLM Applications (2025)**

Several OWASP LLM risks directly relate to findings from this room:

|Room Finding|OWASP LLM ID|Risk Name|
|---|---|---|
|Exposed MLflow servers, Jupyter notebooks, unauthenticated APIs|LLM05|Improper Output Handling (relates to exposed metadata and information leakage through API responses)|
|Model artifacts downloadable from unsecured registries|LLM06|Excessive Agency (relates to model access without proper authorisation controls)|
|Leaked HF tokens, dependency confusion, poisoned model files from public hubs|LLM03|Training Data Poisoning / Supply Chain Vulnerabilities|
|Default credentials, missing authentication on MLflow and Kubeflow|LLM10|Model Theft (reconnaissance enables direct model exfiltration)|

**NIST AI Risk Management Framework (AI RMF 1.0)**

NIST AI RMF organises AI risk into four functions: Govern, Map, Measure, and Manage. This room's content falls primarily under the Map function:

- **Map 1.1:** AI system components and their interactions are identified. This is exactly what Tasks 2 through 4 accomplish. You cannot assess risk in AI infrastructure you have not discovered.
- **Map 1.5:** Potential risks of the AI system are assessed. The attack surface mapping in Task 5 directly supports this. Identifying misconfigurations, exposed registries, and supply chain risks is risk assessment at the infrastructure layer.
- **Map 3.2:** Risks related to third-party AI resources are identified. The supply chain reconnaissance in Task 5 (Hugging Face tokens, PyTorch Hub dependencies, public model registries) maps here.
- **Measure 2.6:** Processes exist to determine whether AI systems are functioning as intended. The Prometheus metrics and debug interface enumeration from Task 4 support this, as unexpected exposed endpoints indicate systems not functioning as intended from a security perspective.

**NIST Cybersecurity Framework (CSF 2.0)**

The room's content aligns with CSF 2.0's Identify function:

- **ID.AM (Asset Management):** Discovering and inventorying AI infrastructure components across the network. This is the core purpose of the entire room.
- **ID.RA (Risk Assessment):** Mapping the AI attack surface and identifying misconfigurations that introduce risk.

Thank you for reading through!
