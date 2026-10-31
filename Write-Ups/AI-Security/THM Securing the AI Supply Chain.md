## Safe Serialisation Formats

### Defence 1: SafeTensors

SafeTensors is a serialisation format created by Hugging Face specifically for ML model weights. Unlike pickle, SafeTensors is designed with one strict guarantee: no code execution is possible during loading. The comparison below shows the difference compared to pickle's format:

|Feature|Pickle|SafeTensors|
|---|---|---|
|**Structure**|Arbitrary Python bytecode|JSON header + raw binary tensor data|
|**Code execution**|Yes, via `__reduce__` and other opcodes|No: format cannot encode executable instructions|
|**Content**|Any Python object|Only tensor data (weights, biases)|
|**Validation**|None: loads and executes blindly|Header is parsed and validated before any data is read|
|**Speed**|Moderate|Fast: zero-copy memory mapping|

#### Migrating from Pickle to SafeTensors

If you have an existing pickle model, be aware that the conversion step requires loading it, and that's where malicious payloads would execute. The SafeTensors output will be clean, but he conversation itself is not risk-free:

```python
import torch
from safetensors.torch import save_file, load_file

# Step 1: Load the existing pickle model safely
# (weights_only=True restricts what pickle can do; explained in Defence 2 below)
model_weights = torch.load("model.pkl", weights_only=True)

# Step 2: Save as SafeTensors
save_file(model_weights, "model.safetensors")

# Step 3: Load the SafeTensors model (always safe)
safe_weights = load_file("model.safetensors")
```

SafeTensors only stores tensor data (model weights). It does not store Python objects, optimiser state, or training configuration. For most interface deployments, this is exactly what you need.

### Defence 2: PyTorch weights_only=True

When you call `torch.load()`, PyTorch uses pickle internally to deserialise the file. By default, this means pickle's full capability is active, including code execution. Setting `weights_only=True` tells PyTorch to restrict the unpickler so that it can only reconstruct tensor objects (the numerical weights and biases that make up the model). Any pickle instructions that try to import modules like `os` or call functions like `system()` are blocked and raise an error instead of executing.

```python
import torch

# UNSAFE: pickle can execute any embedded code
model = torch.load("model.pt")

# SAFE: pickle is restricted to tensor reconstruction only
model = torch.load("model.pt", weights_only=True)
```

Starting with PyTorch 2.6, `weights_only=True` is the **default behaviour**. Earlier versions default to the unsafe mode, so you must set it explicitly.

### Limitations

SafeTensors eliminates serialisation-level code execution. That is a significant win, but it is not a complete defence. Three risks remain:

In 2023, researchers demonstrated that a file with a `.safetensors` extension could actually contain pickle bytecode disguised under the wrong extension. The Hugging Face SFConvertBot service was briefly affected by a similar bypass ([CVE-2023-6730](https://nvd.nist.gov/vuln/detail/CVE-2023-6730)). File extensions alone cannot be trusted.

SafeTensors also only protects against code execution at **load time**. It cannot prevent malicious logic embedded in a model's architecture from executing at _inference time_. A Keras Lambda layer, for example, can run arbitrary Python every time the model makes a prediction. Tasks 6 and 7 cover detection tools for this class of threat.

The key lesson is that **safe serialisation is necessary but not sufficient.** Always verify the actual format, inspect the architecture, and never assume a single defence covers every threat.

### Q&A

What serialisation format was created by Hugging Face to replace pickle for ML models?

`SafeTensors`

What PyTorch parameter prevents code execution when loading pickle-based models?

`weights_only=True`

## Model Verification and Provenance

### SHA-256 Checksum Verification

A **checksum** is a fixed-length string computed from a file's contents. If even a single byte changes, the checksum changes completely. SHA-256 is the standard for file integrity verification in ML pipelines.

For stronger assurance, look for **cryptographic signatures** on model artefacts. A checksum verifies that a file has not changed; a digital signature additionally verifies _who published it_. Signed models are still uncommon, but platforms are moving toward signed commits as a provenance standard.

### Model Cards

A **model card** is documentation that ships with a model, describing what it does, how it was trained, and where it falls short. The format was proposed by Mitchell et al. (2019) and has since become an industry standard for repositories such as Hugging Face.

When evaluating a model's provenance, check these model card sections:

- **Model details:** Author, organisation, version, and licence. No author or missing licence is an immediate red flag.
- **Intended use:** What the model is designed for. Vague or overly broad claims suggest a generic or poorly documented model.
- **Training data:** Dataset name, size, and source. No training data description is a strong warning sign.
- **Performance:** Metrics on standard benchmarks. No metrics, or unrealistically high claims, warrant scepticism.
- **Limitations:** Known failure modes and biases. Every model has limitations. A card with none is incomplete.

A missing or sparse model card is one of the strongest warning signs of a suspicious model. Legitimate model authors invest significant effort in documentation.

### Borrowed Weights, Inherited Risk

The checks above apply to more than just full model files. Two trends have expanded the definition of a supply chain artefact.

### LoRA Adapters

Modern fine-tuning increasingly uses **LoRA** (Low-Rank Adaptation) and similar parameter-efficient methods. Instead of retraining an entire model, teams download a small **adapter** file that bolts onto a base model to modify its behaviour for a specific task. These adapters are shared on the same platforms as full models, often by third-party contributors. A clean base model paired with a malicious adapter is still a compromised model. Apply the same intake process: quarantine the adapter, verify its source, scan it, and only then merge it with your base model.

### Model Merging and Conversion Services

Collaborative platforms offer services that combine multiple models or convert between formats. These run in shared environments, and the artefact you receive may not match what the original author published. Researchers have demonstrated that [conversion services can be exploited to inject malicious code during processing](https://hiddenlayer.com/research/silent-sabotage). Treat any model that has passed through a third-party pipeline as a new artefact requiring fresh verification.

### The Model Acquisition Framework

Every artefact type above (full model files, adapters, converted outputs) needs the same structured intake process. Production teams need more than a checklist of individual checks; they need a gate that every artefact must pass through before it reaches production:

|Step|Action|Purpose|
|---|---|---|
|**1. Quarantine**|Download the model into an isolated staging area, never directly into production|Prevent untested artefacts from reaching live systems|
|**2. Source verification**|Verify the author, organisation, and repository. Check for verification badges, publication history, and community reputation|Establish provenance and credibility|
|**3. Integrity check**|Compute SHA-256 checksums and compare against the author's published values|Confirm the file has not been tampered with in transit|
|**4. Security scan**|Run Fickling, ModelScan, and dependency auditing tools (covered in Tasks 5-8). Inspect the model card for completeness|Detect malicious content and known vulnerabilities|
|**5. Approve or reject**|Based on findings, either promote the model to production or quarantine it permanently|Enforce a gate between untested and trusted artefacts|

Technical scanning tools alone cannot catch everything. A model can pass Fickling and ModelScan but still contain a subtle data poisoning backdoor. The framework's value is in requiring multiple independent checks before trust is granted. No single step is sufficient on its own.

In practice, enterprise teams automate this framework using model registries, where models are tagged `untested` and only advance to `approved` after passing every stage. Common examples include MLflow Model Registry, AWS SageMaker Model Registry, Azure ML Model Registry, and Hugging Face Hub.

<img width="1800" height="900" alt="image" src="https://github.com/user-attachments/assets/f0e0335d-802a-4e38-81ae-19e8b3e1b33e" />

The framework above assumes you have a file. When you consume a model through an API, the gates look different: there is no checksum to verify, no model file to scan. But the governance instinct remains the same: you are still deciding whether to trust an artefact you did not build.

### Q&A

What is the first step in the Model Acquisition Framework when a new model is received?

`Quarantine`

Examine the checksums on the VM. Which model file does not match its expected hash?

`model_review_v2.pkl`

## Behavioural Analysis

### Model Loading Telemetry

Every model load generates a session event stream. A clean load is short and contained:

```bash
SESSION START — model_load
MODEL LOAD BEGIN — /models/sentiment_model.pkl (pickle)
FILE ACCESS — /models/sentiment_model.pkl (rb) [normal]
MODEL LOAD COMPLETE — object_type: SentimentModel
SESSION STOP — model_load
```

Five events, with nothing outside the file boundary. A file is read, and a `SentimentModel` was returned. This is the baseline

### Q&A

What object type does the compromised model's telemetry show on load completion instead of a model?

`int`

## Scanning Models Before Use

### Fickling: Static Pickle Analysis

**Fickling** (by Trail of Bits) decompiles pickle bytecode back into readable Python source without executing the file. It would expose a payload in the telemetry.

Fickling also provides an automated safety check. The `-p` flag prints the assessment to the terminal. As an example, a model calling `os.system` would be flagged as overtly malicious. Without the `-p` flag, results are written silently to a `safety_results.json` file instead of the terminal. 

For a clean model, fickling won't produce an output. Silence means no issues were detected. Note that the exact format of `---check-safety -p` varies by fickling version. the key information is always the severity assessment and any identified dangerous operations. 

### ModelScan: Multi-Format Model Scanning

**ModelScan** (by Protect AI) extends beyond pickle and scans multiple model formats, including PyTorch, TensorFlow, and Keras. It assigns severity levels to findings.

#### Interpreting Scanner Results

|Severity|Meaning|Action|
|---|---|---|
|**CRITICAL**|Confirmed dangerous operation (e.g., `os.system`, `subprocess`)|Do not use. Quarantine immediately.|
|**HIGH**|Likely dangerous operation (e.g., `eval`, network calls)|Do not use without thorough review.|
|**MEDIUM**|Suspicious but potentially legitimate (e.g., custom unpickler)|Review carefully before use.|
|**LOW**|Informational (e.g., non-standard pickle opcodes)|Note and monitor.|

No scanning tool catches everything. Fickling and ModelSCan detect known patterns, But sophisticates attackers may use obfuscated techniques to evade detection. Scanning is one layer of defence, not a guarantee.

### Q&A

Which Trail of Bits tool performs static analysis of pickle files?

`Fickling`

What severity level does ModelScan assign to an `os.system` call in a model file?

`Critical`

## Architecture-Level Threats

Fickling and ModelScan catch malicious code hidden in pickle serialisation. But attackers have another technique: injecting malicious logic directly into a model's architecture. Keras includes **Lambda layers**, which let developers embed arbitrary Python functions into a model. Intended for custom operations during training, attackers can repurpose them to hide malicious code.

A Lambda layer executes code at **inference time**, meaning when the model processes actual inputs, not when the file is loaded. This makes it especially dangerous: the model loads cleanly, passes checksum verification, and only triggers the malicious behaviour when it starts handling real data. This attacker also survives format conversion: a malicious Lambda layer works the same whether the model is stored as `.h5`, `.keras`, or SafeTensors, because the logic lives in the architecture, not the serialisation. 

### Detecting Architecture-Level Threats

A legitimate model inspection produces one event per layer, all clean:

```bash
SESSION START — model_inspect
MODEL INSPECT BEGIN — /models/image_classifier.h5 (keras_h5) via h5py
LAYER — InputLayer "input_layer" [clean]
LAYER — Flatten "flatten" [clean]
LAYER — Dense "dense" [clean]
LAYER — Dense "dense_1" [clean]
MODEL INSPECT COMPLETE — 4 layers, 0 suspicious
SESSION STOP — model_inspect
```

Four layers with no anomalies. This is the baseline for a legitimate image classifier

### Q&A

Open the Telemetry terminal. How many layers does the compromised model's architecture contain?

`5`

## Architecture Inspection
