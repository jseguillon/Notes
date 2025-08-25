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


