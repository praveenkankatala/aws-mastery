# 🔧 Troubleshooting — Amazon Bedrock & Guardrails

> Real error messages, what actually causes them, and how to fix them.
> [← Back to README](README.md) · [Cheat sheet →](commands-cheatsheet.md) · [Labs →](hands-on-labs.md)

---

## 🚑 Start Here — 60-Second Triage

Run this before anything else. It catches roughly 80% of problems.

```bash
#!/usr/bin/env bash
echo "── Identity ────────────────────────────"
aws sts get-caller-identity || echo "❌ credentials broken"

echo "── Tooling ─────────────────────────────"
aws --version
python3 -c "import boto3,botocore; print('boto3',boto3.__version__,'botocore',botocore.__version__)" 2>/dev/null

echo "── Region ──────────────────────────────"
echo "AWS_REGION=${AWS_REGION:-<unset>}  configured=$(aws configure get region)"

echo "── Model reachability ──────────────────"
aws bedrock list-foundation-models --query 'length(modelSummaries)' 2>&1 | head -3

echo "── Model access ────────────────────────"
aws bedrock-runtime converse --model-id "${MODEL_ID:-us.amazon.nova-lite-v1:0}" \
  --messages '[{"role":"user","content":[{"text":"ping"}]}]' \
  --inference-config '{"maxTokens":5}' >/dev/null 2>&1 \
  && echo "✅ model callable" || echo "❌ model NOT callable — check Model access page"

echo "── Guardrails ──────────────────────────"
aws bedrock list-guardrails --query 'guardrails[].{name:name,id:id,v:version,status:status}' --output table 2>&1 | head -12
```

### The Five Usual Suspects

| Symptom | First thing to check |
|---|---|
| `AccessDeniedException` | Model access not granted in this region |
| `Invalid choice: 'converse'` / unknown parameter | Old AWS CLI or boto3 |
| `ValidationException` about on-demand throughput | You need the inference-profile model ID (`us.` prefix) |
| Guardrail doesn't block | You're testing the wrong **version** |
| `ResourceNotFoundException` | Wrong region, or guardrail ID vs ARN confusion |

---

## 📑 Index

