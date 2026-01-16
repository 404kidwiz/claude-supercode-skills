# LLM Architect - Examples & Patterns

## Anti-Patterns

### Anti-Pattern: Fine-Tuning When Prompting Would Suffice

**What it looks like:**
```python
# User wants to change response format or style
# Jumps straight to fine-tuning a custom model
train_model(
    base_model="llama2-7b",
    dataset="responses_formatted_nicely.jsonl",
    epochs=3
)
```

**Why it fails:**
- Costs thousands in compute for what good prompting achieves
- Maintenance burden (need to retrain on updates)
- Slower iteration cycles (days vs minutes)

**Correct approach:**
```python
# Try prompt engineering first (90% of cases this works)
prompt = """Output JSON in this exact format:
{
  "answer": "your response here",
  "confidence": 0.95,
  "sources": ["source1", "source2"]
}"""

# Only fine-tune if:
# - Prompting fails after extensive iteration
# - Need <50ms latency (fine-tuned model smaller/faster)
# - Highly domain-specific behavior required
```

---

### Anti-Pattern: No Fallback Strategy

**What it looks like:**
```python
# Single model, no error handling
response = claude.messages.create(
    model="claude-3-opus",
    messages=[{"role": "user", "content": prompt}]
)
# If Claude API is down → your app is down
```

**Why it fails:**
- API outages happen (99.9% uptime = 43 minutes downtime/month)
- Rate limits can be hit unexpectedly
- Single point of failure

**Correct approach:**
```python
import asyncio

async def resilient_llm_call(prompt):
    # Strategy 1: Retry with exponential backoff
    for attempt in range(3):
        try:
            return await call_primary_llm(prompt)
        except RateLimitError:
            await asyncio.sleep(2 ** attempt)  # 1s, 2s, 4s
        except APIError as e:
            logger.warning(f"Attempt {attempt+1} failed: {e}")
    
    # Strategy 2: Fallback to alternative model
    try:
        return await call_fallback_llm(prompt)
    except Exception:
        pass
    
    # Strategy 3: Degrade gracefully
    return {
        "response": "Service temporarily unavailable, queued for processing",
        "queued": True
    }
```

---

### Anti-Pattern: Ignoring Context Window Limits

**What it looks like:**
```python
# Stuffing entire document into prompt
prompt = f"Summarize this: {entire_100k_document}"
# API error: context length exceeded
```

**Why it fails:**
- Models have token limits (4K-200K depending on model)
- Costs increase linearly with input size
- Quality degrades with very long contexts

**Correct approach:**
```python
def chunk_and_summarize(document: str, max_chunk_tokens: int = 4000) -> str:
    chunks = split_into_chunks(document, max_chunk_tokens)
    
    # Map: Summarize each chunk
    summaries = []
    for chunk in chunks:
        summary = await call_llm(f"Summarize this section:\n{chunk}")
        summaries.append(summary)
    
    # Reduce: Combine summaries
    if len(summaries) == 1:
        return summaries[0]
    
    combined = "\n\n".join(summaries)
    return await call_llm(f"Combine these summaries into one:\n{combined}")
```

---

### Anti-Pattern: Treating LLMs as Databases

**What it looks like:**
```python
# Asking model to recall specific facts from training
prompt = "What is customer ID 12345's current order status?"
```

**Why it fails:**
- Models don't have deterministic memory
- Hallucinate plausible-sounding but false information
- Training data is outdated (months to years old)

**Correct approach:**
```python
# Fetch from database, use LLM for synthesis/presentation
order_data = database.query("SELECT * FROM orders WHERE customer_id = 12345")
prompt = f"Summarize this order status for customer: {order_data}"

# LLM transforms data → user-friendly response
# Database provides facts (deterministic, accurate, up-to-date)
# LLM provides presentation (natural language, helpful)
```

---

### Anti-Pattern: No Output Validation

**What it looks like:**
```python
# Trusting LLM output blindly
response = await call_llm("Generate a JSON config")
config = json.loads(response)  # Crashes if invalid JSON
```

