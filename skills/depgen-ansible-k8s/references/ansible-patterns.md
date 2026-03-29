# Ansible Patterns — Deployment Automation

Ansible playbook and role patterns for deploying containerized applications to Kubernetes.

## Directory Structure

```
ansible/
├── inventory/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── roles/
│   ├── docker-build/
│   │   └── tasks/main.yml
│   ├── k8s-namespace/
│   │   └── tasks/main.yml
│   ├── k8s-config/
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── configmap.yml.j2
│   │       └── secret.yml.j2
│   ├── k8s-deploy/
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── deployment.yml.j2
│   │       ├── service.yml.j2
│   │       └── ingress.yml.j2
│   ├── k8s-deps/
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       └── (per-dependency manifests)
│   └── k8s-verify/
│       └── tasks/main.yml
├── deploy.yml
└── rollback.yml
```

## Inventory — hosts.yml

```yaml
---
all:
  children:
    k8s_control_plane:
      hosts:
        control-1:
          ansible_host: <control-plane-ip>
          ansible_user: <ssh-user>
    k8s_workers:
      hosts:
        worker-1:
          ansible_host: <worker-1-ip>
          ansible_user: <ssh-user>
        worker-2:
          ansible_host: <worker-2-ip>
          ansible_user: <ssh-user>
```

## Inventory — group_vars/all.yml

```yaml
---
# Application
app_name: <app-name>
app_version: "<version>"
app_namespace: <namespace>

# Container Image
image_registry: <registry-url>
image_name: "{{ image_registry }}/<image-name>"
image_tag: "{{ app_version }}"

# --- ConfigMap Variables ---
server_port: "8080"
mongodb_uri: "mongodb://mongo-svc:27017/<db-name>"
keycloak_server_url: "http://keycloak-svc:8080"
keycloak_realm: "<realm>"
keycloak_issuer_uri: "{{ keycloak_server_url }}/realms/{{ keycloak_realm }}"
keycloak_client_id: "<client-id>"
rabbitmq_host: "rabbitmq-svc"
rabbitmq_port: "5672"
mail_host: "mailcatcher-svc"
mail_port: "1025"
cors_allowed_origins: "https://<app-domain>"
log_level_root: "INFO"
log_level_app: "INFO"
# ... all ConfigMap variables from env var detection

# --- Secret Variables (vault-encrypted) ---
# Encrypt with: ansible-vault encrypt_string '<value>' --name '<var>'
keycloak_admin_username: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
keycloak_admin_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
rabbitmq_username: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
rabbitmq_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...

# --- Dependent Services ---
deploy_dependencies: true          # Set to false if using external/managed services
mongodb_version: "<version>"
keycloak_version: "<version>"
rabbitmq_version: "<version>"

# --- Resource Limits ---
app_cpu_request: "250m"
app_cpu_limit: "1000m"
app_memory_request: "512Mi"
app_memory_limit: "1024Mi"
app_replicas: 2
```

## Role: docker-build

```yaml
# roles/docker-build/tasks/main.yml
---
- name: Build container image
  community.docker.docker_image:
    name: "{{ image_name }}"
    tag: "{{ image_tag }}"
    source: build
    build:
      path: "{{ playbook_dir }}/../"
      dockerfile: Dockerfile
      pull: true
    state: present

- name: Tag image as latest
  community.docker.docker_image:
    name: "{{ image_name }}"
    tag: latest
    source: local
    repository: "{{ image_name }}"
    state: present

- name: Push image to registry
  community.docker.docker_image:
    name: "{{ image_name }}"
    tag: "{{ item }}"
    push: true
    source: local
    state: present
  loop:
    - "{{ image_tag }}"
    - latest
```

## Role: k8s-namespace

```yaml
# roles/k8s-namespace/tasks/main.yml
---
- name: Create namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ app_namespace }}"
        labels:
          app.kubernetes.io/part-of: "{{ app_name }}"
```

## Role: k8s-config

```yaml
# roles/k8s-config/tasks/main.yml
---
- name: Apply ConfigMap
  kubernetes.core.k8s:
    state: present
    template: configmap.yml.j2

- name: Apply Secret
  kubernetes.core.k8s:
    state: present
    template: secret.yml.j2
```

### Template: configmap.yml.j2

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "{{ app_name }}-config"
  namespace: "{{ app_namespace }}"
data:
  SERVER_PORT: "{{ server_port }}"
  MONGODB_URI: "{{ mongodb_uri }}"
  KEYCLOAK_SERVER_URL: "{{ keycloak_server_url }}"
  KEYCLOAK_REALM: "{{ keycloak_realm }}"
  KEYCLOAK_ISSUER_URI: "{{ keycloak_issuer_uri }}"
  KEYCLOAK_CLIENT_ID: "{{ keycloak_client_id }}"
  RABBITMQ_HOST: "{{ rabbitmq_host }}"
  RABBITMQ_PORT: "{{ rabbitmq_port }}"
  MAIL_HOST: "{{ mail_host }}"
  MAIL_PORT: "{{ mail_port }}"
  CORS_ALLOWED_ORIGINS: "{{ cors_allowed_origins }}"
  LOG_LEVEL_ROOT: "{{ log_level_root }}"
  LOG_LEVEL_APP: "{{ log_level_app }}"
  # ... all ConfigMap variables
```

### Template: secret.yml.j2

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: "{{ app_name }}-secret"
  namespace: "{{ app_namespace }}"
type: Opaque
stringData:
  KEYCLOAK_ADMIN_USERNAME: "{{ keycloak_admin_username }}"
  KEYCLOAK_ADMIN_PASSWORD: "{{ keycloak_admin_password }}"
  RABBITMQ_USERNAME: "{{ rabbitmq_username }}"
  RABBITMQ_PASSWORD: "{{ rabbitmq_password }}"
  # ... all Secret variables
```

