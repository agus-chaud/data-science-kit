# Chapter 10: AI Engineering Architecture and User Feedback

## Core Idea
Build AI applications by starting with the simplest architecture (query → model API → response) and adding components only as needs arise — context construction, guardrails, router/gateway, caches, agent patterns, observability, orchestration. Then close the loop: the conversational interface makes user feedback a proprietary data source that powers the data flywheel, but extracting signal from conversations requires deliberate feedback design (p. 778).

## Frameworks Introduced
- **Progressive AI architecture build-up** (p. 779): start with query → Model API → response, then add components in the order production teams commonly follow:
  1. **Enhance context** — retrieval (text, image, tabular) and tools (web search, news, weather APIs); "context construction is like feature engineering for foundation models" (p. 780)
  2. **Put in guardrails** — input guardrails (PII leak prevention, prompt-attack defense) and output guardrails (quality + security failure catching, failure policies) (p. 781–787)
  3. **Add model router and gateway** — router = intent classifier / next-action predictor; model gateway = unified, secure interface to all models (p. 788–794)
  4. **Reduce latency with caches** — exact caching and semantic caching at the system level (KV/prompt caching live in the model API layer, Ch 9) (p. 795–798)
  5. **Add agent patterns** — loops feeding outputs back in, and write actions (compose email, place order) with utmost care (p. 799–800)
  - Then: monitoring/observability (integral, not an afterthought) and orchestration (chaining it all) (p. 801, 813)
  - When to use: follow the order that makes sense for your app; this is the common production progression, not a mandate (p. 780).
- **Observability quality metrics — MTTD / MTTR / CFR** (p. 802): mean time to detection, mean time to response, change failure rate. If you don't know your CFR, redesign your platform to be more observable.
- **Conversational feedback taxonomy** (p. 818–827): explicit vs implicit feedback, with a new class — **conversational feedback**: natural language feedback (early termination, error correction, complaints, sentiment, model refusal rate) plus action-based signals (regeneration, conversation organization, conversation length, dialogue diversity).
- **Feedback design: when + how to collect** (p. 829–836): collect at signup (calibration), when something bad happens, and when the model has low confidence; collection must be nonintrusive, in-workflow, easy to ignore, and honest about how data is used.

## Key Concepts
- **model gateway**: intermediate layer giving a unified, secure interface to all models (self-hosted + commercial APIs); centralizes access control, cost management, fallback policies, logging (p. 790–793).
- **model router**: component (usually an intent classifier or next-action predictor) that sends each query to the optimal solution — cheaper model, specialized model, FAQ page, or human (p. 788).
- **guardrails**: input protections (sensitive-data detection, prompt-attack defense) and output protections (catch failures, specify handling policy); placed wherever there is risk exposure (p. 781).
- **exact caching**: reuse cached results only for identical requests; needs an eviction policy (LRU, LFU, FIFO) (p. 795–796).
- **semantic caching**: reuse cached results for semantically similar queries via embeddings + vector search + similarity threshold; higher hit rate but fragile (p. 797–798).
- **observability**: instrumenting a system so internal state can be inferred from external outputs — logs and metrics let you debug without shipping new code; monitoring is just the tracking part (p. 804).
- **trace**: detailed recording of a request's execution path through all components — actions taken, documents retrieved, final prompt, time and cost per step (p. 810).
- **data flywheel**: user feedback is proprietary data; a product that launches early gathers data to keep improving models, making it hard for competitors to catch up (p. 817).
- **conversational feedback**: feedback blended into daily dialogue — easier for users to give, harder for developers to extract (p. 778, 818).
- **degenerate feedback loop**: predictions influence feedback, which influences the next model iteration, amplifying initial biases (exposure/popularity bias, filter bubbles, sycophancy) (p. 845).

