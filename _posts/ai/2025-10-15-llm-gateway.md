---
layout: post
title: "LLM Gateway: One Endpoint for Safety, Cost Control, and Observability"
date: 2025-10-15
categories: ai
published: true
description: "A clear and practical introduction to the concept of an LLM Gateway—what it is, why we need it, how a fixed response format and a simple policy.yaml enable secure, cost-efficient, and traceable AI usage."
tags: [LLM, Gateway, Policy, Cost Control, Observability, LLM Gateway, AI Gateway, Policy.yaml, Cost Cap, PII Redaction, Model Routing, /v1/run]
---

# LLM Gateway: One Endpoint for Safety, Cost Control, and Observability

> This post is written for newcomers who want to **understand and build an LLM Gateway quickly**.

> We’ll walk through one clear idea: **a single HTTP endpoint `/v1/run` plus a configuration file `policy.yaml`** that brings safety, cost management, and observability to your LLM workflow.

## 1. Why Do We Need an “LLM Gateway”?
**What’s an LLM?**  
A Large Language Model (GPT, Claude, Llama, etc.) that takes text as input and returns text as output.

**The problem:** calling these models directly in production can cause headaches:  
1. **Privacy risks**: emails or phone numbers left visible in prompts
2. **Cost explosions**: long inputs, expensive models    
3. **Unstable performance**: latency spikes or timeouts
4. **No visibility**: no unified logging or traceability

**The solution: an LLM Gateway**  
It’s a lightweight layer sitting between your business apps and the LLMs.

Just like an **AI front desk**, before sending a request to a model, it will:  
- **Ensure safety:** redact PII (Personally Identifiable Information), enforce max input length, check data region.  
- **Control cost:** apply daily budgets, automatically choose cheaper or faster models.
- **Enable observability:** generate `requestId` and `traceUrl` for every call.
- **Manage routing:** dynamically switch providers based on latency or price.

> **The LLM generates answers. The LLM Gateway keeps those answers safe, affordable, and traceable.**

---

## 2. The Fixed Response Format: Why It Matters
Every call to `/v1/run` should return the same, predictable JSON response.  
```json
{
  "requestId": "req_123",
  "output": { "summary": "..." },
  "metrics": {
    "latency_ms": 812,
    "provider": "openai/gpt-4.1-mini",
    "cost_per_1k": 1.9
  },
  "traceUrl": "http://localhost:3000/trace/req_123"
}
```

| Field | Meaning | Why it matters |
|-------|----------|----------------|
| **requestId** | Unique ID for each request | Enables troubleshooting and cost tracking |
| **output.summary** | The model’s main result | Upstream systems can parse one consistent field |
| **metrics** | Latency and cost info | Useful for monitoring and automatic routing |
| **traceUrl** | Link to a detailed trace | Allows debugging and transparency |

**Why a fixed response?**  
Even if you switch from GPT to Claude, or change providers and APIs, the **contract stays stable**. Your upstream applications don’t need any code changes.

That’s the essence of a **Stable Contract**: one interface, many interchangeable providers.

---

## 3. `policy.yaml`: The Heart of the Gateway
The gateway’s logic doesn’t live in code. It lives in a policy file. This makes it easy to **adjust strategy without redeploying**.

### Example policy
```yaml
name: finance
region: ca-central-1

providers:
  routing:
    - when: "latency_p95 > 2000 or cost_per_1k > 2.5"
      to: "anthropic/claude-3-5"
    - default: "openai/gpt-4.1-mini"

guardrails:
  pii_redaction: true
  max_context_tokens: 12000
  cost_cap_usd_per_day: 100

observability: { trace: true }
billing: { split_by: ["dept","project"] }
```

### Field explanation

| Section | Key | Purpose | Example or effect |
|----------|-----|----------|------------------|
| **Basic info** | `name` / `region` | Logical name and allowed region | If the incoming request’s region doesn’t match, it will be rejected. |
| **Routing** | `providers.routing` | Dynamic provider switching | If latency or cost exceeds threshold, do switch to backup model |
| **Guardrails** | `pii_redaction` | Enable email/phone masking | Protect privacy before sending data to providers |
| | `max_context_tokens` | Limit input length | Automatically truncate long prompts |
| **Cost control** | `cost_cap_usd_per_day` | Daily budget cap | Stop or downgrade when total spend exceeds limit |
| **Observability** | `trace` | Record detailed traces | Access traces via `traceUrl` |
| **Billing** | `split_by` | Cost attribution dimensions | Split spend by department or project |


### Upgrading Without Breaking Anything
Policies make upgrades **safe and invisible** to upstream systems.  
Example, You want to:  
- Switch from Claude to GPT-4.1-mini, and 
- Raise the daily budget from $100 to $300.