**Correct approach:**
```python
import json
from pydantic import BaseModel, ValidationError

class ConfigOutput(BaseModel):
    setting_a: str
    setting_b: int
    enabled: bool

async def get_validated_config(prompt: str) -> ConfigOutput:
    for attempt in range(3):
        response = await call_llm(prompt)
        
        try:
            # Extract JSON from response
            json_match = re.search(r'\{.*\}', response, re.DOTALL)
            if json_match:
                data = json.loads(json_match.group())
                return ConfigOutput(**data)
        except (json.JSONDecodeError, ValidationError) as e:
            # Retry with error feedback
            prompt = f"{prompt}\n\nPrevious attempt failed: {e}. Please fix."
    
    raise ValueError("Failed to get valid output after 3 attempts")
```

---

## Quality Checklist

Use this checklist before marking an LLM system implementation complete:

### Architecture & Design
- [ ] Latency requirements documented and validated (P50, P95, P99)
- [ ] Cost projections calculated for expected traffic ($/1K requests)
- [ ] Model selection justified with trade-off analysis
- [ ] Fallback strategy implemented and tested
- [ ] Scaling strategy defined (horizontal/vertical, triggers)

### Performance
- [ ] Benchmark results on representative dataset (>1000 examples)
- [ ] Accuracy/quality metrics meet minimum thresholds
- [ ] Latency <2x requirement across P95
- [ ] Throughput tested at 2x expected peak load
- [ ] Cache hit rate measured (if caching implemented)

### Cost Optimization
- [ ] Caching strategy implemented and verified
- [ ] Prompt optimization applied (compression, templates)
- [ ] Multi-model routing configured (if applicable)
- [ ] Cost monitoring dashboards created
- [ ] Budget alerts configured (>110% expected spend)

### Safety & Compliance
- [ ] Content filtering tested against adversarial examples
- [ ] PII detection and redaction validated
- [ ] Prompt injection defenses in place
- [ ] Output validation rules implemented
- [ ] Audit logging configured for all requests
- [ ] Compliance requirements documented and validated

### Monitoring & Observability
- [ ] Latency metrics tracked (P50, P95, P99)
- [ ] Cost metrics tracked ($/day, $/1K requests)
- [ ] Quality metrics tracked (accuracy, user ratings)
- [ ] Error rate tracked and alerted (>5% error rate)
- [ ] Dashboards created for stakeholders

### Operational Readiness
- [ ] Runbook documented with common failure scenarios
- [ ] On-call escalation paths defined
- [ ] Rollback procedure tested
- [ ] A/B testing framework configured (if needed)
- [ ] Model versioning strategy implemented

### Documentation
- [ ] Architecture diagram created and reviewed
- [ ] API documentation published (if exposing APIs)
- [ ] Configuration documentation complete
- [ ] Decision log maintained (why this model, why this approach)
- [ ] Known limitations documented

---

## Prompt Engineering Patterns

### Chain of Thought

```python
prompt = """Solve this step by step:

Question: If a train travels at 60 mph for 2.5 hours, how far does it travel?

Let me think through this:
1. First, I'll identify the formula: distance = speed × time
2. Speed = 60 mph
3. Time = 2.5 hours
4. Distance = 60 × 2.5 = 150 miles

The train travels 150 miles.

Now solve this:
Question: {user_question}

Let me think through this:"""
```

### Few-Shot Examples

```python
prompt = """Classify the sentiment of these reviews:

Review: "The product arrived broken and customer service was unhelpful."
Sentiment: Negative

Review: "Exactly what I needed! Fast shipping too."
Sentiment: Positive

Review: "It works okay, nothing special but does the job."
Sentiment: Neutral

Review: "{user_review}"
Sentiment:"""
```

### Structured Output

```python
prompt = """Extract information from this text and return as JSON:

Text: "John Smith, CEO of Acme Corp, announced Q3 revenue of $5.2M"

Output the following JSON structure:
{
  "person": "name of the person mentioned",
  "role": "their job title",
  "company": "company name",
  "metric": "any financial figures mentioned"
}

JSON:"""
```
