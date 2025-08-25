ajouter declaratif vs imperatif, faire installer k9s au lieu du dashboard ? un équivalent pour docker ?

ajouter colpletion auto


# Préambule

(des VMs aux conteneurs)

# Docker les bases
TODO

## Installation et prise en main
TODO

## Premier run
TODO

## Réseau et port-forward
TODO

## Volumes
TODO

# Construire son conteneur

## Premier build et tag
TODO

## Dockerfile en détail
TODO

## Pousser son conteneur sur un registre
TODO

### En ligne
TODO

### En local
TODO

## Multi-stage builds
TODO

# Comprendre Kubernetes
TODO

## Installation de k3s
TODO

## Composants
TODO

### Boucles de réconcialiation
TODO

### Metadata, labels et specs
TODO

### Il faudrait un sous-chapitre ici pour équilibrer
TODO

## Premiers pas
TODO

### Run pod
TODO

### port-forward
TODO

## Service
TODO (dont headless service)

## Ingress
TODO

## Deploiement, StatefullSets Jobs et CronJobs
TODO

### Déploiements
TODO

### StatefulSets
TODO

### Jobs & CronJobs
TODO

## PV, PVC et Volumes
TODO

# Objets avancés 
TODO

## RBAC
TODO

## Netpol
TODO

## Autre TBD
TODO

## Autre TBD
TODO

# Build, CI et CD
TODO

## Buildha
TODO

## Helm (obligatoire)
TODO

## Argo (pas mieux)
TODO (introduit au passaghe les CRDs et les opérateurs ?) 

## Flux ? 
TODO ?

# A placer obligatoirement: 

Kubernetes - une histoire d'API  (OCI, CRI, CSI, CNI): absolument parler des différents plugins réseaux et des limites de flannel

# Chapitres non retenus 

Docker compose (difficile et pas forcément utile dans une optique docker + Kubernetes?) 



---
- name: Add topologySpreadConstraints to workloads
  hosts: localhost
  gather_facts: false
  collections:
    - community.kubernetes
  vars:
    topology_key: "topology.kubernetes.io/zone"
    when_unsatisfiable: "DoNotSchedule"
    max_skew: 1
    resources:
      - { kind: Deployment,  name: api,   namespace: prod }
      - { kind: StatefulSet, name: redis, namespace: prod }

  tasks:
    - name: Ensure topologySpreadConstraints present
      block:
        - name: Get workload
          k8s_info:
            api_version: apps/v1
            kind: "{{ item.kind }}"
            name: "{{ item.name }}"
            namespace: "{{ item.namespace }}"
          register: workload

        - name: Extract labels and constraints info
          set_fact:
            app_label_key: >-
              {{ 'app' if 'app' in (workload.resources[0].spec.template.metadata.labels | default({}))
                 else ('app.kubernetes.io/name' if 'app.kubernetes.io/name' in (workload.resources[0].spec.template.metadata.labels | default({}))
                       else '') }}
            app_label_value: >-
              {{ (workload.resources[0].spec.template.metadata.labels | default({}))[app_label_key]
                 if app_label_key|length > 0 else '' }}
            has_constraints: >-
              {{ (workload.resources[0].spec.template.spec.topologySpreadConstraints | default([])) | length > 0 }}

        - name: Add topologySpreadConstraints array if missing
          when:
            - app_label_key | length > 0
            - not has_constraints
          k8s_json_patch:
            api_version: apps/v1
            kind: "{{ item.kind }}"
            name: "{{ item.name }}"
            namespace: "{{ item.namespace }}"
            patch:
              - op: add
                path: /spec/template/spec/topologySpreadConstraints
                value:
                  - maxSkew: "{{ max_skew }}"
                    topologyKey: "{{ topology_key }}"
                    whenUnsatisfiable: "{{ when_unsatisfiable }}"
                    labelSelector:
                      matchLabels:
                        "{{ app_label_key }}": "{{ app_label_value }}"

        - name: Append new constraint if array already exists
          when:
            - app_label_key | length > 0
            - has_constraints
          k8s_json_patch:
            api_version: apps/v1
            kind: "{{ item.kind }}"
            name: "{{ item.name }}"
            namespace: "{{ item.namespace }}"
            patch:
              - op: add
                path: /spec/template/spec/topologySpreadConstraints/-
                value:
                  maxSkew: "{{ max_skew }}"
                  topologyKey: "{{ topology_key }}"
                  whenUnsatisfiable: "{{ when_unsatisfiable }}"
                  labelSelector:
                    matchLabels:
                      "{{ app_label_key }}": "{{ app_label_value }}"

        - name: Warn if no app label found
          when: app_label_key | length == 0
          debug:
            msg: >-
              Skipped {{ item.kind }}/{{ item.namespace }}/{{ item.name }}:
              no "app" or "app.kubernetes.io/name" label found on pod template.
      loop: "{{ resources }}"
      loop_control:
        label: "{{ item.kind }}/{{ item.namespace }}/{{ item.name }}"









