# S3 Express One Zone POC

S3 Express One Zone is a single-AZ, high-IOPS storage class using *directory buckets*. Designed for sub-10ms first-byte latency on small objects. POC: benchmark vs. standard S3.

## Goal

Upload N small objects to both a Standard bucket and an Express One Zone directory bucket, then compare wall-clock latency.

## Prerequisites

- AWS CLI v2 (recent enough for directory bucket support)
- IAM permissions: `s3express:*`, `s3:*`
- Pick a single AZ (e.g. `use1-az5`) in a supported region (e.g. `us-east-1`)

## Steps

1. Set vars:
   ```bash
   REGION="us-east-1"
   AZ_ID="use1-az5"
   SUFFIX=$(date +%s)
   STD_BUCKET="poc-standard-$SUFFIX"
   EXPRESS_BUCKET="poc-express-$SUFFIX--${AZ_ID}--x-s3"
   ```

   Note: directory bucket name MUST end with `--<az-id>--x-s3`.

2. Create both buckets:
   ```bash
   aws s3api create-bucket --bucket "$STD_BUCKET" --region "$REGION"

   aws s3api create-bucket \
     --bucket "$EXPRESS_BUCKET" \
     --region "$REGION" \
     --create-bucket-configuration "Location={Type=AvailabilityZone,Name=$AZ_ID},Bucket={Type=Directory,DataRedundancy=SingleAvailabilityZone}"
   ```

3. Generate 100 small files:
   ```bash
   mkdir -p /tmp/poc-files
   for i in $(seq 1 100); do
     dd if=/dev/urandom of=/tmp/poc-files/file-$i.bin bs=4K count=1 2>/dev/null
   done
   ```

4. Benchmark Standard:
   ```bash
   time aws s3 cp /tmp/poc-files "s3://$STD_BUCKET/" --recursive
   ```

5. Benchmark Express One Zone:
   ```bash
   time aws s3 cp /tmp/poc-files "s3://$EXPRESS_BUCKET/" --recursive
   ```

6. Compare. Expect noticeably lower latency per request on Express, especially with `--cli-write-timeout 0` and small concurrent objects. Run from an EC2 instance in the same AZ as the directory bucket for the realistic case.

## Cleanup

```bash
aws s3 rm "s3://$STD_BUCKET" --recursive && aws s3api delete-bucket --bucket "$STD_BUCKET"
aws s3 rm "s3://$EXPRESS_BUCKET" --recursive && aws s3api delete-bucket --bucket "$EXPRESS_BUCKET" --region "$REGION"
rm -rf /tmp/poc-files
```

## Insight

Trade-off: single-AZ (no cross-AZ durability) and higher per-GB cost in exchange for ~10x lower request latency. Fits tightly-coupled workloads: ML training shards, checkpointing, hot ephemeral caches. Use authentication is session-based (`CreateSession`) — handled transparently by the CLI.

## References

- AWS docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-express-one-zone.html
