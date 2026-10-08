## AI Specific Assets and Attack Surfaces

If you have threat modelled traditional applications before, you are used to thinking about a familiar set of assets: databases, source code, configuration files, API keys, and user credentials. You know what they are, where they live, and how to protect them. 

AI systems change the picture. They introduce an entirely new class of assets that most security teams have never had to inventory, classify, or defend. Missing these assets during a threat assessment means missing entire categories of risk, and that's exactly the gap attackers exploit.

### AI Assets

|Asset|What It Is|Why It Matters|
|---|---|---|
|**Training Data**|The datasets used to teach the model its behaviour|Poisoning this data corrupts the model's outputs at the source. Unlike a database compromise, the damage is baked into the model itself.|
|**Model Weights / Parameters**|The numerical values that define what the model has learned|These _are_ the model. Stealing them means an attacker has a functional copy of your AI, months of compute and potentially millions in investment, gone.|
|**Embedding Vectors**|Numerical representations of text or data used for similarity computation, retrieval, or as input features to downstream models|Used in RAG pipelines, recommendation engines, and fraud detection systems. Poisoning or manipulating embeddings alters what information models see at query time.|
|**System Prompts**|Instructions that define the model's behaviour, constraints, and persona|Leaking these reveals your security controls, business logic, and guardrails, giving attackers a roadmap to bypass them.|
|**Feature Stores**|Preprocessed data repositories that feed real-time model inputs|Tampering with features changes what the model "sees" at inference time, without touching the model itself.|
|**Model Registry / Artifacts**|Stored versions of trained models ready for deployment|A compromised registry means an attacker can swap a legitimate model for a backdoored one, and no one may notice until it's too late.|

None of these assets map neatly onto traditional asset categories. A stolen database is serious, but a stolen model is a fundamentally different kind of loss, you can't just rotate a credential and move on. The asset that defines the model's learned behaviour is its model weights: once those are exfiltrated, the attacker has a functional copy of your AI. Meanwhile, if an attacker wants to give themselves a roadmap of your LLM's security controls and behavioural , constraints, the asset they would target is your system prompts. And a poisoned training data set doesn't trigger the same alerts as a modified database record because the corruption only surfaces after the model has been retrained and redeployed. 

### What Else makes AI Systems Different

Beyond new asset types, AI systems behave differently from traditional software, affecting how we model threats. Characteristics include:

- **Non-deterministic behaviour:** AI models, especially LLMs, can produce different outputs for the same input. This makes testing, auditing, and incident reproduction significantly harder than with deterministic software. If you've completed earlier rooms in this path, you'll already be familiar with this concept.
- **The black box problem:** Most AI models, particularly deep neural networks, lack the explainability of traditional application logic. You can't step through a model's reasoning the way you'd trace a code path. This forces defenders to think in terms of input-output behaviour and failure modes rather than code-level inspection.

Both of these characteristics have direct implications for threat modelling, and we will see them repeatedly surface as we work through the frameworks. **AI systems aren't just traditional applications with a model bolted on. They have different assets, behaviours, and ways of failing, and our threat models need to account for all of it.**

### Q&A

In a RAG-based system, which AI asset type is used to retrieve relevant context at query time?

`Embedding Vectors`

An attacker gains access to MegaCorp's model registry and swaps the production model for a modified version. Which AI-specific asset has been compromised?

`Model Registry / Artifacts`

## Data Supply Chain and STRIDE's Gaps

Knowing what to protect is only half the picture. We also need to understand **how those assets are built, moved, and consumed**, because every step in that process is an opportunity for compromise. This is where **data supply chain** comes in. 

### The AI Data Supply Chain

Traditional applications have software supply chains, dependencies, libraries, container images. You have likely already encountered supply chain threats in the form of compromised packages or malicious dependencies. AI systems inherit all of those risks and ass an entirely separate supply chain built around data. Here's how a typical AI model goes from raw data to production:

#### **Stage 1: Data Collection**

Training data is gathered from multiple sources, including web scraping, purchased datasets, internal databases, user-generated content, and third-party providers. At this stage, an attacker who can contribute or influence any of these sources has a foothold.

