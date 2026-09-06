# infra-prototype

OpenTofu configuration for the Canvas LMS EC2 prototype in AWS `us-west-2`.

## Prerequisites

- OpenTofu
- AWS CLI v2
- An AWS CLI profile configured for IAM Identity Center (SSO)
- An SSH public key (defaults to `~/.ssh/id_ed25519.pub`)

## Configure local variables

Create an ignored `terraform.tfvars` file:

```hcl
# Restrict SSH to your current public IPv4 address.
admin_cidr = "YOUR.PUBLIC.IP/32"

# Optional: override the default public-key path.
ssh_public_key_path = "~/.ssh/id_ed25519.pub"
```

## Authenticate with AWS

List the available profiles and select the one to use:

```bash
aws configure list-profiles
export AWS_PROFILE="your-profile-name"
```

Sign in and verify the target AWS identity:

```bash
aws sso login --profile "$AWS_PROFILE"
aws sts get-caller-identity
```

The exported `AWS_PROFILE` is used by both the AWS CLI and OpenTofu.

## Initialize and validate

```bash
tofu fmt -check
tofu init
tofu validate
tofu plan
```

`tofu plan` is the live validation step: inspect it carefully and confirm that
the selected AWS account and proposed resources are correct.

## Apply

To review and apply in one command with an approval prompt:

```bash
tofu apply
```

To save and then apply the exact reviewed plan:

```bash
tofu plan -out=tfplan
tofu apply tfplan
```

## Connect to the instance

After the apply completes:

```bash
tofu output -raw lms_public_ip
ssh -i ~/.ssh/id_ed25519 admin@$(tofu output -raw lms_public_ip)
```

If `ssh_public_key_path` points to another key, pass its corresponding private
key to `ssh -i` instead.

## Disk resize

In case disk in opentofu file get resize, filesystem resize need to be applied on Debian as well to be able to utlize extra space.

```
sudo apt update
sudo apt install -y cloud-guest-utils
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
```

## Instance Setup

[Install docker](https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script)
[Install Caddy](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)
