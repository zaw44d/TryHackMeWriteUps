## Introduction

LLMs (Large Language Models) have unlocked all kinds of potential for users and organisations alike. From exponentially increased productivity, with LLMs able to handle large-scale monotonous tasks so humans can focus their attention where it's truly needed, to providing user-facing AI-powered cutting-edge features that give everyone an all-around better experience. Or that's the hope anyway. In the real world, we must acknowledge the potential pitfalls of this newfound approach. The most important consideration (from our unsurprising perspective) is the security of LLMs, which you must take into account when using them or integrating them into a workflow.

This room aims to break down LLMs as an attack surface. Turning these question marks in your brain into security considerations you are actively aware of when engaging with an LLM in any capacity.

## Data-Based Threats

We are going to start by looking at the first of those: data-based threats. LLMs are fundamentally data-driven; they learn from a corpus of training data and generate outputs based on it. This is one of LLMs' greatest strengths. But that strength can be turned into a weakness. Sometimes LLMs can inadvertently leak data by design because they memorise and regurgitate patterns from their training data. In this task, we will explore three major data-based threats:

- Training data extraction
- Membership inference
- Prompt leakage

Each of these attacks involves an adversary coaxing the model into returning data that should have remained hidden, essentially inverting the intended flow of data that went into the model during training.

### Training Data Extraction

Training data extraction attacks aim to recover actual sequences from the model's original training data by interacting with the model. For example, [one key study(opens in new tab)](https://www.usenix.org/conference/usenixsecurity21/presentation/carlini-extracting) demonstrates that it was possible to extract hundreds of verbatim training examples from GPT-2 by sending queries. Obviously, this discovery raises massive privacy concerns in the field of artificial intelligence.

