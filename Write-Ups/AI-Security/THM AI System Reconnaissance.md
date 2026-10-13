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


### Q&A


## Mapping the AI Attack Surface


### Q&A



## Structured Reconnaissance Methodology and Detection


### Q&A


## Conclusion