Simply update:

```yaml
providers:
  routing:
    - default: "openai/gpt-4.1-mini"
guardrails:
  cost_cap_usd_per_day: 300
```
Restart or reload the gateway, and you’re done. No code changes, no downtime, and your clients still call `/v1/run` the same way. That’s the beauty of **decoupling policy from implementation**.

---

## 4. A Practical Example: From Input to Redacted Output
Imagine a **finance report generator** that sends data to GPT for summarization. The prompt contains personal info, emails and phone numbers, that must not leave your region.

- Auto-redact private data before sending to the LLM  
- Generate a summary  
- Log all cost and latency details

### The request

```bash
curl -s http://localhost:3000/v1/run   -H "Content-Type: application/json"   -H "x-region: ca-central-1"   -H "x-billing-dept: finance"   -H "x-billing-project: daily-report"   -d '{
    "input": {
      "prompt": "Please summarize our Q3 ARR and contact alice@example.com or call +1-416-555-1234 for details."
    },
    "tags": { "usecase": "financial-summary" }
  }'
```

### The response

```json
{
  "requestId": "req_ab12cd",
  "output": {
    "summary": "(openai/gpt-4.1-mini) Summary: Please summarize our Q3 ARR and contact [REDACTED_EMAIL], [REDACTED_PHONE]..."
  },
  "metrics": {
    "latency_ms": 645,
    "provider": "openai/gpt-4.1-mini",
    "cost_per_1k": 1.9
  },
  "traceUrl": "http://localhost:3000/trace/req_ab12cd"
}
```

### What happened behind the scenes
1. The gateway verified region compliance (`ca-central-1`)  
2. Detected PII and redacted it before sending to the model  
3. Selected the provider based on routing rules  
4. Checked cost limits before execution  
5. Logged latency, cost, and trace details  

As a result, the prompt is safe, the response consistent, and the call fully traceable.

---

## 5. Common Questions
### 1. Why use one unified `/v1/run` endpoint?
Because it decouples all your business systems from model details. Every app calls the same API. The gateway handles authentication, retries, routing, and observability.  

### 2. How does automatic downgrade or routing work?
The gateway continuously tracks provider metrics like, 
- `latency_p95`: 95th percentile latency
- `cost_per_1k`: average cost per 1,000 tokens  

Then it evaluates routing rules, for example,
```yaml
when: "latency_p95 > 2000 or cost_per_1k > 2.5"
to: "anthropic/claude-3-5"
```
If OpenAI becomes too slow or expensive, traffic automatically switches to Claude. When performance stabilizes, it switches back. 

### 3. How does PII redaction work?
PII (Personally Identifiable Information) includes emails, phone numbers, IDs, etc. When redaction is enabled, the gateway replaces them before sending the prompt out.
```
alice@example.com → [REDACTED_EMAIL]  
+1-416-555-1234  → [REDACTED_PHONE]
```

In production, you can use more advanced libraries like **Microsoft Presidio** or **AWS Comprehend PII**, but the principle remains the same.

### 4. How is the cost cap enforced?
Before calling a model, the gateway  
1. Estimates token count from the input  
2. Multiplies by the provider’s price  
3. Checks if adding this would exceed `cost_cap_usd_per_day`  

If so, it either blocks the request or switches to a cheaper model.

For Example
```yaml
cost_cap_usd_per_day: 100
```
Once daily spend exceeds $100, any new request gets,
```json
{"error": "Daily cost cap reached"}
```

This simple mechanism prevents runaway costs and keeps budgets predictable.

### 5. What’s the purpose of billing tags?
When multiple teams share one gateway, you need cost attribution. You can pass tags in headers.
```
x-billing-dept: finance  
x-billing-project: daily-report
```

These tags are recorded in each trace, so monthly reports show,

| Department | Project | Requests | Cost (USD) |
|-------------|----------|-----------|-------------|
| Finance | daily-report | 250 | 12.6 |

This makes cost allocation and internal accounting much easier.

---

## 6. Summary
> **An LLM Gateway is your AI firewall.**  
> It doesn’t generate text, but it ensures every generation is **safe, cost-effective, and auditable**.

With it, you can achieve
- Security and compliance  
- Cost governance  
- Auto failover and routing  
- Full observability  
- Zero-change upgrades for upstream systems

---

## Related Resources
- [Stable Names + Task Contracts: Keep Callers Stable While You Evolve Internals](/ai/2025/10/12/stable-names-task-contracts-json-schema.html)  
- [The Reuse Layer: Governing AI as a Capability, Not a Credential](/ai/2025/10/07/capability-pool-simple-guide.html) 
