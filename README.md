# git init

This project demonstrates how to run Ansible playbooks within Kubernetes using a Job. It provides a way to execute Ansible automation tasks in a containerized environment.

## Project Structure

```
.
├── README.md
├── ansible-playbook.yaml      # Main Kubernetes manifest
├── playbooks/
│   └── site.yml              # Ansible playbook
└── inventory/
    └── hosts                 # Ansible inventory file
```

## Components

1. **Kubernetes ConfigMaps**:
   - `ansible-playbooks`: Contains the Ansible playbook content
   - `ansible-inventory`: Contains the Ansible inventory configuration

2. **Kubernetes Job**:
   - Uses `alpine/ansible` image
   - Mounts ConfigMaps as volumes
   - Executes the Ansible playbook

## Prerequisites

- Kubernetes cluster
- kubectl configured to communicate with your cluster

## Usage

1. Apply the Kubernetes manifest:
   ```bash
   kubectl apply -f ansible-playbook.yaml
   ```

2. Check the job status:
   ```bash
   kubectl get jobs
   ```

3. View the job logs:
   ```bash
   kubectl logs job/ansible-job
   ```

## Ansible Playbook Details

The included playbook performs the following tasks:
- Prints a hello message
- Creates a test file at `/tmp/ansible-test.txt`
- Writes execution timestamp to the file

## Notes

- The playbook runs with `connection: local` and `gather_facts: false`
- Host key checking is disabled for simplicity
- The job has a backoff limit of 3 retries

## Cleanup

To remove the resources:
```bash
kubectl delete -f ansible-playbook.yaml
``` 