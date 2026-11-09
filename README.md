# devops-13-lesson: Ansible + AWS EC2 (build/prod)

*Русская версия: [README.md](README.md)*

The `site.yml` playbook deploys two EC2 nodes in AWS:

- **build-node**: builds the Java project (Maven); the resulting JAR is fetched to the control node;
- **prod-node**: receives the JAR and runs it as a systemd service on port 8080.

## Requirements

- Ansible 2.14+
- Python packages `boto3`, `botocore`
- The `amazon.aws` collection
- An AWS account with EC2 permissions (instances, key pairs, security groups)

```bash
pip install boto3 botocore
ansible-galaxy collection install amazon.aws
```

## Quick start (one command)

```bash
AWS_ACCESS_KEY_ID=<your-key-id> AWS_SECRET_ACCESS_KEY=<your-secret> ansible-playbook site.yml
```

## Running

1. Set your AWS credentials:

```bash
export AWS_ACCESS_KEY_ID=<your-key-id>
export AWS_SECRET_ACCESS_KEY=<your-secret>
```

2. (Optional) check the syntax:

```bash
ansible-playbook site.yml --syntax-check
```

3. Run the whole pipeline (provision nodes → build → deploy):

```bash
ansible-playbook site.yml
```

At the end the playbook prints the IP addresses, SSH commands and the application URL (`http://<public-ip>:8080/`).

## Dynamic inventory

There is no static `inventory.ini`. Hosts are discovered in AWS by the `amazon.aws.aws_ec2` plugin
(file [aws_ec2.yml](aws_ec2.yml), enabled in [ansible.cfg](ansible.cfg)).

How it works:
- the plugin looks for running instances tagged `Project=devops-13-lesson`;
- the host name is the `Name` tag (`build-node`, `prod-node`);
- groups are built from the `Role` tag: `build` and `prod`;
- `ansible_host` is the public IP; the user and key are defined in [group_vars/all.yml](group_vars/all.yml).

The playbook itself tags the instances (play `provision`) and then runs
`meta: refresh_inventory`, so the new nodes show up in the inventory within the same run.

Useful commands:

```bash
ansible-inventory --graph          # tree of groups and hosts
ansible-inventory --list           # all host variables
ansible all -m ping                # check SSH access to all nodes
ansible prod -m command -a 'systemctl status java-app'
```

### Changing the region or the project tag

The inventory is parsed before `group_vars`, so these values live in two places and must match:

| What        | Where to change                                                            |
|-------------|----------------------------------------------------------------------------|
| Region      | `aws_region` in `group_vars/all.yml` **and** `regions:` in `aws_ec2.yml`   |
| Project tag | `project_tag` in `group_vars/all.yml` **and** `tag:Project` in `aws_ec2.yml` |

Check that the values match:

```bash
grep -n "aws_region\|project_tag" group_vars/all.yml
grep -n -A1 "regions:\|tag:Project" aws_ec2.yml
```

If the region does not match, `ansible-inventory --graph` shows no hosts, and
`meta: refresh_inventory` will not find the created nodes (the "wait for SSH" step fails).

## Running in parts (tags)

Once the nodes exist, individual steps can be run on their own:

```bash
ansible-playbook site.yml --tags provision   # only create EC2 instances
ansible-playbook site.yml --tags build       # only build and fetch the JAR
ansible-playbook site.yml --tags deploy      # only deploy the JAR to prod
ansible-playbook site.yml --tags build,deploy
```

## Parameters

All settings are in one file: [group_vars/all.yml](group_vars/all.yml). Edit them there
or override with `-e`:

| Variable        | Default            | Description                        |
|-----------------|--------------------|------------------------------------|
| `aws_region`    | `eu-central-1`     | AWS region                         |
| `instance_type` | `t3.small`         | Instance type                      |
| `key_name`      | `devops-13-lesson` | Key pair name (file `<name>.pem`)  |
| `repo_url`      | `gs-spring-boot`   | Git repository with the project    |
| `app_port`      | `8080`             | Application port                   |

Example:

```bash
ansible-playbook site.yml -e instance_type=t3.medium
```

## Finding the VM IP addresses and connecting over SSH

**1. From the playbook output.** At the very end of the run (play `Summary`) the playbook prints the public
and private IPs of both nodes, ready-to-use SSH commands and the application URL. This block is printed
for any tag selection.

**2. Via the dynamic inventory:**

```bash
ansible-inventory --graph
ansible-inventory --host build-node | grep -E 'ansible_host|private_ip_address'
ansible-inventory --host prod-node  | grep -E 'ansible_host|private_ip_address'
```

**3. Via the AWS CLI:**

```bash
aws ec2 describe-instances --region eu-central-1 \
  --filters "Name=tag:Project,Values=devops-13-lesson" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[Tags[?Key=='Name']|[0].Value,PublicIpAddress]" --output table
```

**4. Via the AWS console:** EC2 → Instances → the *Public IPv4 address* column.

### Connecting over SSH

The playbook creates the private key `devops-13-lesson.pem` next to `site.yml`; the user is `ubuntu`:

```bash
chmod 600 devops-13-lesson.pem      # if ssh complains about permissions
ssh -i devops-13-lesson.pem ubuntu@<public-ip>
```

With Ansible you do not need to know the IPs (the key and user come from `group_vars/all.yml`):

```bash
ansible build -m ping
ansible prod -a 'systemctl status java-app'
ansible prod -a 'journalctl -u java-app -n 50 --no-pager' --become
```

Public IPs are dynamic: they change after an instance is stopped and started again.
The inventory looks nodes up by tag, so it picks up the new addresses on its own.

## Verification

```bash
curl http://<prod-public-ip>:8080/
ssh -i devops-13-lesson.pem ubuntu@<prod-public-ip> 'systemctl status java-app'
```

## Cleanup

```bash
ansible-playbook site.yml -e state=absent --tags provision
```

This removes the instances tagged `Project=devops-13-lesson` and the key pair.
The `devops-13-lesson-sg` security group must be deleted manually.

## Notes

- The private key `*.pem` and the `artifacts/` directory are excluded via `.gitignore`.
- Ports 22 and 8080 are open to `0.0.0.0/0`. That is fine for a training lab; for real use,
  restrict access to your own IP.
