# Bedrock invoke-model via CLI POC

Call Claude, Titan, or Llama from pure shell via `aws bedrock-runtime invoke-model`. No SDK, no language runtime — useful for pipelines, cron jobs, CI hooks.

## Goal

Issue a prompt to a foundation model and parse the response with only AWS CLI + `jq`.

## Prerequisites

- AWS CLI v2
- Bedrock model access enabled in the region for the model you choose (request access in Bedrock console once)
- IAM permissions: `bedrock:InvokeModel`
- `jq`

## Steps

### 1. List available models (optional)

```bash
aws bedrock list-foundation-models --region us-east-1 \
  --query 'modelSummaries[].modelId' --output table
```

### 2. Invoke Claude (Messages API)

```bash
cat > /tmp/payload.json <<'EOF'
{
  "anthropic_version": "bedrock-2023-05-31",
  "max_tokens": 256,
  "messages": [
    {"role": "user", "content": "Explain S3 Conditional Writes in two sentences."}
  ]
}
EOF

aws bedrock-runtime invoke-model \
  --region us-east-1 \
  --model-id anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --content-type application/json \
  --accept application/json \
  --body fileb:///tmp/payload.json \
  /tmp/response.json

jq -r '.content[0].text' /tmp/response.json
```

### 3. Invoke Titan Text

```bash
cat > /tmp/titan.json <<'EOF'
{
  "inputText": "Write a haiku about IAM policies.",
  "textGenerationConfig": {"maxTokenCount": 100, "temperature": 0.7}
}
EOF

aws bedrock-runtime invoke-model \
  --region us-east-1 \
  --model-id amazon.titan-text-express-v1 \
  --content-type application/json \
  --accept application/json \
  --body fileb:///tmp/titan.json \
  /tmp/titan-out.json

jq -r '.results[0].outputText' /tmp/titan-out.json
```

### 4. Streaming variant

```bash
aws bedrock-runtime invoke-model-with-response-stream \
  --region us-east-1 \
  --model-id anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --content-type application/json \
  --body fileb:///tmp/payload.json \
  /tmp/stream.json
```

## Insight

The `bedrock-runtime` API is just a signed HTTP POST. Once you can hit it from `aws` CLI, you can hit it from `curl` with SigV4, from a Lambda with no extra dependencies, or from a `make` target. Lets you slot generative AI into a shell pipeline (log triage, commit-message drafts, IaC review) without adopting a heavy SDK.

## Cleanup

Nothing to clean — invocations are stateless and billed per token.

## References

- Bedrock Runtime CLI: https://docs.aws.amazon.com/cli/latest/reference/bedrock-runtime/
- Claude on Bedrock body schema: https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages.html
