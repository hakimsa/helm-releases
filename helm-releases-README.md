# ⎈ helm-releases

Repositorio de **Helm Charts** de la organización. Contiene los charts reutilizables desplegados por el [workflow de CD](https://github.com/hakimsa/shared-workflows).

---

## 📋 Índice

- [Charts disponibles](#-charts-disponibles)
- [backend-mgt-app](#-backend-mgt-app)
  - [Requisitos](#requisitos)
  - [Estructura del chart](#estructura-del-chart)
  - [Templates](#templates)
  - [Values](#values)
  - [Secrets](#secrets)
  - [Instalación manual](#instalación-manual)
  - [Despliegue via CI/CD](#despliegue-via-cicd)
- [Convenciones](#-convenciones)

---

## 📦 Charts disponibles

| Chart | Descripción | Versión | App Version |
|---|---|---|---|
| `backend-mgt-app` | Backend Management Users Application | `0.1.0` | `1.0.0` |

---

## 🚀 backend-mgt-app

Chart para desplegar la aplicación Java/Spring Boot de gestión de usuarios en Kubernetes.

### Requisitos

- Kubernetes `>= 1.21`
- Helm `>= 3.0`
- Una instancia de MySQL accesible en el clúster en `mysql:3306`

### Estructura del chart

```
helm-releases/
├── Chart.yaml                  # Metadatos del chart
├── values.yaml                 # Valores por defecto
└── templates/
    ├── deployment.yaml         # Deployment de la aplicación
    ├── service.yaml            # Service ClusterIP
    ├── configmap.yaml          # application.properties montado como volumen
    └── secret.yaml             # Credenciales de base de datos y seguridad
```

### Templates

#### `deployment.yaml`

Crea un `Deployment` con:

- **Imagen:** `{{ .Values.image.repository }}:{{ .Values.image.tag }}`
- **Puerto del contenedor:** `8086`
- **Variables de entorno** inyectadas desde el Secret:

  | Variable | Clave del Secret |
  |---|---|
  | `DB_URL` | `db-url` |
  | `DB_USER` | `db-user` |
  | `DB_PASSWORD` | `db-password` |
  | `SECURITY_PASSWORD` | `security-password` |

- **Volumen montado:** el ConfigMap se monta en `/app/config/application.properties`

#### `service.yaml`

Crea un `Service` de tipo `ClusterIP` que expone el puerto configurado en `service.port` y lo redirige al puerto `8086` del contenedor.

#### `configmap.yaml`

Crea un `ConfigMap` llamado `<release-name>-config` con el fichero `application.properties`. El valor por defecto solo fija `server.port=8086`. Se puede extender con `config.applicationProperties` en los values.

#### `secret.yaml`

Crea un `Secret` de tipo `Opaque` llamado `<release-name>-secret`. Los valores se codifican en base64 automáticamente con el filtro `b64enc` de Helm a partir de los values `secret.*`.

> ⚠️ **Los values `secret.*` nunca deben commitearse con valores reales.** Deben pasarse en tiempo de despliegue via `--set` o desde GitHub Secrets a través del workflow de CD.

---

### Values

#### Imagen

| Value | Default | Descripción |
|---|---|---|
| `image.repository` | `hakimsamouh/backend-mgt-app` | Repositorio de la imagen Docker |
| `image.tag` | `dev` | Tag de la imagen. En CI/CD se sobreescribe con el SHA del commit |
| `image.pullPolicy` | `Always` | Política de pull de imagen |

#### Réplicas y servicio

| Value | Default | Descripción |
|---|---|---|
| `replicaCount` | `1` | Número de réplicas del Deployment |
| `service.type` | `ClusterIP` | Tipo de Service de Kubernetes |
| `service.port` | `8080` | Puerto expuesto por el Service |

#### Configuración de la aplicación

| Value | Default | Descripción |
|---|---|---|
| `config.applicationProperties` | `server.port=8080` + JPA config | Contenido completo del `application.properties` montado como volumen |

> 💡 El `application.properties` del ConfigMap por defecto solo fija `server.port=8086`. Para extender la configuración, sobreescribe `config.applicationProperties` con el contenido completo.

#### Secrets (credenciales)

| Value | Default | Descripción |
|---|---|---|
| `secret.dbUrl` | `jdbc:mysql://mysql:3306/dbs` | URL de conexión JDBC a MySQL |
| `secret.dbUser` | `root` | Usuario de la base de datos |
| `secret.dbPassword` | `""` | Contraseña de la base de datos — **debe pasarse en runtime** |
| `secret.securityPassword` | *(no definido)* | Contraseña de seguridad de la aplicación — **debe pasarse en runtime** |

---

### Instalación manual

```bash
# Instalar en el namespace 'default'
helm upgrade --install backend-mgt-app ./helm-releases \
  --namespace default \
  --create-namespace \
  --set image.tag=<tag> \
  --set secret.dbPassword=<password> \
  --set secret.securityPassword=<security-password>
```

Para usar un entorno específico con su propio fichero de values:

```bash
helm upgrade --install backend-mgt-app ./helm-releases \
  --namespace staging \
  --create-namespace \
  --values ./staging/values.yaml \
  --set image.tag=${{ github.sha }}
```

Para hacer rollback a la versión anterior:

```bash
helm rollback backend-mgt-app --namespace staging
```

Para listar el historial de releases:

```bash
helm history backend-mgt-app --namespace staging
```

---

### Despliegue via CI/CD

Este chart está pensado para ser desplegado automáticamente por el [reusable-cd.yml](https://github.com/hakimsa/shared-workflows/blob/main/.github/workflows/reusable-cd.yml). Ejemplo de workflow caller:

```yaml
jobs:
  cd:
    uses: hakimsa/shared-workflows/.github/workflows/reusable-cd.yml@main
    with:
      environment: 'staging'
      helm-release-name: 'backend-mgt-app'
      helm-namespace: 'staging'
      helm-charts-repo: 'hakimsa/helm-releases'
      helm-charts-ref: 'main'
      image-name: 'hakimsamouh/backend-mgt-app'
      image-tag: ${{ github.sha }}
    secrets: inherit
```

El workflow de CD ejecuta internamente:

```bash
helm upgrade --install backend-mgt-app ./helm-releases \
  --namespace staging \
  --create-namespace \
  --values ./helm-releases/staging/values.yaml \
  --set image.repository=hakimsamouh/backend-mgt-app \
  --set image.tag=<github.sha> \
  --wait \
  --timeout 5m
```

---

## 📐 Convenciones

**Nomenclatura de recursos**
Todos los recursos Kubernetes usan `{{ .Release.Name }}` como nombre base para evitar colisiones entre releases del mismo chart en distintos namespaces.

| Recurso | Nombre |
|---|---|
| Deployment | `<release-name>` |
| Service | `<release-name>` |
| ConfigMap | `<release-name>-config` |
| Secret | `<release-name>-secret` |

**Gestión de secretos**
Los valores sensibles (`dbPassword`, `securityPassword`) nunca deben definirse en `values.yaml`. Flujo recomendado:
1. Guárdalos como **GitHub Secrets** en el repositorio de la aplicación
2. El workflow de CD los inyecta via `--set` o `secrets: inherit`
3. Helm los codifica en base64 al crear el `Secret` de Kubernetes

**Ficheros de values por entorno**
Para gestionar configuraciones distintas por entorno, crea un fichero `values.yaml` en una carpeta con el nombre del entorno:

```
helm-releases/
├── staging/
│   └── values.yaml
└── production/
    └── values.yaml
```

El workflow de CD usará automáticamente `{environment}/values.yaml` si `helm-values-file` no está definido.