---
- name: Add topologySpreadConstraints using kubectl + jq (split tasks)
  hosts: localhost
  gather_facts: false
  vars:
    topology_key: "topology.kubernetes.io/zone"  # e.g. kubernetes.io/hostname
    when_unsatisfiable: "DoNotSchedule"          # or ScheduleAnyway
    max_skew: 1
    resources:
      - { kind: Deployment,  name: api,   namespace: prod }
      - { kind: StatefulSet, name: redis, namespace: prod }

  tasks:
    - name: Process workloads one by one
      block:
        - name: Get object JSON
          shell: |
            set -euo pipefail
            kubectl -n "{{ item.namespace }}" get "{{ item.kind }}" "{{ item.name }}" -o json
          args:
            executable: /bin/bash
          register: obj_json

        - name: Determine app label key (app or app.kubernetes.io/name)
          shell: |
            set -euo pipefail
            jq -r '
              (.spec.template.metadata.labels // {}) as $l |
              if $l["app"] then "app"
              elif $l["app.kubernetes.io/name"] then "app.kubernetes.io/name"
              else "" end
            ' <<< "{{ obj_json.stdout }}"
          args:
            executable: /bin/bash
          register: app_key

        - name: Fail or skip if no app label key found
          when: app_key.stdout | length == 0
          debug:
            msg: >-
              SKIPPED: {{ item.kind }}/{{ item.namespace }}/{{ item.name }}
              -> no "app" or "app.kubernetes.io/name" label on pod template

        - name: Read app label value
          when: app_key.stdout | length > 0
          shell: |
            set -euo pipefail
            jq -r --arg k "{{ app_key.stdout }}" '
              (.spec.template.metadata.labels // {})[$k]
            ' <<< "{{ obj_json.stdout }}"
          args:
            executable: /bin/bash
          register: app_val

        - name: Skip if label value missing
          when:
            - app_key.stdout | length > 0
            - (app_val.stdout | length == 0 or app_val.stdout == "null")
          debug:
            msg: >-
              SKIPPED: {{ item.kind }}/{{ item.namespace }}/{{ item.name }}
              -> label {{ app_key.stdout }} has no value

        - name: Check if topologySpreadConstraints exists
          when:
            - app_key.stdout | length > 0
            - app_val.stdout | length > 0
            - app_val.stdout != "null"
          shell: |
            set -euo pipefail
            jq -r '((.spec.template.spec.topologySpreadConstraints // []) | length) > 0'
            <<< "{{ obj_json.stdout }}"
          args:
            executable: /bin/bash
          register: has_constraints

        - name: Check if matching constraint already exists (idempotency)
          when:
            - app_key.stdout | length > 0
            - app_val.stdout | length > 0
            - app_val.stdout != "null"
          shell: |
            set -euo pipefail
            jq -r \
              --arg k "{{ topology_key }}" \
              --arg lk "{{ app_key.stdout }}" \
              --arg lv "{{ app_val.stdout }}" '
              any((.spec.template.spec.topologySpreadConstraints // [])[];
                  (.topologyKey == $k)
                  and ((.labelSelector.matchLabels[$lk] // "") == $lv)
              )
            ' <<< "{{ obj_json.stdout }}"
          args:
            executable: /bin/bash
          register: exists

        - name: Skip if matching constraint already present
          when:
            - exists.stdout is defined
            - exists.stdout == "true"
          debug:
            msg: >-
              SKIPPED: {{ item.kind }}/{{ item.namespace }}/{{ item.name }}
              -> matching constraint already present for {{ app_key.stdout }}={{ app_val.stdout }}
              on {{ topology_key }}

        - name: Build patch payload (append to existing array)
          when:
            - app_key.stdout | length > 0
            - app_val.stdout | length > 0
            - app_val.stdout != "null"
            - has_constraints.stdout == "true"
            - exists.stdout == "false"
          shell: |
            set -euo pipefail
            jq -n \
              --arg k "{{ topology_key }}" \
              --arg w "{{ when_unsatisfiable }}" \
              --arg lk "{{ app_key.stdout }}" \
              --arg lv "{{ app_val.stdout }}" \
              --argjson m {{ max_skew | int }} '
              [
                {
                  op: "add",
                  path: "/spec/template/spec/topologySpreadConstraints/-",
                  value: {
                    maxSkew: $m,
                    topologyKey: $k,
                    whenUnsatisfiable: $w,
                    labelSelector: { matchLabels: { ($lk): $lv } }
                  }
                }
              ]'
          args:
            executable: /bin/bash
          register: patch_append

        - name: Build patch payload (create array if missing)
          when:
            - app_key.stdout | length > 0
            - app_val.stdout | length > 0
            - app_val.stdout != "null"
            - has_constraints.stdout == "false"
            - exists.stdout == "false"
          shell: |
            set -euo pipefail
            jq -n \
              --arg k "{{ topology_key }}" \
              --arg w "{{ when_unsatisfiable }}" \
              --arg lk "{{ app_key.stdout }}" \
              --arg lv "{{ app_val.stdout }}" \
              --argjson m {{ max_skew | int }} '
              [
                {
                  op: "add",
                  path: "/spec/template/spec/topologySpreadConstraints",
                  value: [
                    {
                      maxSkew: $m,
                      topologyKey: $k,
                      whenUnsatisfiable: $w,
                      labelSelector: { matchLabels: { ($lk): $lv } }
                    }
                  ]
                }
              ]'
          args:
            executable: /bin/bash
          register: patch_create

        - name: Apply patch (append)
          when:
            - patch_append.stdout is defined
            - patch_append.stdout | length > 0
          shell: |
            set -euo pipefail
            kubectl -n "{{ item.namespace }}" patch "{{ item.kind }}" "{{ item.name }}" \
              --type='json' -p '{{ patch_append.stdout | replace("'", "'\"'\"'") }}'
          args:
            executable: /bin/bash
          register: patch_apply_append
          changed_when: patch_apply_append.rc == 0

        - name: Apply patch (create)
          when:
            - patch_create.stdout is defined
            - patch_create.stdout | length > 0
          shell: |
            set -euo pipefail
            kubectl -n "{{ item.namespace }}" patch "{{ item.kind }}" "{{ item.name }}" \
              --type='json' -p '{{ patch_create.stdout | replace("'", "'\"'\"'") }}'
          args:
            executable: /bin/bash
          register: patch_apply_create
          changed_when: patch_apply_create.rc == 0

        - name: Report result
          debug:
            msg: >-
              {{ (patch_apply_append is defined and patch_apply_append.rc == 0) or
                 (patch_apply_create is defined and patch_apply_create.rc == 0)
                 | ternary(
                   "PATCHED: " ~ item.kind ~ "/" ~ item.namespace ~ "/" ~ item.name
                   ~ " -> constraint for " ~ app_key.stdout ~ "=" ~ app_val.stdout
                   ~ " on " ~ topology_key,
                   "NO CHANGE: " ~ item.kind ~ "/" ~ item.namespace ~ "/" ~ item.name
                 )
              }}
      loop: "{{ resources }}"
      loop_control:
        label: "{{ item.kind }}/{{ item.namespace }}/{{ item.name }}"