## Mental Models
- Think of context construction as feature engineering for foundation models — it is the main lever on output quality (p. 780).
- Use a router before an expensive model call: an intent classifier can decline out-of-scope queries with a stock response "without wasting an API call" (p. 788). Routers must be fast and cheap because you'll stack several.
- Use parallel redundant calls when retry latency is unacceptable: send the query twice at once and pick the better response, trading cost for latency (p. 785).
- Treat every user edit as a preference pair: original generation = losing response, edited version = winning response — ready-made data for preference finetuning (p. 822).
- Design metrics backward from failure modes: decide which failures you must catch (hallucination, cost burn), then design metrics that detect them — metrics are not the goal (p. 805).
- Debug with the metrics → logs → correlate loop: metrics tell you *when* something broke, logs tell you *what* happened, correlation confirms you found the right issue (p. 808).

## Anti-patterns
- **Skipping output guardrails for streaming latency**: partial responses are hard to evaluate, so unsafe content can reach users before it's blocked — know the reliability vs latency trade-off you're making (p. 786).
- **Caching user-specific or time-sensitive queries**: a cached answer built on user X's membership data can leak to user Y asking the same "generic" question (p. 796).
- **Adopting semantic caching by default**: it depends on high-quality embeddings, functional vector search, and a tuned similarity threshold — every part is prone to failure; evaluate hit rate vs risk first (p. 798).
- **Jumping straight to an orchestrator**: it abstracts away critical details and makes the system hard to debug; build without one first, adopt later (p. 815).
- **Asking users for impossible feedback**: don't make users choose between two answers to a factual/math question — the right answer isn't a preference; offer "I don't know" (p. 840).
- **Making feedback expensive**: extra work triggers leniency bias — users pick the positive option just to avoid a follow-up form (p. 843).
- **Training indiscriminately on user feedback**: models trained on human feedback tend toward sycophancy — telling users what they want to hear (Sharma et al., 2023) (p. 845–846).
- **Ignoring the false refusal rate**: an over-secure system that blocks legitimate requests frustrates users; track refusals alongside security failures (p. 784).

## Code Examples
```python
import openai

def openai_model(input_data, model_name, max_tokens):
    openai.api_key = os.environ["OPENAI_API_KEY"]
    response = openai.Completion.create(
        engine=model_name, prompt=input_data, max_tokens=max_tokens)
    return {"response": response.choices[0].text}

@app.route('/model', methods=['POST'])
def model_gateway():
    data = request.get_json()
    model_type = data.get("model_type")
    if model_type == "openai":
        result = openai_model(...)
    elif model_type == "gemini":
        result = gemini_model(...)
```
- **What it demonstrates**: a model gateway in its simplest form is a unified wrapper routing requests to different providers behind one endpoint (p. 792). Off-the-shelf options: Portkey AI Gateway, MLflow AI Gateway, Wealthsimple LLM Gateway, TrueFoundry, Kong, Cloudflare (p. 794).

## Reference Tables

**The five architecture steps** (p. 779–800):

| Step | Component added | Problem it solves | Key risk introduced |
|---|---|---|---|
| 1 | Context construction (RAG + tools) | Model lacks needed information | Provider limits on docs/tools differ (p. 780) |
| 2 | Input/output guardrails | PII leaks, prompt attacks, bad outputs | Latency; streaming hard to guard (p. 786) |
| 3 | Router + model gateway | Multi-model cost/complexity, access control | Context limits vary across routed models (p. 789) |
| 4 | Exact + semantic caches | Latency and cost | Data leaks via cache; semantic false hits (p. 796–798) |
| 5 | Agent patterns + write actions | Complex loops, real-world actions | Vastly more failure modes and risk (p. 800) |

**Natural language feedback signals to track in production** (p. 820–825): early termination; error correction ("No, …", "I meant, …", rephrasing); action-correcting feedback ("You should also check…"); confirmation requests ("Are you sure?" — signals distrust or missing detail); direct user edits (strong signal + preference data); complaints; sentiment trajectory; model refusal rate.

**FITS dataset feedback clusters** (Xu et al., 2022) (p. 823–824): clarify demand again (26%); complain bot didn't answer/irrelevant (16%); point to search results that answer it (16%); suggest bot use search results (15%); factually incorrect/not grounded (11%); not specific/complete (9%); bot unconfident ("I'm not sure") (4%); repetition/rudeness (<1%).

