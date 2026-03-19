# **ioeu-osp-day2-automation**

```Copyright (c) MAPFRE DevOps Platform```

## 🖥️ Descripción

Este repositorio contiene playbooks y roles de Ansible para automatización de operaciones Day 2 en clústeres OpenShift. Actualmente implementa la creación automatizada de namespaces con todos sus recursos asociados siguiendo un enfoque GitOps.

## 📁 Estructura de directorios

```
ioeu-osp-day2-automation/
├── ansible.cfg                          # Configuración de Ansible
├── inventory/
│   └── localhost.yaml                   # Inventario para ejecución local
├── playbooks/
│   └── create_namespace.yaml            # Playbook principal de creación de namespace
└── roles/
    └── gitops_namespace/                # Rol para alta de namespaces GitOps
        ├── defaults/
        │   └── main.yaml                # Valores por defecto (quotas, limits, etc.)
        ├── tasks/
        │   ├── main.yml                 # Orquestación principal del rol
        │   ├── validate.yml             # Validaciones de parámetros
        │   ├── workspace.yml            # Preparación y clonado del repo
        │   ├── build_resources.yml      # Construcción dinámica de recursos
        │   ├── render.yml               # Renderizado de templates
        │   ├── git_push.yml             # Commit y push al repositorio
        │   └── cleanup.yml              # Limpieza del workspace temporal
        ├── templates/                   # Templates Jinja2 para manifiestos K8s
        │   ├── kustomization.yaml.j2
        │   ├── namespace.yaml.j2
        │   ├── quota.yaml.j2
        │   ├── limitrange.yaml.j2
        │   ├── rbac-groups.yaml.j2
        │   ├── rbac-serviceaccounts-bindings.yaml.j2
        │   ├── serviceaccount.yaml.j2
        │   ├── networkpolicy-default-deny.yaml.j2
        │   ├── networkpolicy-allow-dns.yaml.j2
        │   └── networkpolicy-allow-ingress-controller.yaml.j2
        └── vars/
            └── main.yaml                # Variables derivadas (paths, etc.)
```

## 📚 Playbooks

### `create_namespace.yaml`

Crea un namespace completo en OpenShift siguiendo el patrón GitOps con Kustomize. El playbook clona un repositorio Git destino, genera los manifiestos del namespace y realiza commit/push para que Argo CD sincronice los cambios.

**Recursos generados:**

- `namespace.yaml` - Definición del namespace con labels y annotations
- `quota.yaml` - ResourceQuota para limitar recursos del namespace
- `limitrange.yaml` - LimitRange con valores por defecto para pods
- `rbac/groups.yaml` - RoleBindings para grupos de administradores y lectores
- `rbac/serviceaccounts-bindings.yaml` - Bindings para service accounts
- `serviceaccounts/*.yaml` - Service accounts configurados
- `networkpolicies/*.yaml` - NetworkPolicies (deny-all, allow-dns, allow-ingress)
- `kustomization.yaml` - Archivo Kustomize que orquesta todos los recursos

## ⚙️ Variables

### Parámetros obligatorios

| Variable | Descripción |
|----------|-------------|
| `repo_url` | URL del repositorio Git destino |
| `namespace_name` | Nombre del namespace (debe cumplir formato K8s) |
| `admin_group` | Grupo LDAP con permisos de administrador |
| `reader_group` | Grupo LDAP con permisos de lectura |

### Parámetros opcionales (defaults)

| Variable | Default | Descripción |
|----------|---------|-------------|
| `git_branch` | `main` | Rama del repositorio |
| `repo_path` | `clusters/prod/namespaces` | Ruta dentro del repo |
| `quota.requests_cpu` | `2` | CPU requests máximo |
| `quota.requests_memory` | `4Gi` | Memoria requests máximo |
| `quota.limits_cpu` | `4` | CPU limits máximo |
| `quota.limits_memory` | `8Gi` | Memoria limits máximo |
| `quota.pods` | `20` | Número máximo de pods |
| `limitrange.default_cpu` | `500m` | CPU limit por defecto |
| `limitrange.default_memory` | `512Mi` | Memoria limit por defecto |
| `limitrange.default_request_cpu` | `250m` | CPU request por defecto |
| `limitrange.default_request_memory` | `256Mi` | Memoria request por defecto |
| `namespace_labels` | `mapfre.com/type: autoservice` | Labels del namespace |
| `network_policies.default_deny` | `true` | Habilitar deny-all policy |
| `network_policies.allow_dns` | `true` | Permitir salida DNS |
| `network_policies.allow_ingress_controller` | `true` | Permitir tráfico del ingress |

## 🚀 Uso

### Ejecución básica

```bash
ansible-playbook playbooks/create_namespace.yaml \
  -e repo_url=git@github.com:mapfre/gitops-repo.git \
  -e namespace_name=mi-aplicacion \
  -e admin_group=CN=APP-ADMINS,OU=Groups,DC=mapfre,DC=com \
  -e reader_group=CN=APP-READERS,OU=Groups,DC=mapfre,DC=com
```

### Con parámetros personalizados

```bash
ansible-playbook playbooks/create_namespace.yaml \
  -e repo_url=git@github.com:mapfre/gitops-repo.git \
  -e namespace_name=mi-aplicacion \
  -e admin_group=CN=APP-ADMINS,OU=Groups,DC=mapfre,DC=com \
  -e reader_group=CN=APP-READERS,OU=Groups,DC=mapfre,DC=com \
  -e git_branch=develop \
  -e '{"quota": {"requests_cpu": "4", "limits_memory": "16Gi"}}'
```

## 📋 Prerrequisitos

- Ansible 2.9+
- Git instalado y configurado con acceso al repositorio destino
- Python 3.x

## 🔄 Flujo de ejecución

1. **Validación** - Verifica parámetros obligatorios y formato del namespace
2. **Workspace** - Crea directorio temporal y clona el repositorio destino
3. **Build** - Construye la lista dinámica de recursos a generar
4. **Render** - Renderiza los templates Jinja2 con los valores proporcionados
5. **Git Push** - Añade, hace commit y push de los cambios al repositorio
6. **Cleanup** - Elimina el workspace temporal

## 🤜🤛 Contribución

Para contribuir a este repositorio debes leer la *[Guía de Contribución](CONTRIBUTING.md)*