**Access & Permissions:** [1](#1-accessdeniedexception-on-invoke) · [2](#2-accessdeniedexception-on-guardrail-operations) · [3](#3-kms-accessdenied)
**Requests & Validation:** [4](#4-validationexception--on-demand-throughput-isnt-supported) · [5](#5-validationexception--malformed-input-request) · [6](#6-standard-tier--cross-region-errors) · [7](#7-prompt_attack-output-strength-error)
**Guardrail Behaviour:** [8](#8-guardrail-not-triggering-when-it-should) · [9](#9-guardrail-blocking-too-much-false-positives) · [10](#10-pii-not-detected) · [11](#11-contextual-grounding-always-blocks-or-never-blocks) · [12](#12-guardrail_intervened-but-nothing-looks-wrong)
**Resources & Versions:** [13](#13-resourcenotfoundexception) · [14](#14-update-guardrail-wiped-my-config) · [15](#15-conflictexception--resource-in-use)
**Runtime & Scale:** [16](#16-throttlingexception) · [17](#17-timeouts-and-read-timeout) · [18](#18-modelstreamerrorexception--streaming-cuts-off) · [19](#19-latency-too-high)
**Tooling:** [20](#20-unknown-parameter--invalid-choice) · [21](#21-cli-body-encoding-errors) · [22](#22-json-parsing-and-quoting-hell)
**RAG & Agents:** [23](#23-knowledge-base-returns-nothing) · [24](#24-rag-answers-are-wrong-despite-good-retrieval) · [25](#25-agent-issues)
**Ops:** [26](#26-logs-are-empty) · [27](#27-terraform-issues) · [28](#28-unexpected-costs) · [29](#29-expiredtokenexception--credential-issues) · [30](#30-region--availability-problems)

---

# Access & Permissions

## 1. AccessDeniedException on invoke

```
An error occurred (AccessDeniedException) when calling the Converse operation:
You don't have access to the model with the specified model ID.
```

**The #1 first-day error.** Three distinct causes, in likelihood order:

### Cause A — Model access not requested
Bedrock models are opt-in per account **per region**.

```bash
# Fix: Console → Bedrock → Model access → Modify model access → tick → Submit
# Verify:
aws bedrock list-foundation-models --query 'modelSummaries[?modelId==`amazon.nova-lite-v1:0`]'
```
If the model is listed but you still can't call it, access isn't granted — listing shows the catalogue, not your entitlement.

### Cause B — Wrong region
```bash
aws configure get region
echo $AWS_REGION
# Model access granted in us-east-1 does NOT apply in ap-south-1.
```

### Cause C — IAM missing the action
```bash
aws iam simulate-principal-policy \
  --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
  --action-names bedrock:Converse bedrock:InvokeModel \
  --resource-arns "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0" \
  --query 'EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}'
```
Minimum needed: `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:Converse`, `bedrock:ConverseStream`.

> ⚠️ **Special case:** if you set up the enforcement policy from [Lab 10](hands-on-labs.md#lab-10--iam-policy-based-enforcement), calls **without** `guardrailConfig` are denied by design. Check whether the deny is your own policy working correctly:
> ```bash
> aws iam list-role-policies --role-name <your-role>
> ```

---

## 2. AccessDeniedException on guardrail operations

```
User: arn:aws:iam::111122223333:user/dev is not authorized to perform:
bedrock:CreateGuardrail on resource: arn:aws:bedrock:us-east-1:111122223333:guardrail/*
```

Guardrail management permissions are **separate** from inference permissions.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:CreateGuardrail", "bedrock:CreateGuardrailVersion",
      "bedrock:GetGuardrail", "bedrock:ListGuardrails",
      "bedrock:UpdateGuardrail", "bedrock:DeleteGuardrail",
      "bedrock:ApplyGuardrail",
      "bedrock:TagResource", "bedrock:UntagResource", "bedrock:ListTagsForResource"
    ],
    "Resource": "*"
  }]
}
```

> 💡 `bedrock:ApplyGuardrail` is a **runtime** action but is authorised against the **guardrail ARN**, not the model ARN. Easy to miss when writing least-privilege policies.

---

## 3. KMS AccessDenied

```
An error occurred (AccessDeniedException): The ciphertext refers to a customer master key
that does not exist, does not exist in this region, or you are not allowed to access.
```

If your guardrail uses a customer-managed KMS key, every principal reading it needs key permissions.

```bash
aws kms get-key-policy --key-id <key-id> --policy-name default --output text | jq
```

Add to the key policy:
```json
{
  "Sid": "AllowBedrockGuardrailUse",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::111122223333:role/MyAppRole" },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
  "Resource": "*"
}
```
Also confirm the key is in the **same region** as the guardrail.

---

# Requests & Validation

## 4. ValidationException — on-demand throughput isn't supported

```
Invocation of model ID anthropic.claude-sonnet-4-5-20250929-v1:0 with on-demand
throughput isn't supported. Retry your request with the ID or ARN of an inference profile
that contains this model.
```

**Cause:** newer models are only reachable through a **cross-region inference profile**, not the bare model ID.

**Fix:** add the geographic prefix.

```bash
aws bedrock list-inference-profiles \
  --query 'inferenceProfileSummaries[].{ID:inferenceProfileId,Name:inferenceProfileName}' --output table
```

| Wrong | Right |
|---|---|
| `amazon.nova-lite-v1:0` | `us.amazon.nova-lite-v1:0` |
| `anthropic.claude-…` | `us.anthropic.claude-…` |
| — | `eu.…` / `apac.…` for those geographies |

```bash
export MODEL_ID=us.amazon.nova-lite-v1:0
```

---

## 5. ValidationException — malformed input request

```
An error occurred (ValidationException) when calling the InvokeModel operation:
Malformed input request: #: extraneous key [max_tokens] is not permitted, please reformat your input and try again.
```

**Cause:** you sent one model vendor's body format to a different vendor's model. `InvokeModel` bodies are **vendor-specific**.

**Fix (best):** switch to `Converse`, which is identical across all models.
```bash
aws bedrock-runtime converse --model-id $MODEL_ID \
  --messages '[{"role":"user","content":[{"text":"hi"}]}]'
```

**Fix (if you must use InvokeModel):** match the vendor schema.

| Vendor | Key fields |
|---|---|
| Amazon Nova | `messages`, `inferenceConfig.maxTokens` |
| Anthropic Claude | `anthropic_version`, `max_tokens`, `messages` |
| Meta Llama | `prompt`, `max_gen_len` |
| Mistral | `prompt`, `max_tokens` |
| Cohere | `message`, `max_tokens` |

Other common validation messages:

| Message | Cause | Fix |
|---|---|---|
| `messages: at least 1 message is required` | Empty array or bad JSON quoting | Build with `jq -nc` |
| `The value at messages.0.content is invalid` | Content must be an *array of blocks* | `"content":[{"text":"…"}]` |
| `Input is too long for requested model` | Exceeded context window | Truncate history, chunk input |
| `Conversation blocks must alternate` | Two consecutive `user` messages | Fix history construction |
| `topicsConfig: exceeds max 30` | Too many denied topics | Consolidate |
| `examples: exceeds max 5` | Too many examples per topic | Trim to 5 |
| `threshold must be between 0 and 0.99` | Used `1.0` | Use `0.99` max |

---

## 6. Standard tier / cross-region errors

```
ValidationException: The STANDARD tier requires cross-region inference to be configured
for this guardrail.
```
or
```
ValidationException: Unknown parameter in input: "tierConfig"
```

### If `tierConfig` is unknown
Your CLI/boto3 is too old:
```bash
pip install --upgrade boto3 botocore
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip \
  && unzip -q awscliv2.zip && sudo ./aws/install --update
```

### If Standard requires cross-region config
Add the guardrail profile:
```json
"crossRegionConfig": { "guardrailProfileIdentifier": "us.guardrail.v1:0" }
```
Use the profile matching your geography (`us.`, `eu.`, `apac.`).

### If your region doesn't support Standard yet
Fall back to Classic — remove both `tierConfig` blocks and `crossRegionConfig`:
```bash
jq 'del(.contentPolicyConfig.tierConfig, .topicPolicyConfig.tierConfig, .crossRegionConfig)' \
  guardrail-config.json > guardrail-classic.json
aws bedrock create-guardrail --cli-input-json file://guardrail-classic.json
```

> ⚠️ **Trade-off you're accepting:** Classic covers English, French and Spanish only, and has limited prompt-attack detection. If your users write in other languages, Classic will silently under-detect. Test with real multilingual samples before accepting this fallback.

---

## 7. PROMPT_ATTACK output strength error

```
ValidationException: The outputStrength for PROMPT_ATTACK filter must be NONE.
```

Prompt-attack detection applies to **input only** — a model's own output can't be a prompt injection against itself.

```json
// ❌ Wrong
{ "type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "HIGH" }

// ✅ Right
{ "type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "NONE" }
```

Terraform:
```hcl
filters_config {
  type            = "PROMPT_ATTACK"
  input_strength  = "HIGH"
  output_strength = "NONE"
}
```
This is also why `dynamic` blocks over a filter list usually need `PROMPT_ATTACK` handled separately.

---

# Guardrail Behaviour

## 8. Guardrail not triggering when it should

**No error — it just lets things through.** Work down this list in order.

### Check 1 — Are you testing the right version?
```bash
aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version DRAFT \
  --source INPUT --content '[{"text":{"text":"test prompt"}}]' --query 'action' --output text
aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version 1 \
  --source INPUT --content '[{"text":{"text":"test prompt"}}]' --query 'action' --output text
```
**By far the most common cause.** You updated `DRAFT` but your app calls version `1`. Versions are immutable — `update-guardrail` never touches a published version.

**Fix:**
```bash
aws bedrock create-guardrail-version --guardrail-identifier $GR_ID --description "v2 with the new topic"
# then update your app to reference version 2
```

### Check 2 — Is the policy actually enabled for that direction?
```bash
aws bedrock get-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --query '{filters:contentPolicy.filters,topics:topicPolicy.topics[].name,pii:sensitiveInformationPolicy.piiEntities[].type}'
```
A filter with `inputStrength: NONE` doesn't run on input. A topic with `inputEnabled: false` doesn't evaluate input.

### Check 3 — Is the filter strength too low?
`LOW` is genuinely permissive. Try `HIGH` for the failing category and re-test.

### Check 4 — Is your topic definition too narrow?
```json
// ❌ Too narrow — matches almost nothing
{ "name": "Investment", "definition": "stocks", "examples": ["stocks"] }

// ✅ Descriptive, with varied examples
{
  "name": "InvestmentAdvice",
  "definition": "Requests for personalised recommendations about buying, selling or holding financial instruments, or about portfolio allocation and expected returns.",
  "examples": ["Should I buy Tesla stock?", "Which fund gives the best return?",
               "Is crypto a good investment?", "How should I allocate my 401k?"]
}
```
Definition quality matters far more than example count.

### Check 5 — Is it a language the Classic tier doesn't cover?
Classic covers English, French and Spanish. Non-English input may pass straight through.
**Fix:** switch to the Standard tier (see [#6](#6-standard-tier--cross-region-errors)).

### Check 6 — Are you missing input tagging?
Without `guardContent` qualifiers, the guardrail can't distinguish your trusted system prompt from untrusted user text, which weakens prompt-attack detection.
```bash
--messages '[{"role":"user","content":[
  {"guardContent":{"text":{"text":"USER TEXT HERE","qualifiers":["guard_content"]}}}]}]'
```

### Check 7 — Is it evasion?
Word filters are **exact match**. `C0mpetitorBank` will not match `CompetitorBank`.
**Fix:** cover semantic intent with a denied topic; use word filters only for exact strings.

---

## 9. Guardrail blocking too much (false positives)

Users complain the bot refuses ordinary questions. **This is as serious as under-blocking** — a bot that refuses real customers gets switched off.

### Step 1 — Find out what's firing
```bash
aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --source INPUT --content '[{"text":{"text":"I want to invest in your fixed deposit"}}]' \
  --query 'assessments[0]' | jq
```

### Step 2 — Fix by cause

| Firing policy | Fix |
|---|---|
| **Denied topic** | Add an explicit exclusion to the definition: *"Excludes questions about our own deposit products, rates and account features."* |
| **Content filter** | Drop that category from `HIGH` to `MEDIUM`. `MISCONDUCT` at `HIGH` over-fires on security, fraud-prevention and compliance topics |
| **Word filter** | Remove overly generic words. Blocking `credit` breaks a banking bot |
| **PII** | Switch `NAME`/`EMAIL` from `BLOCK` to `ANONYMIZE` — customers legitimately give their name |
| **Grounding** | Lower the threshold, or improve retrieval so real answers score higher |

### Step 3 — Measure, don't guess
```bash
cat > benign.txt <<'EOF'
What are your savings account rates?
I want to open a fixed deposit
How do I dispute a transaction?
What documents do I need for a loan?
Can I add a nominee to my account?
What are your branch timings?
How do I update my registered mobile number?
Is there a fee for international transfers?
EOF

blocked=0; total=0
while IFS= read -r p; do
  ((total++))
  a=$(aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID \
      --guardrail-version $GR_VER --source INPUT \
      --content "$(jq -nc --arg t "$p" '[{text:{text:$t}}]')" --query 'action' --output text)
  [[ "$a" == "GUARDRAIL_INTERVENED" ]] && { ((blocked++)); echo "❌ FP: $p"; }
done < benign.txt
echo "False positive rate: $blocked/$total"
```

> 🎯 **Target:** under 2% on legitimate traffic. Above 5% and users will route around your product — which is worse than no guardrail, because you've also lost the telemetry.

---

## 10. PII not detected

### Cause A — The entity type isn't enabled
```bash
aws bedrock get-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --query 'sensitiveInformationPolicy.piiEntities[].type'
```
Only the types you list are detected. There is no "detect everything" switch.

### Cause B — Unusual formatting
Detection is pattern-based. These commonly slip through:
- Spelled out: *"one two three, four five, six seven eight nine"*
- Spaced or hyphenated unusually: `123 45 6789`
- Split across turns
- Embedded in base64 or a URL

**Fix:** add a custom regex for your known formats.
```json
{ "name": "SSNVariants", "description": "SSN with any separator",
  "pattern": "[0-9]{3}[- .]?[0-9]{2}[- .]?[0-9]{4}", "action": "BLOCK" }
```

### Cause C — Regional format not covered
`US_SOCIAL_SECURITY_NUMBER` won't catch an Aadhaar, PAN, or other national ID. Write a regex.
```json
{ "name": "PAN", "description": "Indian PAN", "pattern": "[A-Z]{5}[0-9]{4}[A-Z]", "action": "BLOCK" }
{ "name": "Aadhaar", "description": "12-digit ID", "pattern": "[0-9]{4}[ -]?[0-9]{4}[ -]?[0-9]{4}", "action": "BLOCK" }
```

### Cause D — Testing the wrong direction
PII in a model response is only caught if the policy runs on `OUTPUT`. Test both:
```bash
for src in INPUT OUTPUT; do
  echo -n "$src: "
  aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
    --source $src --content '[{"text":{"text":"SSN 123-45-6789"}}]' --query 'action' --output text
done
```

> ⚠️ **Be honest with your stakeholders:** PII detection is probabilistic. It is strong defence-in-depth and a weak sole control. If a regulator asks whether PII *can* leak, the answer is "we reduce the probability substantially," not "no."

---

## 11. Contextual grounding always blocks (or never blocks)

### Always blocks
**Cause A — No grounding source supplied.** Without a source, there's nothing to be grounded against, so everything scores low.
```bash
# ❌ Missing the source
--content '[{"text":{"text":"the answer"}}]'

# ✅ Source + query + answer
--content '[
  {"text":{"text":"REFERENCE DOCUMENTS","qualifiers":["grounding_source"]}},
  {"text":{"text":"THE USER QUESTION","qualifiers":["query"]}},
  {"text":{"text":"THE MODEL ANSWER"}}
]'
```

**Cause B — Threshold too high.** `0.95` demands near-verbatim support. Reasonable summarisation fails it.

**Cause C — Retrieval is bad.** If the chunks don't contain the answer, no threshold saves you. Fix retrieval first:
```bash
aws bedrock-agent-runtime retrieve --knowledge-base-id $KB_ID \
  --retrieval-query '{"text":"your question"}' \
  --query 'retrievalResults[].{score:score,text:content.text}'
```
If the top chunks don't contain the answer, that's your real bug.

### Never blocks
**Cause A — Threshold too low.** `0.3` passes almost anything.
**Cause B — Policy not configured on the version you're using.**
```bash
aws bedrock get-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --query 'contextualGroundingPolicy'
```
**Cause C — You're checking `INPUT`.** Grounding only applies to `OUTPUT`.

### Choose the threshold from data
```bash
for th in 0.3 0.5 0.7 0.9; do
  echo "threshold $th → observe scores below"
done
# Use the sweep script from Lab 8 to print actual GROUNDING/RELEVANCE scores,
# then set the threshold in the gap between your worst-good and best-bad answer.
```

---

## 12. GUARDRAIL_INTERVENED but nothing looks wrong

**Cause:** `ANONYMIZE` also returns `GUARDRAIL_INTERVENED`. The request wasn't rejected — text was masked.

```bash
aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --source INPUT --content '[{"text":{"text":"Hi, I am Priya, email priya@example.com"}}]' \
  --query '{action:action,output:outputs[0].text}'
```
```json
{ "action": "GUARDRAIL_INTERVENED", "output": "Hi, I am {NAME}, email {EMAIL}" }
```

**Fix your application logic** — don't treat `GUARDRAIL_INTERVENED` as binary:

```python
r = rt.apply_guardrail(...)
if r["action"] == "GUARDRAIL_INTERVENED":
    assess = r.get("assessments", [{}])[0]
    pii = assess.get("sensitiveInformationPolicy", {})
    only_masked = (
        bool(pii)
        and all(e.get("action") == "ANONYMIZED"
                for e in pii.get("piiEntities", []) + pii.get("regexes", []))
        and not any(k for k in assess
                    if k in ("topicPolicy", "contentPolicy", "wordPolicy",
                             "contextualGroundingPolicy"))
    )
    if only_masked:
        text = r["outputs"][0]["text"]      # continue with masked text
    else:
        text = r["outputs"][0]["text"]      # this is your blocked message
        refused = True
```

---

# Resources & Versions

## 13. ResourceNotFoundException

```
An error occurred (ResourceNotFoundException) when calling the ApplyGuardrail operation:
Could not find guardrail with id abcd1234efgh
```

| Cause | Check |
|---|---|
| **Wrong region** | `aws bedrock list-guardrails --region us-east-1` vs your actual region |
| **ID vs ARN confusion** | Both work, but not a *truncated* ARN. Use the bare ID or the full ARN |
| **Version doesn't exist** | `aws bedrock list-guardrails --guardrail-identifier $GR_ID --query 'guardrails[].version'` |
| **Deleted** | Check CloudTrail for `DeleteGuardrail` |
| **Cross-account** | The guardrail lives in another account; you need the full ARN plus a resource policy |

```bash
# Confirm what actually exists, where
for r in us-east-1 us-west-2 eu-west-1 ap-south-1; do
  n=$(aws bedrock list-guardrails --region $r --query 'length(guardrails)' 2>/dev/null)
  echo "$r: ${n:-n/a}"
done
```

---

## 14. update-guardrail wiped my config

**Cause:** `update-guardrail` is a **full replacement**, not a patch. Any field you omit is removed.

```bash
# ❌ This deletes every other policy
aws bedrock update-guardrail --guardrail-identifier $GR_ID \
  --name my-guardrail --blocked-input-messaging "..." \
  --topic-policy-config '{"topicsConfig":[...]}'
```

**Correct workflow — fetch, modify, send whole:**
```bash
aws bedrock get-guardrail --guardrail-identifier $GR_ID --guardrail-version DRAFT > current.json

# Strip read-only fields the update API rejects
jq 'del(.guardrailId, .guardrailArn, .version, .status, .statusReasons,
        .createdAt, .updatedAt, .ResponseMetadata)' current.json > update.json

# Modify update.json, then:
aws bedrock update-guardrail --guardrail-identifier $GR_ID --cli-input-json file://update.json
```

### Recovery
Published versions are immutable — if you have one, your config is not lost:
```bash
aws bedrock get-guardrail --guardrail-identifier $GR_ID --guardrail-version 1 > recovered.json
jq 'del(.guardrailId,.guardrailArn,.version,.status,.createdAt,.updatedAt)' recovered.json > restore.json
aws bedrock update-guardrail --guardrail-identifier $GR_ID --cli-input-json file://restore.json
```

> ✅ **Prevention:** keep the config in Git and manage it with Terraform. Never hand-edit in the console for anything production.

---

## 15. ConflictException / resource in use

```
ConflictException: Cannot delete guardrail version 1 because it is referenced by an active resource.
```

**Cause:** an agent, knowledge base, or IAM policy still references that guardrail version.

```bash
# Find references
aws bedrock-agent list-agents --query 'agentSummaries[].agentId' --output text
aws iam list-policies --scope Local --query 'Policies[].PolicyName' --output text
grep -rl "$GR_ID" ~/projects/ 2>/dev/null
```

Order of operations: detach from agents/KBs → remove IAM references → delete the version → delete the guardrail.

Also seen:
```
ConflictException: A guardrail with the name 'x' already exists.
```
Guardrail names are unique per account per region. Rename, or update the existing one.

---

# Runtime & Scale

## 16. ThrottlingException

```
An error occurred (ThrottlingException) when calling the Converse operation:
Too many requests, please wait before trying again.
```

### Immediate fix — exponential backoff with jitter
```python
from botocore.config import Config
rt = boto3.client("bedrock-runtime",
                  config=Config(retries={"max_attempts": 8, "mode": "adaptive"}))
```
Adaptive mode adds client-side rate limiting on top of retries — significantly better than `standard` under sustained throttling.

Manual version:
```python
import time, random
from botocore.exceptions import ClientError

def call_with_backoff(fn, *a, **kw):
    for attempt in range(6):
        try:
            return fn(*a, **kw)
        except ClientError as e:
            if e.response["Error"]["Code"] != "ThrottlingException":
                raise
            time.sleep(min(2 ** attempt + random.random(), 30))
    raise RuntimeError("throttled after 6 attempts")
```

### Medium-term
```bash
aws service-quotas list-service-quotas --service-code bedrock \
  --query 'Quotas[?contains(QuotaName,`nova`)].{Name:QuotaName,Value:Value,Code:QuotaCode}' --output table

aws service-quotas request-service-quota-increase \
  --service-code bedrock --quota-code L-XXXXXXXX --desired-value 500
```

### Architectural
| Option | When |
|---|---|
| Cross-region inference profile (`us.` prefix) | Immediate — spreads load across regions |
| Provisioned throughput | Steady high volume, predictable latency needed |
| Batch inference | Non-interactive workloads |
| Request queue (SQS) with a worker pool | Smooths bursty traffic |
| Cache repeated prompts | FAQ answers don't need regenerating |
| Route easy work to a smaller model | Different quota pool, lower cost |

### Watch it
```bash
aws cloudwatch get-metric-statistics --namespace AWS/Bedrock \
  --metric-name InvocationThrottles \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 300 --statistics Sum --output table
```

---

## 17. Timeouts and read timeout

```
botocore.exceptions.ReadTimeoutError: Read timeout on endpoint URL
```
or Lambda: `Task timed out after 3.00 seconds`

| Layer | Fix |
|---|---|
| **boto3** | `Config(read_timeout=120, connect_timeout=10)` |
| **Lambda** | `--timeout 30` (or more) — must exceed the boto3 read timeout budget |
| **API Gateway** | Hard 29-second integration limit — you cannot raise it |
| **ALB** | `idle_timeout.timeout_seconds` |

**If you're near the API Gateway 29s ceiling**, the answer is architectural, not configuration:

1. **Stream** with `converse_stream` — first token arrives in ~1s
2. **Cap `maxTokens`** — long generations are the usual culprit
3. **Use a faster model** for latency-sensitive paths
4. **Go async**: return a job ID immediately, process in a queue, deliver via WebSocket or polling

```python
rt = boto3.client("bedrock-runtime", config=Config(
    read_timeout=120, connect_timeout=10,
    retries={"max_attempts": 3, "mode": "adaptive"}))
```

---

## 18. ModelStreamErrorException / streaming cuts off

```
ModelStreamErrorException: An error occurred while streaming the response.
```

| Cause | Fix |
|---|---|
| Client disconnected mid-stream | Handle disconnects; don't assume the stream completes |
| Guardrail intervened mid-stream | Expected behaviour — handle `guardrail_intervened` in your event loop |
| Network instability | Retry the whole request; you cannot resume a stream |
| Timeout during generation | Raise `read_timeout` |

**Robust stream handling:**
```python
try:
    stream = rt.converse_stream(**kwargs)
    for event in stream["stream"]:
        if "contentBlockDelta" in event:
            yield event["contentBlockDelta"]["delta"]["text"]
        elif "messageStop" in event:
            reason = event["messageStop"]["stopReason"]
            if reason == "guardrail_intervened":
                yield "\n[This response was withheld by policy.]"
        elif "internalServerException" in event:
            log.error("stream server error: %s", event)
            break
        elif "modelStreamErrorException" in event:
            log.error("stream model error: %s", event)
            break
except ClientError as e:
    log.error("stream failed: %s", e)
```

**Guardrails + streaming caveat:** by default output is evaluated in chunks, so a few unsafe tokens can reach the client before an intervention. If that's unacceptable:
```json
"guardrailConfig": { "guardrailIdentifier": "...", "guardrailVersion": "1", "streamProcessingMode": "sync" }
```
Higher latency, stronger guarantee. Pick deliberately.

---

## 19. Latency too high

### Measure first
```bash
for i in 1 2 3 4 5; do
  aws bedrock-runtime converse --model-id $MODEL_ID \
    --messages '[{"role":"user","content":[{"text":"Say hello"}]}]' \
    --inference-config '{"maxTokens":20}' \
    --query 'metrics.latencyMs' --output text
done
```

### Isolate the guardrail's contribution
```bash
echo "without guardrail:"
aws bedrock-runtime converse --model-id $MODEL_ID \
  --messages '[{"role":"user","content":[{"text":"Say hello"}]}]' \
  --inference-config '{"maxTokens":20}' --query 'metrics.latencyMs' --output text

echo "with guardrail:"
aws bedrock-runtime converse --model-id $MODEL_ID \
  --messages '[{"role":"user","content":[{"text":"Say hello"}]}]' \
  --inference-config '{"maxTokens":20}' \
  --guardrail-config "{\"guardrailIdentifier\":\"$GR_ID\",\"guardrailVersion\":\"$GR_VER\"}" \
  --query 'metrics.latencyMs' --output text
```

### Reduce it

| Lever | Impact |
|---|---|
| Disable unused guardrail policies | Each enabled policy adds evaluation time |
| Classic tier instead of Standard | Lower latency, weaker detection — a real trade-off |
| Smaller/faster model | Often the single biggest win |
| Lower `maxTokens` | Generation time scales with output length |
| Shorter system prompt | Fewer input tokens to process |
| Stream | Doesn't reduce total latency, transforms perceived latency |
| Same-region VPC endpoint | Removes internet round-trip |
| Cache | Zero latency for repeats |

---

# Tooling

## 20. Unknown parameter / Invalid choice

```
Unknown parameter in input: "guardrailConfig", must be one of: modelId, body, ...
```
```
Invalid choice: 'converse', maybe you meant: 'invoke-model'
```
```
Invalid choice: 'apply-guardrail'
```

**One cause: your tooling is out of date.** Bedrock ships new parameters constantly.

```bash
aws --version                                    # need v2, recent
pip show boto3 | grep -i version

pip install --upgrade boto3 botocore
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install --update
hash -r      # clear the shell's command cache
aws --version
```

**Lambda note:** the bundled runtime boto3 may lag. Ship your own:
```bash
mkdir -p package && pip install boto3 botocore -t package/
cd package && zip -qr ../function.zip . && cd ..
zip -g function.zip lambda_function.py
aws lambda update-function-code --function-name my-fn --zip-file fileb://function.zip
```
Or attach a Lambda layer with a current boto3.

---

## 21. CLI body encoding errors

```
Invalid base64: "{"messages":[...]}"
```

**Cause:** AWS CLI v2 base64-encodes binary parameters by default.

```bash
# ❌
aws bedrock-runtime invoke-model --model-id $MODEL_ID --body '{"...":"..."}' out.json

# ✅
aws bedrock-runtime invoke-model --model-id $MODEL_ID --body '{"...":"..."}' \
  --cli-binary-format raw-in-base64-out out.json
```

Or set it permanently:
```bash
aws configure set cli_binary_format raw-in-base64-out
```

> 💡 `converse` doesn't have this problem — another reason to prefer it.

---

## 22. JSON parsing and quoting hell

Nested quotes in shell JSON are a reliable source of wasted hours.

### Build JSON with jq, not string concatenation
```bash
# ❌ Breaks on apostrophes, quotes, newlines
aws bedrock-runtime converse --messages "[{\"role\":\"user\",\"content\":[{\"text\":\"$USER_INPUT\"}]}]"

# ✅ Correct escaping, always
MSG=$(jq -nc --arg t "$USER_INPUT" '[{role:"user",content:[{text:$t}]}]')
aws bedrock-runtime converse --model-id $MODEL_ID --messages "$MSG"
```

### Use files for anything complex
```bash
aws bedrock create-guardrail --cli-input-json file://guardrail-config.json
jq empty guardrail-config.json && echo "✅ valid JSON"
```

### Generate the skeleton instead of guessing
```bash
aws bedrock create-guardrail --generate-cli-skeleton > skeleton.json
```

### Watch for smart quotes
Copying JSON from a web page, Word doc or chat often brings `"` `"` instead of `"`:
```bash
sed -i 's/[""]/"/g; s/['"'"''"'"']/'"'"'/g' config.json
jq empty config.json
```

---

# RAG & Agents

## 23. Knowledge Base returns nothing

```bash
aws bedrock-agent-runtime retrieve --knowledge-base-id $KB_ID \
  --retrieval-query '{"text":"test"}' --query 'length(retrievalResults)'
# → 0
```

| Cause | Check / fix |
|---|---|
| **Sync never ran or failed** | `aws bedrock-agent list-ingestion-jobs --knowledge-base-id $KB_ID --data-source-id $DS_ID` — look at `statistics` for documents scanned/indexed/failed |
| **S3 path wrong** | `aws s3 ls s3://bucket/prefix/ --recursive` |
| **Unsupported file types** | Use `.txt`, `.md`, `.pdf`, `.docx`, `.html`, `.csv` |
| **KB service role lacks S3 access** | Check the role's trust and S3 read permissions |
| **Vector store not ready** | `aws opensearchserverless batch-get-collection --ids <id>` → `ACTIVE` |
| **Embedding model access not granted** | Titan Embeddings needs model access too |
| **Query too different from content** | Semantic search still needs conceptual overlap |

```bash
aws bedrock-agent get-ingestion-job --knowledge-base-id $KB_ID \
  --data-source-id $DS_ID --ingestion-job-id $JOB_ID \
  --query '{status:status,stats:statistics,failures:failureReasons}'
```

---

## 24. RAG answers are wrong despite good retrieval

**Diagnose in order — retrieval, then prompt, then model:**

```bash
# 1. Are the right chunks coming back?
aws bedrock-agent-runtime retrieve --knowledge-base-id $KB_ID \
  --retrieval-query '{"text":"YOUR QUESTION"}' \
  --retrieval-configuration '{"vectorSearchConfiguration":{"numberOfResults":10}}' \
  --query 'retrievalResults[].{score:score,snippet:content.text}' | jq -r '.[] | "\(.score)\t\(.snippet[0:110])"'
```

| Symptom | Fix |
|---|---|
| Right chunks, wrong answer | Strengthen the system prompt: *"Answer ONLY from the provided context. If it isn't there, say you don't know."* |
| Chunks cut mid-fact | Adjust chunking strategy — larger chunks or more overlap |
| Relevant chunk ranked low | Increase `numberOfResults`, or add a reranking model |
| Model blends context with its own knowledge | Lower `temperature` to `0.0–0.1` and enable contextual grounding |
| Answer contradicts source | Enable grounding checks — this is exactly what they're for |
| Right answer, wrong question answered | Enable the relevance check |

> 🎯 **Retrieval quality is the ceiling on answer quality.** No amount of prompt engineering or guardrail tuning fixes chunks that don't contain the answer.

---

## 25. Agent issues

| Error / symptom | Cause | Fix |
|---|---|---|
| `Agent is not prepared` | Changes not published | `aws bedrock-agent prepare-agent --agent-id $ID` |
| `Access denied invoking Lambda` | Missing resource policy | `aws lambda add-permission --principal bedrock.amazonaws.com --source-arn <agent-arn>` |
| Agent ignores an action group | Vague description | Rewrite the action-group and parameter descriptions — the model chooses tools by reading them |
| Agent loops | No terminal condition | Add explicit stop instructions; cap session length |
| KB not used | Association disabled | `associate-agent-knowledge-base` with `--knowledge-base-state ENABLED` |
| Trace shows nothing | Trace off | Add `--enable-trace` |

```bash
aws bedrock-agent-runtime invoke-agent \
  --agent-id $AGENT_ID --agent-alias-id $ALIAS_ID \
  --session-id "debug-$(date +%s)" --input-text "test" --enable-trace out.txt
```

> 🧠 **Agents vs Guardrails:** Guardrails governs *what is said*. It does **not** stop an agent from calling a destructive API. For that you need AgentCore Policy (Cedar rules at the gateway) or tightly-scoped IAM on the action-group Lambda.

---

# Ops

## 26. Logs are empty

| Cause | Fix |
|---|---|
| Logging never enabled | `aws bedrock get-model-invocation-logging-configuration` — if empty, enable it |
| Logging role can't write | Role needs `logs:CreateLogStream` + `logs:PutLogEvents` on the log group |
| `textDataDeliveryEnabled: false` | Set to `true` |
| Wrong region | Logs land in the region of the invocation |
| Delivery lag | Wait a few minutes |
| Trust policy wrong | Principal must be `bedrock.amazonaws.com` |

```bash
aws bedrock get-model-invocation-logging-configuration
aws iam get-role --role-name BedrockLoggingRole --query 'Role.AssumeRolePolicyDocument'
aws logs describe-log-streams --log-group-name /aws/bedrock/modelinvocations \
  --order-by LastEventTime --descending --max-items 5
```

**Missing guardrail traces specifically:** you must pass `"trace": "enabled"` per request. It is off by default.

---

## 27. Terraform issues

### `Error: Invalid value for output_strength`
```hcl
filters_config {
  type            = "PROMPT_ATTACK"
  input_strength  = "HIGH"
  output_strength = "NONE"   # must be NONE
}
```

### Guardrail recreated on every apply
Usually a `timestamp()` or similar in an attribute:
```hcl
resource "aws_bedrock_guardrail_version" "v" {
  guardrail_arn = aws_bedrock_guardrail.main.guardrail_arn
  description   = "Managed by Terraform"
  lifecycle { ignore_changes = [description] }
}
```

### Drift after a console edit
```bash
terraform plan                                   # see what changed
terraform apply                                  # revert to code (correct answer)
# or, to adopt the console change:
terraform state rm aws_bedrock_guardrail.main
terraform import aws_bedrock_guardrail.main <guardrail-id>
```

### `Error: creating guardrail: ValidationException`
Almost always a policy-shape problem. Reproduce with the CLI to get a clearer message:
```bash
aws bedrock create-guardrail --cli-input-json file://equivalent-config.json
```

### Provider too old
```hcl
terraform { required_providers { aws = { source = "hashicorp/aws", version = "~> 5.0" } } }
```
```bash
terraform init -upgrade
```

---

## 28. Unexpected costs

### Find the source
```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%F),End=$(date +%F) \
  --granularity DAILY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --query 'ResultsByTime[-7:].{Date:TimePeriod.Start,Groups:Groups}' | jq
```

### Usual culprits, in order

| Culprit | Detection | Fix |
|---|---|---|
| **Provisioned throughput left running** | `aws bedrock list-provisioned-model-throughputs` | Delete it — bills hourly regardless of use |
| **OpenSearch Serverless collection** | `aws opensearchserverless list-collections` | Minimum OCU charge accrues 24/7. Delete when idle |
| **Runaway retry loop** | `OutputTokenCount` metric spike | Cap retries; add a circuit breaker |
| **Huge system prompt** | Compare `inputTokens` to expectation | Trim — you pay for it on every single call |
| **Unbounded conversation history** | `inputTokens` grows over a session | Cap history to N turns |
| **Model too large for the task** | Reading the code | Route classification/routing to a micro model |
| **All six guardrail policies enabled** | `usage` block in the response | Disable ones you don't need |
| **maxTokens too high** | `outputTokens` near the cap | Lower it |

### Guardrail cost visibility
```bash
aws bedrock-runtime apply-guardrail --guardrail-identifier $GR_ID --guardrail-version $GR_VER \
  --source INPUT --content '[{"text":{"text":"test"}}]' --query 'usage'
```
```json
{ "topicPolicyUnits": 1, "contentPolicyUnits": 1, "wordPolicyUnits": 1,
  "sensitiveInformationPolicyUnits": 1, "contextualGroundingPolicyUnits": 0 }
```
Every non-zero line is a billed policy. Zero it by disabling the policy.

### Preventive
```bash
aws budgets create-budget --account-id $ACCOUNT_ID --budget '{
  "BudgetName":"bedrock-monthly","BudgetLimit":{"Amount":"200","Unit":"USD"},
  "TimeUnit":"MONTHLY","BudgetType":"COST",
  "CostFilters":{"Service":["Amazon Bedrock"]}}'

aws cloudwatch put-metric-alarm --alarm-name bedrock-token-burn \
  --namespace AWS/Bedrock --metric-name OutputTokenCount \
  --statistic Sum --period 3600 --evaluation-periods 1 --threshold 1000000 \
  --comparison-operator GreaterThanThreshold --alarm-actions $TOPIC
```

> 💡 Alarm on **tokens**, not just dollars. Billing data lags by hours; token metrics are near real-time.

---

## 29. ExpiredTokenException / credential issues

```
An error occurred (ExpiredTokenException): The security token included in the request is expired.
```

```bash
# SSO
aws sso login --profile my-profile

# Assumed role — re-assume
CREDS=$(aws sts assume-role --role-arn <arn> --role-session-name s --query Credentials)
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .SessionToken)

# Stale env vars overriding your profile? This is a classic.
env | grep -E 'AWS_(ACCESS|SECRET|SESSION|PROFILE)'
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

Other credential errors:

| Error | Fix |
|---|---|
| `Unable to locate credentials` | `aws configure`, or attach an instance/task role |
| `InvalidClientTokenId` | Key deleted or belongs to another account |
| `SignatureDoesNotMatch` | Wrong secret key, or severe clock skew — check NTP |
| `AccessDenied: not authorized to perform sts:AssumeRole` | Role trust policy doesn't include your principal |

---

## 30. Region & availability problems

```
Could not connect to the endpoint URL: "https://bedrock-runtime.<region>.amazonaws.com/"
```

**Cause:** Bedrock isn't available in that region, or the region name is misspelled.

```bash
aws ssm get-parameters-by-path \
  --path /aws/service/global-infrastructure/services/bedrock/regions \
  --query 'Parameters[].Value' --output text | tr '\t' '\n' | sort
```

### Availability varies by three independent axes
1. **Is Bedrock in the region?**
2. **Is the model in that region?** — `aws bedrock list-foundation-models --region <r>`
3. **Is the feature in that region?** — Guardrails Standard tier, Automated Reasoning checks and AgentCore all have narrower footprints than Bedrock itself

```bash
for r in us-east-1 us-west-2 eu-west-1 eu-central-1 ap-south-1 ap-southeast-1; do
  n=$(aws bedrock list-foundation-models --region $r --query 'length(modelSummaries)' 2>/dev/null)
  echo "$r → ${n:-unavailable} models"
done
```

> 💡 **Practical advice:** develop in `us-east-1` or `us-west-2` where coverage is broadest, and validate feature availability in your production region *before* you design around a feature. Data-residency requirements may force a region with narrower support — find that out early, not at go-live.

---

## 🧭 Still Stuck? A Systematic Approach

1. **Reproduce with the smallest possible request.** Strip the guardrail, then the system prompt, then the parameters — until it works. The last thing you removed is the cause.
2. **Compare `DRAFT` against your version.** Different behaviour here explains most "it works locally" reports.
3. **Enable `trace: "enabled"`** and read the assessments. Nearly every guardrail question is answered there.
4. **Check CloudTrail** — someone may have changed the resource:
   ```bash
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
     --max-results 25 --query 'Events[].{Time:EventTime,Event:EventName,User:Username}' --output table
   ```
5. **Test in `us-east-1`.** If it works there and not in your region, it's an availability issue.
6. **Upgrade boto3 and the CLI.** Cheap to do, and it resolves a surprising share of "impossible" bugs.
7. **Get the request ID** and open an AWS Support case:
   ```bash
   aws bedrock-runtime converse --model-id $MODEL_ID \
     --messages '[{"role":"user","content":[{"text":"hi"}]}]' --debug 2>&1 | grep -i 'x-amzn-requestid'
   ```

---

## 📋 Pre-Production Checklist

Before anything customer-facing:

- [ ] Guardrail references a **numbered version**, never `DRAFT`
- [ ] Regression suite passes — violating prompts **and** benign prompts
- [ ] False-positive rate measured on real traffic, under 2%
- [ ] IAM enforcement via `bedrock:GuardrailIdentifier`, with the version pinned
- [ ] Model invocation logging on, log group encrypted, retention set
- [ ] CloudWatch alarms: interventions, throttles, latency, token burn
- [ ] Retries with exponential backoff and jitter (`mode: adaptive`)
- [ ] Timeouts set at every layer (client, Lambda, API Gateway)
- [ ] Rate limiting at the API layer
- [ ] Input length validation before any paid call
- [ ] Conversation history capped
- [ ] Blocked messages are helpful, not hostile
- [ ] Human escalation path exists and is reachable
- [ ] Budget alarm set
- [ ] Guardrail config in Git, deployed via IaC
- [ ] Runbook written: who to call, how to roll back a guardrail version
- [ ] Region availability confirmed for every feature you depend on
- [ ] Logs reviewed for PII exposure — they contain raw prompts

---

## ⚠️ A Closing Note on Expectations

Guardrails meaningfully reduces risk. It does not eliminate it.

- Detection is probabilistic — false positives and false negatives both happen.
- Determined adversaries find gaps; treat safety as an ongoing programme, not a project.
- No automated filter substitutes for professional human judgement in medical, legal, financial or safety-critical contexts.
- Keep a human escalation path, and be honest with stakeholders about what the system does and doesn't guarantee.

Build defence in depth: input validation → rate limiting → guardrails → IAM enforcement → monitoring → human review. Each layer catches what the others miss.

---

[← Back to README](README.md) · [Cheat sheet →](commands-cheatsheet.md) · [Labs →](hands-on-labs.md)