#### **Stage 2: Cleaning and Labelling**

Raw data is preprocessed, filtered, and labelled. In some pipelines this involves external annotation teams or automated labelling tools. In other cases, such as fraud detection, labels are derived implicitly from outcomes, like chargebacks or investigation results. Regardless of the method, compromised labels lead the model to learn the wrong associations. A mislabelled dataset doesn't look corrupted. It just quietly teaches the model to make incorrect decisions.

#### **Stage 3: Model Training**

The model learns patterns from the prepared data over days or weeks of compute. Any poison that survived the first two stages is now embedded in the model's **weights**. Unlike a compromised library you can patch, a poisoned model may need to be retrained from scratch, at significant time and cost.

#### **Stage 4: Validation and Packaging**

The trained model is evaluated, versioned, and stored in a model registry for deployment. If the registry itself is compromised, an attacker can swap a validated model for a backdoored one. The backdoored model passes standard validation checks because the trigger inputs (the specific patterns that activate the malicious behaviour) are absent from the validation dataset. Everything looks clean until the model encounters those triggers in production.

#### **Stage 5: Inference**

The model serves predictions in production. For LLM-based systems, this stage often includes a retrieval pipeline that retrieves additional context from vector databases or document stores at query time, introducing yet another injection point that doesn't exist in traditional applications.

Each stage is a link in the chain, and each link is a potential point of compromise. The critical difference from traditional software supply chains is **time**. A compromised npm package can be detected and reverted within hours. A poisoned training dataset may not reveal its effects for weeks or months, only surfacing after the model is retrained, validated, and deployed to production.

`Think about it for MegaCorp: The fraud detection system is retrained monthly on new transaction data. If an attacker can inject crafted transactions into that training pipeline over several months, they can gradually shift the model's decision boundaries, making specific fraud patterns invisible to detection. By the time anyone notices, the model has been approving fraudulent transactions for weeks.`

### Why STRIDE Alone Falls Short

Now that we understand AI's new assets and new supply chain concept, let's address the framework question: **can we use STRIDE as is?**

STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege), has been the backbone of threat modeling since Microsoft introduced it in the late 1990s. It remains highly effective for traditional applications. But when applied to AI systems without adaptation, it has documented gaps:

**Data integrity isn't a first-class concern at the training level.** STRIDE's Tampering category works well for data in transit or at rest. But tampering with training data is fundamentally different, the effects are diffuse, delayed, and nearly invisible. A poisoned training set doesn't throw an error. It produces a model that behaves incorrectly in subtle, hard-to-detect ways.

**Adversarial manipulation of model behaviour doesn't fit neatly into one category.** Crafting inputs designed to make a model misclassify, hallucinate, or bypass safety guardrails spans multiple STRIDE categories simultaneously, it's part Tampering, part Spoofing, part Elevation of Privilege depending on context. STRIDE wasn't designed for threats that blur across categories this way.

**The scope of privilege has expanded beyond what STRIDE originally envisioned.** When a model can take actions, browse the web, execute code, send emails, query databases, the **Elevation of Privilege** category still applies, but what constitutes "privilege" is fundamentally broader. A jailbroken chatbot with tool access isn't just a traditional privilege escalation. The model's entire set of tool permissions becomes the attacker's capabilities.

**Model-specific intellectual property theft is a different kind of disclosure.** Extracting a model's weights through carefully crafted API queries is technically **Information Disclosure**, but it's profoundly different from exfiltrating a database. The stolen asset is the organisation's entire AI capability, not a dataset, but a trained intelligence.

This isn't a criticism of STRIDE, it's a recognition that the framework needs adaptation, not replacement. The six categories are still valuable lenses for threat identification. They just need to be retuned for the AI context.

### Q&A

An attacker injects crafted data points into a training pipeline over several months, gradually shifting the model's decision boundaries. At which supply chain stage does the attacker inject the malicious data?

`Data Collection`

Which STRIDE category is insufficient for capturing the delayed, diffuse effects of training data poisoning?

`Tampering`

## Adapting STRIDE for AI Systems