## Role: k8s-deploy

```yaml
# roles/k8s-deploy/tasks/main.yml
---
- name: Apply Deployment
  kubernetes.core.k8s:
    state: present
    template: deployment.yml.j2

- name: Apply Service
  kubernetes.core.k8s:
    state: present
    template: service.yml.j2

- name: Apply Ingress
  kubernetes.core.k8s:
    state: present
    template: ingress.yml.j2
  when: app_ingress_enabled | default(false)

- name: Wait for rollout to complete
  kubernetes.core.k8s_info:
    kind: Deployment
    name: "{{ app_name }}"
    namespace: "{{ app_namespace }}"
    wait: true
    wait_condition:
      type: Available
      status: "True"
    wait_timeout: 300
```

## Role: k8s-deps

```yaml
# roles/k8s-deps/tasks/main.yml
---
- name: Deploy MongoDB
  kubernetes.core.k8s:
    state: present
    template: mongodb.yml.j2
  when: deploy_dependencies | default(false)

- name: Deploy Keycloak
  kubernetes.core.k8s:
    state: present
    template: keycloak.yml.j2
  when: deploy_dependencies | default(false)

- name: Deploy RabbitMQ
  kubernetes.core.k8s:
    state: present
    template: rabbitmq.yml.j2
  when: deploy_dependencies | default(false)

- name: Wait for dependencies to be ready
  kubernetes.core.k8s_info:
    kind: StatefulSet
    name: "{{ item }}"
    namespace: "{{ app_namespace }}"
    wait: true
    wait_condition:
      type: Ready
    wait_timeout: 300
  loop: "{{ dependency_statefulsets | default([]) }}"
  when: deploy_dependencies | default(false)
```

## Role: k8s-verify

```yaml
# roles/k8s-verify/tasks/main.yml
---
- name: Get pod status
  kubernetes.core.k8s_info:
    kind: Pod
    namespace: "{{ app_namespace }}"
    label_selectors:
      - "app={{ app_name }}"
  register: pod_info

- name: Verify all pods are running
  ansible.builtin.assert:
    that:
      - pod_info.resources | length > 0
      - pod_info.resources | map(attribute='status.phase') | list | unique == ['Running']
    fail_msg: "Not all pods are in Running state"
    success_msg: "All {{ pod_info.resources | length }} pods are Running"

- name: Check health endpoint from within cluster
  kubernetes.core.k8s_exec:
    namespace: "{{ app_namespace }}"
    pod: "{{ pod_info.resources[0].metadata.name }}"
    command: >
      wget --no-verbose --tries=3 --spider
      http://localhost:{{ server_port }}{{ health_endpoint | default('/') }}
  register: health_check
  retries: 5
  delay: 10
  until: health_check.rc == 0

- name: Display deployment summary
  ansible.builtin.debug:
    msg: |
      Deployment Summary:
        Application: {{ app_name }}
        Version:     {{ app_version }}
        Namespace:   {{ app_namespace }}
        Replicas:    {{ pod_info.resources | length }}
        Image:       {{ image_name }}:{{ image_tag }}
        Health:      OK
```

## Main Playbook — deploy.yml

```yaml
---
- name: Deploy {{ app_name }}
  hosts: k8s_control_plane
  gather_facts: false

  pre_tasks:
    - name: Validate required variables
      ansible.builtin.assert:
        that:
          - app_name is defined
          - app_version is defined
          - app_namespace is defined
          - image_registry is defined
        fail_msg: "Missing required deployment variables"

  roles:
    - role: docker-build
      tags: [build]
    - role: k8s-namespace
      tags: [deploy]
    - role: k8s-deps
      tags: [deploy, deps]
      when: deploy_dependencies | default(false)
    - role: k8s-config
      tags: [deploy, config]
    - role: k8s-deploy
      tags: [deploy]
    - role: k8s-verify
      tags: [verify]
```

## Rollback Playbook — rollback.yml

```yaml
---
- name: Rollback {{ app_name }}
  hosts: k8s_control_plane
  gather_facts: false

  tasks:
    - name: Rollback to previous revision
      ansible.builtin.command:
        cmd: >
          kubectl rollout undo deployment/{{ app_name }}
          -n {{ app_namespace }}
          {% if rollback_revision is defined %}
          --to-revision={{ rollback_revision }}
          {% endif %}
      register: rollback_result

    - name: Display rollback result
      ansible.builtin.debug:
        msg: "{{ rollback_result.stdout }}"

    - name: Wait for rollback to complete
      kubernetes.core.k8s_info:
        kind: Deployment
        name: "{{ app_name }}"
        namespace: "{{ app_namespace }}"
        wait: true
        wait_condition:
          type: Available
          status: "True"
        wait_timeout: 300

    - name: Verify health after rollback
      ansible.builtin.include_role:
        name: k8s-verify
```

## Usage

```bash
# Deploy to dev
ansible-playbook deploy.yml -i inventory/dev/hosts.yml

# Deploy to prod
ansible-playbook deploy.yml -i inventory/prod/hosts.yml --ask-vault-pass

# Deploy only (skip build)
ansible-playbook deploy.yml -i inventory/dev/hosts.yml --skip-tags build

# Rollback
ansible-playbook rollback.yml -i inventory/prod/hosts.yml

# Rollback to specific revision
ansible-playbook rollback.yml -i inventory/prod/hosts.yml -e rollback_revision=3
```
