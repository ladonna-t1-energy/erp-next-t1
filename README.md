# Security + AWS Architecture Review

Date: 2026-04-09

## Scope and limitation

- Attempted to clone and inspect `https://github.com/frappe/erpnext.git`, but outbound GitHub access is blocked in this environment (`CONNECT tunnel failed, response 403`).
- This review therefore analyzes the code currently present in this repository and provides architecture guidance for both this repo's Lambda workflow and an ERPNext deployment target.

## Critical cybersecurity concerns in this repository

### 1) Secrets and tokens appear committed in-repo (**Critical**)

- The repo contains credential-bearing files (`client_secret.json`, `youtube_token.pkl`, and `youtube_token.b64.txt`) in the project root.
- The README also documents decoding and use of an OAuth pickle token from source control.
- Exposure impact: account takeover of YouTube uploader identity, abuse of OAuth refresh tokens, and potential lateral movement if credentials are reused.

**Immediate actions**
- Remove these files from git history (not only latest commit), rotate OAuth client secret + tokens, and re-authenticate.
- Enforce secret scanning in CI and pre-commit.
- Use runtime retrieval only from AWS Secrets Manager / SSM and never store serialized tokens in the repository.

### 2) Unsafe deserialization via `pickle.load` on secret data (**Critical**)

- `youtube_uploader.py` decodes a base64 secret then executes `pickle.load(...)`.
- If Secrets Manager content is modified (accident, compromised IAM principal, or pipeline tampering), unpickling can execute arbitrary Python code during Lambda runtime.

**Immediate actions**
- Replace pickle with a non-executable format (JSON) and reconstruct credential objects safely.
- Tighten IAM policy so only Lambda execution role can read that exact secret ARN.
- Add secret integrity checks (versioning + expected schema validation).

### 3) Unverified third-party binary download in Docker build (**High**)

- Dockerfile downloads FFmpeg over HTTPS and extracts it without checksum/signature validation.
- A compromised mirror/connection or unexpected artifact swap becomes a supply-chain risk in your runtime image.

**Immediate actions**
- Pin to exact artifact digest and verify SHA256 before extraction.
- Prefer a trusted package source or immutable artifact in your own artifact registry.

### 4) Network calls without explicit timeout boundaries (**High**)

- `voiceover.py` uses `requests.post(...)` without a timeout.
- Under upstream latency/failure, Lambda invocations can hang and increase cost / denial-of-wallet risk.

**Immediate actions**
- Set strict connect/read timeouts, retries with jitter, and total budget per stage.
- Add dead-letter handling and idempotency keys for retry-safe operation.

## AWS architecture fit: intermediate cloud required?

## For this repository's current workflow

- You can **directly connect to your existing AWS architecture** (Lambda + EventBridge; API Gateway optional).
- No separate "intermediate cloud" is required.

### Recommended integration pattern
- **EventBridge schedule** invokes Lambda every 4 hours for automated batch runs.
- **API Gateway -> Lambda** only if you need manual/adhoc trigger, status endpoint, or operational controls.
- **Secrets Manager** stores OpenAI/ElevenLabs/YouTube credentials.
- **CloudWatch Logs + alarms** for failures/timeouts.
- Optional: **SQS buffer** and split pipeline into multiple Lambdas if processing grows beyond timeout/storage limits.

## If your true target is ERPNext (the linked GitHub project)

- ERPNext is a stateful web app stack (Python app + workers + DB + cache); it is **not a natural fit** for Lambda-only hosting.
- In practice, you would run ERPNext on **ECS/EKS/EC2** and integrate with AWS managed services (RDS + ElastiCache + S3/CloudFront, etc.).
- You can still use API Gateway/EventBridge around ERPNext-adjacent workflows, but those services are not a full hosting replacement for ERPNext core.

## Priority remediation checklist

1. Purge committed secrets/tokens from history and rotate immediately.
2. Remove `pickle`-based credential loading and migrate to safe serialization.
3. Add timeout/retry/idempotency policies for all external API calls.
4. Verify downloaded build artifacts (FFmpeg) via pinned checksums/signatures.
5. Add IAM least-privilege + CloudWatch alerting for secret access and Lambda failures.
