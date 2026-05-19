# S3 Conditional Writes POC

Released late 2024. Use `If-None-Match: *` on `aws s3api put-object` to prevent accidental overwrites. Replaces the old DynamoDB-lock pattern for concurrent-writer races.

## Goal

Demonstrate that two consecutive `put-object` calls to the same key — the second one with `--if-none-match "*"` — fail with `PreconditionFailed` when the key already exists.

## Prerequisites

- AWS CLI v2 (recent version with `--if-none-match` support on `put-object`)
- IAM permissions: `s3:CreateBucket`, `s3:PutObject`, `s3:DeleteObject`, `s3:DeleteBucket`
- Region: any standard region (feature is GA in all commercial regions)

## Steps

1. Create a test bucket:
   ```bash
   BUCKET="poc-conditional-writes-$(date +%s)"
   aws s3api create-bucket --bucket "$BUCKET" --region us-east-1
   ```

2. First write succeeds:
   ```bash
   echo "v1" > /tmp/obj.txt
   aws s3api put-object \
     --bucket "$BUCKET" \
     --key file.txt \
     --body /tmp/obj.txt \
     --if-none-match "*"
   ```

3. Second write fails with `PreconditionFailed`:
   ```bash
   echo "v2" > /tmp/obj.txt
   aws s3api put-object \
     --bucket "$BUCKET" \
     --key file.txt \
     --body /tmp/obj.txt \
     --if-none-match "*"
   # Expected: An error occurred (PreconditionFailed) when calling the PutObject operation
   ```

4. Without the flag, the second write would overwrite silently:
   ```bash
   aws s3api put-object --bucket "$BUCKET" --key file.txt --body /tmp/obj.txt
   ```

## Cleanup

```bash
aws s3 rm "s3://$BUCKET" --recursive
aws s3api delete-bucket --bucket "$BUCKET"
```

## Insight

Before this feature, atomic "create-if-not-exists" semantics on S3 required an external lock (DynamoDB conditional write, Redis SETNX). Conditional writes move the guarantee server-side at S3, eliminating an entire class of distributed-lock infrastructure for idempotent uploads, dedup pipelines, and leader-election-style sentinel keys.

## References

- AWS announcement: https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-conditional-writes/
