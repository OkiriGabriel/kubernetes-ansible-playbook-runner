# Kubernetes Ansible Playbook Runner

Run Ansible playbooks inside Kubernetes. The repo is organized as a playbook: load the Ansible content once, then apply the runner pattern you need.

The included playbooks target `localhost` so every pattern works in a cluster without SSH keys or extra hosts. Point the inventory at real machines when you are ready to manage them.

## Prerequisites

- A Kubernetes cluster
- `kubectl` configured for that cluster
- Optional: Ansible on your laptop if you want to dry-run playbooks locally

## Layout

```
.
├── ansible.cfg
├── inventory/hosts
├── playbooks/
│   ├── site.yml              # smoke test
│   ├── bootstrap.yml         # writes files for an init container
│   ├── maintenance.yml       # scheduled health report
│   └── vars/common.yml
├── playbook-job/             # one-shot Job
├── playbook-cronjob/         # daily CronJob
├── playbook-deployment/      # long-running runner you can exec into
├── playbook-init/            # init container bootstrap
└── pod-script-runner/        # ConfigMap scripts plus an inline Job
```

## 1. Load the playbooks into the cluster

ConfigMaps are the source of truth the runners mount. Re-run these commands after you edit files in `playbooks/` or `inventory/`.

```bash
kubectl create configmap ansible-playbooks \
  --from-file=site.yml=playbooks/site.yml \
  --from-file=bootstrap.yml=playbooks/bootstrap.yml \
  --from-file=maintenance.yml=playbooks/maintenance.yml \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create configmap ansible-playbook-vars \
  --from-file=playbooks/vars \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create configmap ansible-inventory \
  --from-file=hosts=inventory/hosts \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create configmap ansible-config \
  --from-file=ansible.cfg \
  --dry-run=client -o yaml | kubectl apply -f -
```

Confirm they exist:

```bash
kubectl get configmap ansible-playbooks ansible-playbook-vars ansible-inventory ansible-config
```

## 2. Pick a runner pattern

### Pattern A — one-shot Job

Use this for a playbook that should run once and exit.

```bash
kubectl apply -f playbook-job/ansible-job.yaml
kubectl get jobs
kubectl logs job/ansible-job
```

`playbooks/site.yml` prints the runner identity, writes `/tmp/ansible-test.txt`, and asserts the file exists.

### Pattern B — scheduled CronJob

Use this for recurring work. The example runs `playbooks/maintenance.yml` at 02:00 every day and writes `/tmp/maintenance-report.txt`.

```bash
kubectl apply -f playbook-cronjob/ansible-cronjob.yaml
kubectl get cronjobs
```

Trigger a run immediately without waiting for the schedule:

```bash
kubectl create job ansible-maintenance-now --from=cronjob/ansible-maintenance
kubectl logs job/ansible-maintenance-now
```

### Pattern C — long-running Deployment

Use this when you want a pod you can exec into and run playbooks by hand.

```bash
kubectl apply -f playbook-deployment/ansible-deployment.yaml
kubectl get pods -l pattern=long-running-runner
```

Run a playbook inside the runner:

```bash
kubectl exec -it deploy/ansible-runner -- \
  ansible-playbook site.yml -i /inventory/hosts -v
```

### Pattern D — init container

Use this when an application pod should not start until Ansible has prepared shared files.

```bash
kubectl apply -f playbook-init/ansible-init.yaml
kubectl logs ansible-init-demo -c ansible-bootstrap
kubectl logs ansible-init-demo -c app
```

The init container runs `playbooks/bootstrap.yml`, writes `/shared/app.conf` and `/shared/ready`, then the `app` container reads those files from `/app/data`.

### Pattern E — script runner

Use this when the job should run shell or Python helpers and then an Ansible playbook.

```bash
kubectl apply -f pod-script-runner/scripts-configmap.yaml
kubectl apply -f pod-script-runner/script-job.yaml
kubectl logs job/script-runner-job
```

For a Job with no ConfigMap and an inline shell script:

```bash
kubectl apply -f pod-script-runner/inline-script-job.yaml
kubectl logs job/inline-script-job
```

## 3. Target remote hosts later

The sample inventory only enables `localhost`. To manage other machines:

1. Uncomment and edit hosts in `inventory/hosts`.
2. Reload the `ansible-inventory` ConfigMap from step 1.
3. Store an SSH key as a Secret (do not commit the key):

```bash
kubectl create secret generic ansible-ssh-key \
  --from-file=id_rsa="$HOME/.ssh/id_rsa"
```

4. Mount that Secret at `/root/.ssh` on the runner and set `ansible_user` on each host.

## Cleanup

Remove a single pattern:

```bash
kubectl delete -f playbook-job/ansible-job.yaml
kubectl delete -f playbook-cronjob/ansible-cronjob.yaml
kubectl delete -f playbook-deployment/ansible-deployment.yaml
kubectl delete pod ansible-init-demo
kubectl delete -f pod-script-runner/script-job.yaml
kubectl delete -f pod-script-runner/inline-script-job.yaml
kubectl delete -f pod-script-runner/scripts-configmap.yaml
```

Remove the shared ConfigMaps:

```bash
kubectl delete configmap \
  ansible-playbooks \
  ansible-playbook-vars \
  ansible-inventory \
  ansible-config
```

## Notes

- Runners use the `alpine/ansible` image.
- Playbooks gather facts so timestamps and hostnames are real.
- Host key checking is disabled in `ansible.cfg` for the demo. Turn it back on for anything that is not a local smoke test.
- Jobs keep logs for 10 minutes (`ttlSecondsAfterFinished: 600`), then Kubernetes deletes them.