**Feedback biases** (p. 842–845): leniency bias (positive to avoid extra work; Uber drivers averaged 4.8/5, <4.6 risked deactivation); randomness (unmotivated users click at random); position bias (first option gets clicked — mitigate by randomizing positions); preference bias (longer answers win comparisons even when less accurate; recency bias favors the last-seen answer).

**Latency/cost metrics to monitor** (p. 807): TTFT, TPOT, total latency (per user); tokens per second, input/output token volume, requests per second vs rate limits.

## Worked Example
**Midjourney's implicit feedback design** (p. 836–837). For each prompt, Midjourney generates four images and offers three options:
1. Upscale one image → strongest positive signal for that image.
2. Generate variations of one image → weaker positive signal.
3. Regenerate → none of the four was good enough (though users may regenerate out of curiosity — noise).

Every option is a normal workflow action, yet each yields a distinct-strength training signal without ever asking "rate this." Contrast with GitHub Copilot (p. 838): drafts render in lighter color; Tab = accept, keep typing = reject — feedback is a byproduct of use. The lesson for standalone chat apps (ChatGPT, Claude): they sit outside the user's workflow, so they can't observe whether the generated email was actually sent — integrated products collect fundamentally better feedback (p. 838–839). If you need conversation context around explicit feedback (previous 5–10 turns), get consent — e.g., a data-donation checkbox at feedback submission (p. 839).

**Hotel example — conversational preference extraction** (p. 818): the assistant recommends three Sydney hotels ($400 Rocks boutique, $200 Surry Hills near galleries, $300 Bondi beachside). Reply "Yes book me the one close to galleries" → reveals art interest (personalization signal). Reply "Is there nothing under $200?" → price-conscious preference *and* the assistant hasn't understood the user yet (evaluation signal). Same feedback stream serves evaluation, development (training data), and personalization (p. 819).

## Key Takeaways
1. Start with the simplest query→model→response architecture and add components only when a concrete need appears; each addition brings capability and new failure modes (p. 779, 847).
2. Component boundaries are fluid — guardrails can live in the inference service, the model gateway, or standalone; some gateways also do caching and guardrails (p. 787, 793, 847).
3. Mask PII before it leaves your org (placeholder + reverse PII map to unmask responses); block or scrub, never send raw (p. 782–783).
4. Route simple queries to cheap models and out-of-scope ones to stock responses; routing–retrieval–generation–scoring is the most common pipeline pattern (p. 788–789).
5. Log everything with tags/IDs (configs, sampling settings, prompts, tool calls, intermediate outputs) and trace each query step-by-step so failures can be pinpointed; inspect production data manually daily (p. 809–810).
6. Watch for silent drift: system prompt changes, user behavior adaptation, and undisclosed underlying model updates (GPT-3.5-turbo version swap cost Voiceflow 10% performance) (p. 811–812).
7. Feedback design is now an engineering responsibility, not just product — it's the data source for the flywheel; but audit for biases and degenerate feedback loops before training on it (p. 846–848).

## Connects To
- **Ch 1**: Crawl-Walk-Run automation levels, business metrics, and the ship-product-first workflow culminate in this architecture and feedback loop.
- **Ch 3**: AI judges become monitoring metrics; judge biases (position, verbosity) mirror the human feedback biases cataloged here.
- **Ch 4**: the quality failures output guardrails must catch are ch04's criteria; drift detection extends the evaluation pipeline into production.
- **Ch 5**: prompt attacks and the system-level defenses that input guardrails implement; false refusal rate tracked alongside violations.
- **Ch 6**: RAG and agentic patterns are architecture Steps 1 and 5; write actions carry the trust/approval requirements set there.
- **Ch 8**: user feedback feeds the data flywheel; user edits arrive as (losing, winning) preference pairs for finetuning data.
- **Ch 9**: inference-level caching (KV, prompt) vs this chapter's exact/semantic caches; TTFT/TPOT latency metrics; inference servers sit behind the model gateway.
- **DevOps/SRE practice**: MTTD/MTTR/CFR come from the DevOps community; observability vs monitoring distinction (p. 802–804).
