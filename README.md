# ansible_roles

Automatización Day 2 con Ansible para operar un repositorio GitOps de OpenShift mediante playbooks de orquestación y roles reutilizables.

## Alcance

Este repositorio cubre cuatro flujos operativos principales:

- creación de namespaces GitOps;
- renderizado y mantenimiento de RBAC;
- generación de políticas ACM;
- escalado declarativo de workloads.

Los playbooks consumen roles comunes para validar entradas, preparar un workspace temporal, renderizar manifiestos en el checkout del repositorio GitOps, hacer commit/push y limpiar el workspace al final.

## Estructura del repositorio

```text
ansible_roles/
├── playbooks/
│   ├── create_namespace.yml
│   ├── create_acm_policy.yml
│   ├── manage_rbac.yml
│   └── scale_workload.yml
└── roles/
    ├── gitops_acm_policy_render/
    ├── gitops_cleanup/
    ├── gitops_common_validate/
    ├── gitops_git_push/
    ├── gitops_namespace/
    ├── gitops_namespace_render/
    ├── gitops_rbac_render/
    ├── gitops_workload_scale_render/
    └── gitops_workspace/
```

## Playbooks disponibles

### `playbooks/create_namespace.yml`

Orquesta la creación completa de un namespace en el repositorio GitOps.

Flujo:

1. valida `repo_url` y `namespace_name`;
2. prepara el workspace temporal y clona el repositorio destino;
3. renderiza el namespace mediante `gitops_namespace_render`;
4. hace `git add`, `commit` y `push`;
5. limpia el workspace temporal.

Ejemplo:

```bash
ansible-playbook playbooks/create_namespace.yml \
  -e repo_url=git@github.com:org/gitops-repo.git \
  -e namespace_name=mi-aplicacion \
  -e admin_group='CN=APP-ADMINS,OU=Groups,DC=example,DC=com' \
  -e reader_group='CN=APP-READERS,OU=Groups,DC=example,DC=com'
```

### `playbooks/create_acm_policy.yml`

Genera la estructura base de una política ACM dentro del árbol GitOps.

Flujo:

1. valida `repo_url`, `acm_policy_name`, `acm_policy_namespace` y `acm_configurationpolicy_object_definitions`;
2. prepara el workspace temporal;
3. renderiza la política mediante `gitops_acm_policy_render`;
4. confirma los cambios en Git;
5. elimina el workspace temporal.

Ejemplo:

```bash
ansible-playbook playbooks/create_acm_policy.yml \
  -e repo_url=git@github.com:org/gitops-repo.git \
  -e acm_policy_name=allow-registry \
  -e acm_policy_namespace=policies \
  -e '{"acm_configurationpolicy_object_definitions":[{"apiVersion":"v1","kind":"ConfigMap","metadata":{"name":"example-config","namespace":"default"},"data":{"enabled":"true"}}]}'
```

### `playbooks/manage_rbac.yml`

Renderiza RBAC reutilizable para un namespace existente dentro del repositorio GitOps.

Flujo:

1. valida `repo_url` y `namespace_name`;
2. prepara el workspace temporal;
3. normaliza cuentas de servicio y bindings legacy si no se proporcionan estructuras explícitas;
4. renderiza RBAC mediante `gitops_rbac_render`;
5. confirma los cambios en Git;
6. limpia el workspace temporal.

Ejemplo:

```bash
ansible-playbook playbooks/manage_rbac.yml \
  -e repo_url=git@github.com:org/gitops-repo.git \
  -e namespace_name=mi-aplicacion \
  -e '{"gitops_rbac_group_bindings":[{"name":"mi-aplicacion-admin-group","namespace":"mi-aplicacion","role_ref_kind":"ClusterRole","role_ref_name":"admin","subjects":[{"name":"CN=APP-ADMINS,OU=Groups,DC=example,DC=com"}]},{"name":"mi-aplicacion-reader-group","namespace":"mi-aplicacion","role_ref_kind":"ClusterRole","role_ref_name":"view","subjects":[{"name":"CN=APP-READERS,OU=Groups,DC=example,DC=com"}]}]}'
```

### `playbooks/scale_workload.yml`

Genera overlays y parches de Kustomize para escalar workloads declarativamente.

Flujo:

1. valida `repo_url` y `gitops_scaled_workloads`;
2. prepara el workspace temporal;
3. renderiza parches mediante `gitops_workload_scale_render`;
4. confirma los cambios en Git;
5. limpia el workspace temporal.

Ejemplo:

```bash
ansible-playbook playbooks/scale_workload.yml \
  -e repo_url=git@github.com:org/gitops-repo.git \
  -e '{"gitops_scaled_workloads":[{"kind":"Deployment","name":"api","namespace":"mi-aplicacion","replicas":3,"patch_strategy":"strategic-merge"},{"kind":"StatefulSet","name":"postgresql","namespace":"mi-aplicacion","replicas":2,"patch_strategy":"patches"}]}'
```

## Roles comunes

### `gitops_common_validate`

Valida que un listado de variables obligatorias exista y no esté vacío. Se usa como primera barrera en los playbooks de orquestación.

### `gitops_workspace`

Construye un workspace temporal, fija identificadores de ejecución, expone aliases legacy (`workspace_dir` y `repo_checkout_dir`), clona el repositorio GitOps y crea directorios raíz opcionales.

### `gitops_git_push`

Comprueba si hay cambios con `git status --porcelain`; si existen, ejecuta `git add`, `git commit` y `git push`. Si no hay cambios, informa explícitamente que no hay nada que subir.

### `gitops_cleanup`

Elimina el directorio temporal de trabajo al final de cada orquestación.

## Roles de dominio

### `gitops_namespace_render`

Rol canónico para renderizar un namespace GitOps. Valida el nombre del namespace, normaliza group bindings y service accounts, renderiza los manifiestos base del namespace y actualiza el `kustomization.yaml` padre.

Recursos que renderiza:

- `namespace.yaml`
- `quota.yaml`
- `limitrange.yaml`
- `rbac/groups.yaml` cuando hay group bindings;
- `rbac/serviceaccounts-bindings.yaml` cuando hay bindings de service accounts;
- `serviceaccounts/*.yaml` para las cuentas marcadas para creación;
- `kustomization.yaml` del namespace.

> Compatibilidad: `gitops_namespace` se mantiene explícitamente como shim backward-compatible y delega en `gitops_namespace_render`.

> NetworkPolicies: las plantillas se conservan como referencia, pero están fuera del flujo operativo del namespace y orientadas al dominio ACM; por eso no se renderizan ni se añaden al `kustomization.yaml` del namespace.

### `gitops_rbac_render`

Renderiza artefactos RBAC reutilizables: group bindings, service accounts y bindings de service accounts, en los directorios de destino que reciba por variables.

### `gitops_acm_policy_render`

Renderiza la estructura base de una política ACM, incluyendo `policy.yml`, `placement.yml`, `placementbinding.yml`, `configuration-policy.yml` y el `kustomization.yml` del propio recurso, además de registrar la política en el `kustomization.yaml` padre.

### `gitops_workload_scale_render`

Normaliza la definición de workloads a escalar, valida combinaciones soportadas (`Deployment`, `StatefulSet`, `DaemonSet` y estrategias `strategic-merge` o `patches`), genera los parches y actualiza el `kustomization.yml` del overlay de escalado.

## Variables y contratos principales

### Namespace y RBAC

Variables públicas y/o legacy soportadas por el flujo de namespace:

- `repo_url`
- `git_branch`
- `repo_path`
- `workspace_base`
- `namespace_name`
- `admin_group`
- `reader_group`
- `ns_service_accounts`
- `gitops_rbac_group_bindings`
- `gitops_rbac_serviceaccounts`
- `gitops_rbac_serviceaccount_bindings`
- `quota`
- `limitrange`
- `namespace_labels`
- `namespace_annotations`
- `default_service_accounts`
- `git_commit_message`

### ACM

Variables principales del flujo ACM:

- `repo_url`
- `git_branch`
- `repo_path`
- `workspace_base`
- `acm_policy_name`
- `acm_policy_namespace`
- `acm_configurationpolicy_object_definitions`
- `acm_policy_remediation_action`
- `acm_policy_placement_name`
- `acm_policy_placementbinding_name`
- `acm_configurationpolicy_name`
- `acm_configurationpolicy_severity`
- `git_commit_message`

### Escalado

Variables principales del flujo de escalado:

- `repo_url`
- `git_branch`
- `repo_path`
- `workspace_base`
- `gitops_scaled_workloads`
- `gitops_scale_base_resources`
- `git_commit_message`

## Convenciones operativas

- Los playbooks están nombrados con el naming real del repositorio: `create_namespace.yml`, `create_acm_policy.yml`, `scale_workload.yml` y `manage_rbac.yml`.
- El checkout GitOps siempre se realiza en un workspace temporal.
- El flujo hace limpieza en bloque `always` para evitar residuos locales.
- El render de namespace y RBAC mantiene compatibilidad con variables legacy mientras prioriza estructuras explícitas más nuevas.
