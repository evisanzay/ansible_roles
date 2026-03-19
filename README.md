# ioeu-osp-day2-automation

`Copyright (c) MAPFRE DevOps Platform`

## Descripción

Este repositorio contiene playbooks y roles de Ansible para automatización Day 2 sobre OpenShift. El flujo principal crea namespaces en un repositorio GitOps y renderiza los manifiestos base necesarios para su kustomization.

## Estructura

```text
ioeu-osp-day2-automation/
|-- ansible.cfg
|-- inventory/
|   `-- localhost.yaml
|-- playbooks/
|   |-- create_namespace.yml
|   `-- manage_rbac.yml
`-- roles/
    |-- gitops_namespace_render/
    |   |-- defaults/main.yml
    |   |-- tasks/main.yml
    |   |-- templates/
    |   `-- vars/main.yml
    |-- gitops_namespace/
    |   `-- tasks/main.yml
    |-- gitops_rbac_render/
    |-- gitops_workspace/
    |-- gitops_git_push/
    `-- gitops_cleanup/
```

`gitops_namespace_render` es el rol canónico. `gitops_namespace` se mantiene como shim backward-compatible y delega en el nuevo rol.

## Recursos renderizados

- `namespace.yaml`
- `quota.yaml`
- `limitrange.yaml`
- `rbac/groups.yaml`
- `rbac/serviceaccounts-bindings.yaml`
- `serviceaccounts/*.yaml`
- `kustomization.yaml`

Las plantillas de `NetworkPolicies` se conservan como referencia, pero ya no se renderizan ni se añaden al `kustomization` del namespace. Ese dominio queda reservado para ACM para evitar manifiestos huérfanos.

## Contrato de variables

Variables públicas validadas y mantenidas por compatibilidad:

- `repo_url`
- `git_branch`
- `repo_path`
- `namespace_name`
- `admin_group`
- `reader_group`
- `ns_service_accounts`

También siguen disponibles `gitops_rbac_group_bindings`, `gitops_rbac_serviceaccounts` y `gitops_rbac_serviceaccount_bindings` para personalizaciones explícitas de RBAC.

## Uso

```bash
ansible-playbook playbooks/create_namespace.yml \
  -e repo_url=git@github.com:mapfre/gitops-repo.git \
  -e namespace_name=mi-aplicacion \
  -e admin_group=CN=APP-ADMINS,OU=Groups,DC=mapfre,DC=com \
  -e reader_group=CN=APP-READERS,OU=Groups,DC=mapfre,DC=com
```

## Flujo

1. Validación de parámetros de entrada.
2. Preparación de workspace y checkout del repositorio GitOps.
3. Normalización de RBAC y service accounts, y cálculo de `kustomize_resources`.
4. Render de manifiestos base del namespace.
5. Commit y push de cambios.
6. Limpieza del workspace temporal.