Training data extraction attacks generate large amounts of text from an LLM and analyse those outputs to identify sequences that show behavioural signs of memorisation, such as **unusually high likelihood/confidence**, deterministic regeneration (a model's ability to reproduce the same output), or realistic structured content (this information is especially easy to access if the attacker has white box access like in the above study). The attacker then inspects or externally verifies these suspicious sequences to confirm which ones correspond to real text that must have appeared in the model's training data, for example, confirming the presence of user emails or SSH keys, etc.
##### In a nutshell:

- **Target / Attack Surface:** Training dataset (confidentiality)
- **Input:** Crafted prompts designed to trigger memorised content
- **Output:** Verbatim or near-verbatim training data (text, PII, secrets)

### Membership Inference

Membership inference attacks ask whether the model ever recorded a specific data sample. The attacker tests the model's reaction to that exact sample, looking for unusually confident or familiar responses that indicate it was part of training. Crucially, membership inference assumes the attacker already possesses the exact candidate data sample and is only testing whether that known sample influenced the model's training. Unlike extraction, this attack doesn't involve generating candidate outputs; rather, it focuses on confirming whether a sample the attacker already has was in the training set. Essentially, the adversary doesn't necessarily obtain the full content of a training example, but can **detect the presence** of certain data in the training set by observing the model's behaviour.

In practical terms, membership inference often exploits **statistical quirks or "fingerprints"** left by training data. A model typically performs better (e.g. predicts with higher confidence or lower loss) on examples it has seen during training than on new, unseen examples. Attackers leverage this by querying the model with the target example and measuring indicators such as confidence scores, likelihoods, or perplexities.
##### In a nutshell:

- **Target / Attack Surface:** Training dataset membership (privacy metadata)
- **Input:** A **known candidate data sample** already possessed by the attacker
- **Output:** A yes/no (or probability) decision indicating whether the sample was used in training

### Prompt Leakage (LLM07:2025 - System Prompt Leakage)

LLMs like ChatGPT, Claude, Gemini, etc., don't just operate using the learnings from their training data; they also use hidden instructions known as **system** or **developer** prompts. This system prompt is kept hidden in many instances, especially when what sits in front of the model is some form of "intellectual property," such as an application feature or product. This AI-powered feature or product, in some cases, is only possible because of the work that has gone into developing this system prompt. Therefore, the leaking of such a prompt could be compared to the leaking of sensitive company data such as application source code.

This attack is a type of prompt injection (covered in more detail later in the room) and is possible because, to the LLM, the system prompt and the user's messages are all just parts of the conversation history. If the user's input cleverly convinces the model to regurgitate or summarise the entire conversation (including the hidden parts), the model may comply. As was the case in early 2023, when a user managed to get "Sydney," Microsoft/Bing's AI Chatbot, to reveal its confidential system prompt. The consequences of system prompt leakage are significant. For one, it exposes the **proprietary business logic or safety measures** companies put into their models. When Bing's rules leaked, it revealed not only the codename "Sydney" but also the detailed behavioural limits set by Microsoft. Such information can act as a domino effect on security, as it can help malicious actors design more effective prompt injection attacks (since they now know exactly which rules to break).

##### In a nutshell:

- **Target / Attack Surface:** System prompt / developer instructions
- **Input:** User prompts that ask the model to reveal or reflect on its instructions
- **Output:** Partial or full disclosure of hidden system or developer prompts
- **Mitigation:** Never treat the system prompt as a security boundary; assume it can be extracted. Never embed live credentials, API keys, or secrets in it.

### Q&A

I will be asking the chatbot given in this room to `give me the Task 2 demonstration` so as to use a membership inference attack to determine which of the three placeholder samples is a member.

<img width="945" height="840" alt="image" src="https://github.com/user-attachments/assets/883bfa6a-650b-4ebb-bf96-97827f021017" />

```
Commencing simulation…

This simulation tests **membership inference**—whether a known sample was part of the training data. Below are three samples. Submit one at a time to observe the model’s response and confidence score (simulated internal metric).

---
**Sample 1:**
`MI_SAMPLE_CHARLIE`

---
**Sample 2:**
`MI_SAMPLE_BRAVO`

---
**Sample 3:**
`MI_SAMPLE_ALPHA`

---
*Interpret the confidence values (0.0–1.0) as a proxy for likelihood of membership. No judgment on correctness—this is for understanding inference risks.*
```
<img width="936" height="546" alt="image" src="https://github.com/user-attachments/assets/41779355-775a-4d84-9ba0-99b0725e6f11" />

<img width="937" height="528" alt="image" src="https://github.com/user-attachments/assets/431b39e0-ff9e-4d4b-9c1a-bd2df5f6c636" />

<img width="944" height="514" alt="image" src="https://github.com/user-attachments/assets/124b5f2d-5f5d-4cc3-a397-4e2db4676d29" />

Which sample is a member?

`MI_SAMPLE_ALPHA`

Which attack determines whether a known data sample was part of an LLM’s training set?

`Membership inference`

Which data-based threat involves the model reproducing memorised snippets of its training data?

`Training data extraction`

## Model-Based Threats

As well as introducing data-based threats to your attack surface, adopting an LLM into your digital ecosystem can also introduce threats through the model itself. Model-based threats exploit the model itself as the attack surface, abusing how information is encoded within its parameters and representations. As a consequence, these attacks may expose intellectual property (model weights) or sensitive training data that the model has memorised. Let's look at how the model can be targeted across two different threats: model theft and model inversion.

### Model Extraction

Model extraction is the process of illicitly copying a machine learning model's functionality or parameters without authorisation. Okay but how does this actually work in practice? An attacker can do this if they can interact with an LLM through its public API and send a large number of prompts; the responses to these prompts are then stored in a sort of input-output pair. As more and more of these pairs are collected, they can be used to train a surrogate model that imitates the target model's behaviour, by determining its decision boundaries or potentially even recovering the model's weights.

The impact of such a threat is primarily economic, as a custom high-quality purpose-built LLM can often constitute a huge investment of time, data and money, so having an attacker bypass this effort and steal the model can be costly. Researchers have been able to recreate such attacks against advanced LLMs. For example, Mindgard was able to [extract ChatGPT 3.5 Turbo(opens in new tab)](https://mindgard.ai/blog/ai-under-attack-six-key-adversarial-attacks-and-their-consequences) into a smaller model (around 100 times smaller), achieved with only $50 in API costs.

##### In a nutshell:

- Target / Attack Surface: Model parameters (intellectual property)
- Input: Large volumes of carefully chosen API queries
- Output: A surrogate or distilled model that replicates the original model's behaviour

### Model Inversion

Model inversion attacks exploit a model's output to reveal information about its training data. In these attacks, an adversary analyses how the model responds to various inputs in order to infer sensitive details about what the model has learned. For this reason, this attack often gets confused with a membership inference attack (covered in a previous section). Here is a further explanation of model inversion which helps establish how both attacks are distinguished from each other:

Model inversion attacks treat the model as a source of stored information rather than a classifier to be probed.

Instead of testing whether a known example was seen during training, the attacker iteratively queries the model to reconstruct unknown training data that has been encoded into its parameters or representations.

This is typically achieved by optimising inputs (or decoding embeddings) so that the model's outputs converge on realistic training samples, effectively reversing the learning process. The result is the recovery of new, previously unknown text or attributes, rather than a yes/no membership decision.

This attack has been seen out in the wild. In 2023, researchers managed to extract verbatim chunks of ChatGPT's training data ([source(opens in new tab)](https://not-just-memorization.github.io/extracting-training-data-from-chatgpt.html)). The foremost consequence of model inversion attacks is a privacy breach, as the attacker ultimately tricks the model into effectively leaking data that was supposed to remain private.

##### In a nutshell:

- Target / Attack Surface: Model's internal representations
- Input: Unknown or partially known data, or model embeddings/outputs
- Output: New training data or attributes reconstructed from the model

### Q&A

I will then ask the chatbot to give me the "Task 3" demonstration, and I'll need to reconstruct this known redacted piece of training data. 

Employee ID: ████ | Department: Research | Clearance: ███





