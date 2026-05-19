# EC2 Instance Connect Endpoint (EICE) POC

SSH into a private EC2 instance with no bastion, no persistent key, no public IP. Compare with SSM Session Manager.

## Goal

Reach a private-subnet EC2 instance via `aws ec2-instance-connect ssh`, then reach the same instance via `aws ssm start-session`. Demonstrate both side-by-side.

## Prerequisites

- A VPC with at least one private subnet
- An EC2 instance in that private subnet (Amazon Linux 2023 / Ubuntu — must have `ec2-instance-connect` package and SSM Agent)
- IAM permissions: `ec2-instance-connect:OpenTunnel`, `ec2:DescribeInstances`, `ec2:DescribeInstanceConnectEndpoints`, `ssm:StartSession`, `ssm:TerminateSession`
- Instance role: `AmazonSSMManagedInstanceCore` (for the SSM path)

## Steps

### Setup EICE

1. Create the endpoint in the private subnet:
   ```bash
   aws ec2 create-instance-connect-endpoint \
     --subnet-id subnet-xxxxxxxx \
     --security-group-ids sg-xxxxxxxx
   ```

   The SG attached to the endpoint must be allowed by the target instance SG on port 22.

2. Wait for `State=available`:
   ```bash
   aws ec2 describe-instance-connect-endpoints
   ```

### Path A — EC2 Instance Connect Endpoint

```bash
aws ec2-instance-connect ssh --instance-id i-xxxxxxxx
```

- One command. No key file. No bastion. Connection is tunneled through the EICE.
- Ephemeral key is pushed via `ec2-instance-connect:SendSSHPublicKey` for ~60s.

### Path B — SSM Session Manager

```bash
aws ssm start-session --target i-xxxxxxxx
```

- No SSH at all. Connection brokered through SSM Agent over the SSM service endpoint.
- Requires SSM Agent running and IAM instance profile.

## Comparison

| Aspect              | EICE                          | SSM Session Manager           |
|---------------------|-------------------------------|-------------------------------|
| Protocol            | SSH (real)                    | SSM channel (not SSH)         |
| Agent required      | sshd                          | SSM Agent                     |
| IAM model           | `ec2-instance-connect:*`      | `ssm:StartSession`            |
| Port forwarding     | Native SSH `-L`               | `AWS-StartPortForwardingSession` |
| Audit               | CloudTrail (limited payload)  | CloudTrail + Session logs to S3/CW |
| File copy           | `scp` works                   | Needs port-forward or S3 hop  |
| Network path        | VPC endpoint (private)        | AWS public service endpoint   |
| Cost                | Hourly per endpoint           | Free                          |

## Cleanup

```bash
aws ec2 delete-instance-connect-endpoint --instance-connect-endpoint-id eice-xxxxxxxx
```

## Insight

EICE wins when you need a real SSH transport (scp, agent forwarding, native tooling). SSM wins when you want zero open ports, no inbound SG rules, and rich session auditing. Most orgs end up with both: SSM for ops, EICE for engineers who actually need SSH semantics.

## References

- EICE docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eice.html
- SSM Session Manager: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html
