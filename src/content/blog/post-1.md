---
title: 'Building a Domain-Specific Medical Reasoning Model with Supervised Fine-Tuning on GPT-OSS-20B'
excerpt: Artificial Intelligence has made its way into healthcare, but deploying it meaningfully remains a challenge. Most general-purpose large language models (LLMs) are not built with the precision, contextual awareness, or domain-specific reasoning that healthcare demands. ...
publishDate: 'September 25 2025'
tags:
  - LLMs

---

Artificial Intelligence has made its way into healthcare, but deploying it meaningfully remains a challenge. Most general-purpose large language models (LLMs) are not built with the precision, contextual awareness, or domain-specific reasoning that healthcare demands. They often hallucinate, misinterpret clinical terminology, or fail to capture the nuances of multi-step diagnostic reasoning, all of which can have serious consequences when lives are at stake.

This is where scaling matters. While smaller models can handle simple summarization or keyword extraction, advanced healthcare tasks such as clinical decision support, longitudinal patient record analysis, or multi-hop reasoning over medical literature require models with both capacity and depth of reasoning. A 20B parameter model like GPT-OSS-20B provides the representational power to capture medical knowledge at scale, but out of the box, it still lacks the specialization needed for analyzing, reasoning, and clinical accuracy.

Fine-tuning bridges this gap. By aligning a large model to domain-specific datasets and medical reasoning tasks, we move from “generic language understanding” to trusted health NLP.

A fine-tuned GPT-OSS-20B can help:

- Enhance clinical reasoning by grounding outputs in evidence-based patterns.
- Reduce errors and hallucinations in sensitive domains like diagnosis and treatment recommendations.
- Support practitioners with structured insights from unstructured records, research articles, and medical - dialogues.
- Scale responsibly, ensuring that AI assistants can adapt to the complexities of healthcare workflows rather than offering one-size-fits-all answers.

In short, the need is clear: healthcare requires not just language fluency but deep, domain-specific reasoning. Fine-tuning a powerful open-source foundation model like gpt-oss-20B is a step toward building medical AI systems that are both scalable and safe.

## Supervised Fine-Tuning: How It Works and Why It Matters

Supervised fine-tuning (SFT) is the key step that turns a large, general model into a healthcare-ready reasoning engine. Instead of retraining all 20B parameters of gpt-oss-20B, we guide the model with curated input–output examples from medical datasets, showing it not just what to answer, but how to reason.

The process can be broken down into four steps:

- Curated data: high-quality medical Q&A, reasoning chains, and clinical text.
- Efficient fine-tuning: use LoRA adapters to update only a small slice of parameters while retaining the base model’s reasoning depth.
- Validation: test against medical benchmarks to ensure safer, more accurate outputs.
- Deployment: integrate into healthcare NLP pipelines, medical education tools, or knowledge retrieval systems, always as an assistive system with human oversight.

Why does this matter? Because healthcare AI cannot afford generic, surface-level responses. With SFT, the model learns to follow trusted reasoning patterns, reduces hallucinations, and adapts to the language of clinicians and researchers. In short, it’s what makes a 20B-parameter model not just big, but useful for medicine.

## Requirements

The training stack was built with the latest open-source tooling optimized for large-scale models:

```txt
torch>=2.0.0
transformers==4.56.0.dev0 @ git+https://github.com/huggingface/transformers.git ## Development version for GPT-OSS-20B support
datasets>=2.12.0
peft>=0.10.0
accelerate>=0.22.0
bitsandbytes>=0.41.0
sentencepiece>=0.1.99
```
This ensured compatibility with 20B-parameter scale models while also enabling mixed-precision and parameter-efficient fine-tuning.

## Dataset and Prompt Formatting

We fine-tuned on the FreedomIntelligence/medical-o1-reasoning-SFT dataset, which provides medical questions, chain-of-thought (CoT) reasoning, and final answers.

Each training sample was formatted as:

```txt
Question: <question>
Reasoning: <Complex_CoT>
Answer: <response>
```

This structure explicitly teaches the model to reason step-by-step before producing an answer, an essential feature for medical tasks where transparency and interpretability matter.

## Training Parameters Explained

The fine-tuning was performed on TIR with 2× H100 GPUs, using the following hyperparameters:

```python
    # Training arguments
    training_args = TrainingArguments(
        output_dir="./medical_gpt_oss_20b",
        per_device_train_batch_size=1,    
        gradient_accumulation_steps=8,    
        num_train_epochs=3,
        learning_rate=5e-5,
        warmup_steps=500,                        
        fp16=False,                              
        bf16=True,                              
        save_strategy="steps",
        save_steps=100,
        logging_steps=10,
        optim="adamw_torch",
        max_grad_norm=1.0,                      
    )
```
### Notes:

