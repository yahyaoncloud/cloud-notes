# JSON, YAML & Task Definition Wrangling Interview Scenarios

In CI/CD pipelines and GitOps workflows, cloud engineers constantly transform JSON and YAML files—updating ECS task definitions with new container image tags, injecting environment variables into Kubernetes manifests, and validating CloudFormation schemas. This guide covers the high-frequency scripting scenarios asked in technical rounds.

---

## Scenario 1: Dynamic ECS Task Definition Image & Environment Injection

### The Interview Question
"In our GitHub Actions or Jenkins pipeline, we pull the active ECS Task Definition JSON from AWS using `aws ecs describe-task-definition`. Before registering a new revision with `aws ecs register-task-definition`, AWS will reject the payload if it contains metadata fields like `taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, and `registeredAt`.
Write a script (using Python and `jq`) to clean the task definition, update the container image tag to the newly built commit SHA, inject an updated environment variable, and output a valid JSON payload ready for registration."

### Solution A: Using `jq` in Shell / CI Pipelines

```bash
#!/usr/bin/env bash
set -euo pipefail

NEW_IMAGE="123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-service:v2.1.0"
COMMIT_SHA="a1b2c3d4e5f6"
INPUT_FILE="raw-task-def.json"
OUTPUT_FILE="cleaned-task-def.json"

# Clean AWS-generated metadata and inject new image and environment variable
jq --arg img "${NEW_IMAGE}" --arg sha "${COMMIT_SHA}" '
  .taskDefinition |
  # Remove immutable metadata fields injected by AWS describe-task-definition
  del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy) |
  # Update container image for the primary container
  (.containerDefinitions[] | select(.name == "payment-app")).image = $img |
  # Update or append GIT_COMMIT environment variable
  (.containerDefinitions[] | select(.name == "payment-app")).environment = (
    [ .containerDefinitions[] | select(.name == "payment-app") | .environment[] | select(.name != "GIT_COMMIT") ] +
    [ { "name": "GIT_COMMIT", "value": $sha } ]
  )
' "${INPUT_FILE}" > "${OUTPUT_FILE}"

echo "Cleaned task definition written to ${OUTPUT_FILE}."
```

### Solution B: Production Python Script

```python
#!/usr/bin/env python3
"""
ECS Task Definition Cleaner and Mutator.
Filters immutable metadata and updates container image tags.
"""

import json
import sys
import boto3

METADATA_KEYS_TO_REMOVE = {
    'taskDefinitionArn', 'revision', 'status', 'requiresAttributes',
    'compatibilities', 'registeredAt', 'registeredBy'
}

def clean_and_update_task_def(task_family, new_image, commit_sha):
    ecs = boto3.client('ecs', region_name='us-east-1')
    
    # Fetch active task definition
    response = ecs.describe_task_definition(taskDefinition=task_family)
    task_def = response['taskDefinition']

    # Remove AWS runtime metadata
    cleaned = {k: v for k, v in task_def.items() if k not in METADATA_KEYS_TO_REMOVE}

    # Update container image and environment
    for container in cleaned.get('containerDefinitions', []):
        if container.get('name') == 'payment-app':
            container['image'] = new_image
            
            # Filter existing GIT_COMMIT and append new
            env_vars = [env for env in container.get('environment', []) if env['name'] != 'GIT_COMMIT']
            env_vars.append({'name': 'GIT_COMMIT', 'value': commit_sha})
            container['environment'] = env_vars

    # Register new task definition revision
    new_reg = ecs.register_task_definition(**cleaned)
    new_revision = new_reg['taskDefinition']['revision']
    print(f"Successfully registered new task definition revision: {task_family}:{new_revision}")
    return new_revision

if __name__ == '__main__':
    if len(sys.argv) < 4:
        print("Usage: python3 update_task_def.py <family> <image_tag> <commit_sha>")
        sys.exit(1)
    clean_and_update_task_def(sys.argv[1], sys.argv[2], sys.argv[3])
```

---

## Scenario 2: Multi-Document Kubernetes YAML Manifest Validation & Injection

### The Interview Question
"Write a Python script that parses a multi-document Kubernetes YAML file (containing Deployment, Service, and ConfigMap), verifies that all container specs have explicit `resources.limits` and `resources.requests` defined (preventing unbounded node CPU/RAM exhaustion), and injects a standard label `environment: production` across all metadata blocks."

### Production Python Script

```python
#!/usr/bin/env python3
"""
Kubernetes Manifest Guardrail Validator & Label Injector.
Ensures CPU/Memory limits are set and enforces uniform governance labels.
"""

import sys
import yaml

def validate_and_mutate_k8s_manifests(input_file, output_file, env_name="production"):
    with open(input_file, 'r') as f:
        docs = list(yaml.safe_load_all(f))

    modified_docs = []
    violations = []

    for idx, doc in enumerate(docs):
        if not doc:
            continue
        
        kind = doc.get('kind', 'Unknown')
        name = doc.get('metadata', {}).get('name', f'unnamed-{idx}')

        # 1. Inject governance labels
        if 'metadata' not in doc:
            doc['metadata'] = {}
        if 'labels' not in doc['metadata']:
            doc['metadata']['labels'] = {}
        doc['metadata']['labels']['environment'] = env_name
        doc['metadata']['labels']['managed-by'] = 'gitops-pipeline'

        # 2. Inspect containers for resource boundaries in Deployments, StatefulSets, DaemonSets
        if kind in {'Deployment', 'StatefulSet', 'DaemonSet'}:
            spec = doc.get('spec', {}).get('template', {}).get('spec', {})
            containers = spec.get('containers', [])

            for container in containers:
                c_name = container.get('name', 'unnamed-container')
                resources = container.get('resources', {})
                limits = resources.get('limits', {})
                requests = resources.get('requests', {})

                if not limits.get('cpu') or not limits.get('memory'):
                    violations.append(f"Resource Violation in {kind}/{name}: Container '{c_name}' missing limits.cpu or limits.memory.")
                if not requests.get('cpu') or not requests.get('memory'):
                    violations.append(f"Resource Violation in {kind}/{name}: Container '{c_name}' missing requests.cpu or requests.memory.")

        modified_docs.append(doc)

    if violations:
        print("POLICY ENFORCEMENT FAILURES DETECTED:")
        for v in violations:
            print(f"  - {v}")
        print("Pipeline aborted. Please define resource requests and limits.")
        sys.exit(1)

    with open(output_file, 'w') as out_f:
        yaml.dump_all(modified_docs, out_f, default_flow_style=False, sort_keys=False)

    print(f"Validation passed. Mutated manifest written to {output_file}.")

if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: python3 k8s_guardrail.py <input.yaml> <output.yaml>")
        sys.exit(1)
    validate_and_mutate_k8s_manifests(sys.argv[1], sys.argv[2])
```