- Batch size = 1 + gradient accumulation: Large models can’t fit large batches in memory. Gradient accumulation makes training feasible.
- bf16 precision: Best suited for H100/H200 GPUs, balancing speed and numerical stability.
- Learning rate (5e-5): Optimized for LoRA adapters, preventing catastrophic forgetting.
- Warmup (500 steps): Smooths the training curve and avoids unstable early updates.
- Gradient clipping: Ensures stable convergence at large scale.

## LoRA for Parameter-Efficient Fine-Tuning

Instead of retraining 20B parameters, we fine-tuned with LoRA adapters:

- Target modules: q_proj, v_proj
- LoRA rank (r): 16
- LoRA alpha: 32
- Dropout: 0.05

This approach reduced compute and memory costs drastically, while still aligning the model to medical reasoning tasks.

## Compute Infrastructure

Training a 20B parameter model is non-trivial. With the TIR platform and multi-core NVIDIA H100/H200 GPUs, we achieved an optimal balance of performance and cost.

- Mixed precision (bf16): Reduced memory usage and increased throughput.
- 2× H100 GPUs: Provided enough parallelism to finish training in ~12 hours.
- 2048-token sequences: Supported long-context reasoning for medical scenarios.

Without parameter-efficient fine-tuning and modern GPUs, this scale of training would have been impractical.

## Testing the Fine-Tuned Model: Generating a Sample Answer

After fine-tuning, the next step is to validate the model’s reasoning ability by running inference on clinical-style prompts. Below is an example that loads the GPT-OSS-20B base model with the LoRA adapter we trained, and generates a sample response for a medical scenario.

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import PeftModel
import torch


# Load the base GPT-OSS-20B model
base_model = AutoModelForCausalLM.from_pretrained(
    "openai/gpt-oss-20b",
    device_map="auto",
    torch_dtype=torch.bfloat16,
    trust_remote_code=True
)


# Load the fine-tuned LoRA adapter
model = PeftModel.from_pretrained(base_model, "./medical_gpt_oss_20b_final")


# Load tokenizer
tokenizer = AutoTokenizer.from_pretrained("./medical_gpt_oss_20b_final", trust_remote_code=True)


# Example medical query
prompt = "Patient shows persistent cough, chest tightness, and low-grade fever. Suggest possible diagnoses."
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)


# Generate an answer
with torch.no_grad():
    output = model.generate(**inputs, max_new_tokens=1024)
    print(tokenizer.decode(output[0], skip_special_tokens=True))

```
## Understanding the Code

- Loads the base model + LoRA adapter: This ensures the model retains its general reasoning ability while applying the domain-specific fine-tuning.
- Uses bfloat16 precision: Optimized for H100/H200 GPUs, reducing memory usage while maintaining stability.
- Runs inference with a clinical-style prompt: The output is a reasoning-driven response, showing potential diagnoses based on the symptoms provided.

This step is critical in verifying whether the fine-tuned model produces outputs that are relevant, coherent, and medically aligned before integrating it into larger applications (e.g., a chatbot, RAG pipeline, or decision support tool).

## Key Takeaways from Training

- Base Model: openai/gpt-oss-20b
- Training Duration: ~12 hours on 2× NVIDIA H100 GPUs
- Frameworks & Tools: PyTorch, Hugging Face Transformers (v4.56.0.dev0), and PEFT for parameter-efficient fine-tuning
- Resulting Output: LoRA-adapted GPT-OSS-20B specialized for medical reasoning tasks
- Access the Model: [arunimas1107/gpt-oss-medical](https://huggingface.co/arunimas1107/gpt-oss-medical)

## AI in Medical Reasoning: Key Applications

The fine-tuned Medical GPT-OSS-20B model unlocks a wide range of possibilities across the healthcare and education domains. By combining its large-scale reasoning capabilities with domain-specific knowledge, it can be applied in several impactful ways:

- Medical Q&A: The model can answer clinical and educational queries, providing reasoning-driven explanations that support both practitioners and learners.
- Summarization: It can condense lengthy and complex medical literature into concise, digestible insights, helpful for staying updated with fast-moving research.
- Educational Tools: With its ability to generate step-by-step reasoning, the model can serve as a valuable tutor for medical students, breaking down concepts into clear explanations.
- RAG Pipelines: The model integrates smoothly with retrieval-augmented generation (RAG) frameworks, where it can be paired with specialized medical databases to deliver more accurate and contextually grounded outputs.

Important Note: This model is intended as an assistive tool. It should never be used as a standalone diagnostic system or for direct clinical decision-making without qualified human oversight.