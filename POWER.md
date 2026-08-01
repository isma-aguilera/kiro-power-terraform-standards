---
name: "terraform-standards"
displayName: "Estándares Terraform de <TuOrg>"
description: "Gobierna la IaC con Terraform en AWS mediante módulos privados versionados: política module-only, tagging obligatorio, assume_role sobre terraform-iac y backend S3 con locking nativo"
author: "<TuOrg>"
keywords:
    [
        "terraform",
        "iac",
        "aws",
        "vpc",
        "ec2",
        "infraestructura",
        "governance",
        "modulos",
        "backend",
        "s3",
        "tfstate",
        "tagging",
        "assume_role",
    ]
---

# Terraform Standards Power

Un Kiro Power que aplica los estándares de IaC con Terraform de <TuOrg>. Garantiza que toda la infraestructura se aprovisione exclusivamente mediante módulos privados y versionados de la organización `<TuOrg>` en GitHub, siguiendo convenciones estrictas de nombres, tagging, estructura de archivos y configuración del backend.

## Qué aporta este Power

- **MCP Server** — Tools para descubrir, inspeccionar y scaffoldear los módulos privados de Terraform de <TuOrg>
- **Agent Steering** — Convenciones incluidas automáticamente en cada interacción con el agente (política module-only, tagging, nombres, subneteo, backend)
- **Agent Hooks** — Verificaciones automáticas al guardar y antes de escribir, creadas en el workspace durante el onboarding

---

## Onboarding

Ejecuta estos pasos la primera vez que se usa el Power en un workspace.

### 1. Verificar prerrequisitos

Comprueba que estén disponibles y avisa al usuario de lo que falte:

- `terraform version` → debe ser >= 1.10 (necesario para `use_lockfile`)
- `uvx --version` → necesario para arrancar el MCP server
- La variable `GITHUB_TOKEN` definida, con scope `repo` (los módulos son privados)
- `tflint` es opcional: si no está, el hook de guardado lo omite sin fallar

### 2. Crear los agent hooks del workspace

Los hooks no se instalan con el Power: viven en el workspace. Crea estos tres archivos
si no existen (no sobrescribas los que ya estén, avisa al usuario en su lugar).

**`.kiro/hooks/terraform-validate-save.kiro.hook`**

```json
{
  "enabled": true,
  "name": "Validar Terraform al guardar",
  "description": "Ejecuta fmt, validate y tflint cada vez que se guarda un archivo .tf",
  "version": "1.2.0",
  "when": {
    "type": "fileEdited",
    "patterns": ["*.tf"]
  },
  "then": {
    "type": "runCommand",
    "command": "terraform fmt -check -diff . 2>&1; terraform validate -no-color 2>&1; if command -v tflint >/dev/null 2>&1; then tflint 2>&1; fi; true"
  }
}
```

**`.kiro/hooks/terraform-tag-check.kiro.hook`**

```json
{
  "enabled": true,
  "name": "Verificar convenciones de Terraform",
  "description": "Verifica tags, versiones pineadas, assume_role y backend.tf antes de escribir un .tf",
  "version": "1.4.0",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["write"],
    "patterns": ["*.tf"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "Antes de escribir este archivo .tf, verifica que incluya los tags obligatorios (Environment, Team, CostCenter, ManagedBy), ya sea directamente o vía default_tags en el bloque provider. Verifica también que la restricción de versión de Terraform sea >= 1.10 y que la versión del provider de AWS esté pineada (por ejemplo, ~> 5.0 en root modules, o un rango acotado como >= 5.0, < 7.0 en módulos reutilizables — nunca un >= abierto). Si el archivo declara un bloque provider de aws, DEBE contener un bloque assume_role apuntando al rol terraform-iac (role_arn = \"arn:aws:iam::${var.aws_account_id}:role/terraform-iac\") — un provider sin assume_role es una violación del estándar y no debe escribirse; agrega el bloque y la variable obligatoria aws_account_id en su lugar. El bloque backend \"s3\" va únicamente en backend.tf: si lo detectas en versions.tf o providers.tf, muévelo. Y nunca escribas un backend comentado o con placeholders — si falta el bucket o la region, pregúntaselos al usuario antes de escribir. Si falta alguna convención, corrígela antes de escribir. Si el archivo que se va a escribir no es un archivo Terraform (.tf), omite esta verificación."
  }
}
```

**`.kiro/hooks/review-terraform-standards.kiro.hook`**

```json
{
  "enabled": true,
  "name": "Revisar Terraform contra los estándares",
  "description": "Revisa todos los .tf contra los estándares cuando cambia el steering",
  "version": "1.1.0",
  "when": {
    "type": "fileEdited",
    "patterns": [".kiro/steering/*.md", ".kiro/skills/*"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "Se acaba de actualizar un archivo de steering o una skill del workspace. Lee la última versión de todos los archivos de steering en .kiro/steering/ y de las skills en .kiro/skills/ (si existen), y luego revisa todos los archivos Terraform de la carpeta para asegurar que cumplen los estándares y convenciones vigentes. Reporta cualquier desviación y sugiere o aplica las correcciones."
  }
}
```

### 3. Confirmar al usuario

Indica qué hooks quedaron activos y recuérdale que el steering de convenciones se aplica
automáticamente en cada interacción, sin que tenga que pedirlo.

---

## Inicio rápido

### Prerrequisitos

- Kiro IDE instalado
- Python 3.10+
- `uvx` instalado (`pip install uv`)
- Terraform >= 1.10 instalado (necesario para el locking nativo de state en S3)
- Personal Access Token (PAT) de GitHub con scope `repo`

### Paso 1: Instalar el Power

1. Abre Kiro IDE
2. Ve a la extensión **Kiro Powers**
3. Haz clic en **"Add Custom Power"**
4. Pega la URL: `https://github.com/<TuOrg>/kiro-power-terraform-standards`
5. Haz clic en **Install**

> **Si el repositorio es privado**, la instalación por URL no funciona: Kiro no puede
> autenticarse y el clone falla en silencio aunque muestre un mensaje de éxito.
> La vía soportada es clonar el repo y usar **Import power from a folder**
> ([documentación oficial](https://kiro.dev/docs/powers/installation/)).

### Paso 2: Configurar las variables de entorno

**Obligatoria:**

```bash
export GITHUB_TOKEN="ghp_your_token_here"
```

**Opcional (para resolver el backend automáticamente):**

```bash
export BACKEND_CONFIG='{
  "backends": {
    "prod": {"bucket": "your-tfstate-prod-bucket", "region": "us-east-1"},
    "sdlc": {"bucket": "your-tfstate-sdlc-bucket", "region": "us-east-1"}
  },
  "environment_mapping": {
    "prod": "prod",
    "staging": "sdlc",
    "dev": "sdlc",
    "qa": "sdlc"
  }
}'
```

> Si `BACKEND_CONFIG` no está definido, Kiro te pedirá el nombre del bucket y la region de forma interactiva.

### Paso 3: Abrir tu proyecto

```bash
git clone git@github.com:<TuOrg>/my-project-iac.git
cd my-project-iac
# Ábrelo en Kiro IDE
```

### Paso 4: Pedirle a Kiro lo que necesitas en lenguaje natural

Eso es todo. Ya puedes empezar.

---

## Ejemplos de prompts

### Listar los módulos disponibles

> "¿Qué módulos de terraform tenemos disponibles?"

Kiro llama a `search_modules` y devuelve la lista completa de módulos aprobados con su última versión.

### Crear una VPC

> "Creame una VPC con segmento 10.50.0.0/16 en us-east-1 para el proyecto onboarding, equipo platform, en dev"

Kiro va a:
1. Validar que el CIDR sea un /16
2. Encontrar el módulo de VPC y su última versión
3. Resolver la configuración del backend (por tool o de forma interactiva)
4. Generar todos los archivos: `versions.tf`, `backend.tf`, `providers.tf`, `variables.tf`, `terraform.tfvars`, `main.tf`, `outputs.tf`, `data.tf`

### Crear varios recursos

> "Necesito una VPC y una instancia EC2 t3.medium para el proyecto payments en prod"

Kiro busca ambos módulos, obtiene sus interfaces y los compone conectando correctamente los outputs de uno con los inputs del otro.

### Consultar el detalle de un módulo

> "¿Qué variables recibe el módulo de EC2?"

Kiro llama a `get_module("ec2")` y te muestra la interfaz completa: variables, tipos, descripciones y outputs.

---

## Tools MCP disponibles

| Tool | Descripción |
|------|-------------|
| `search_modules(query)` | Busca módulos de Terraform disponibles en la organización <TuOrg> |
| `get_module(service_name)` | Obtiene README, variables, outputs y última versión de un módulo |
| `list_module_versions(service_name)` | Lista todos los tags git (versiones) de un módulo |
| `scaffold_terraform(service_name, variables)` | Genera la estructura completa de archivos Terraform para un módulo |
| `get_backend_config(team, project, environment, bucket?, region?)` | Obtiene la configuración del backend S3 para `backend.tf`. Sin `BACKEND_CONFIG`, pide los datos al usuario y se vuelve a llamar con `bucket` y `region` |

---

## Convenciones que se aplican

### Política module-only
- Toda la infraestructura DEBE aprovisionarse mediante módulos privados de <TuOrg> (`terraform-aws-module-*`)
- Los bloques `resource` sueltos NO están permitidos
- Si no existe módulo para un servicio, Kiro te avisa y sugiere contactar al platform team

### Estructura de archivos
Todo proyecto debe tener: `versions.tf`, `backend.tf`, `providers.tf`, `variables.tf`, `terraform.tfvars`, `data.tf`, `main.tf`, `outputs.tf`

### Versionado
- `required_version` de Terraform >= 1.10
- Versión del provider de AWS siempre pineada: `~> 5.0` en root modules, `>= 5.0, < 7.0` en módulos reutilizables
- El `.terraform.lock.hcl` se commitea al control de versiones

### Seguridad y autenticación
- **El `assume_role` es obligatorio en todo bloque provider de `aws`** — el scaffold siempre lo genera y el hook de pre-write rechaza los providers que no lo tengan
- Sin access keys estáticas de AWS — todas las ejecuciones asumen el rol IAM `terraform-iac` (credenciales temporales)
- `aws_account_id` es una variable obligatoria del root module (12 dígitos, validada, sin default) que alimenta el ARN del rol
- El CI/CD se autentica vía federación OIDC contra el rol `terraform-iac`
- Los secretos viven en AWS Secrets Manager, nunca en el código, los tfvars ni el state; los outputs sensibles se marcan con `sensitive = true`

### Tagging
Todos los recursos deben incluir: `Environment`, `Team`, `CostCenter`, `ManagedBy` (siempre "terraform") — aplicados vía `default_tags` en el bloque provider.

### Nombres
Los recursos siguen el patrón: `{team}-{project}-{environment}`

### VPC / networking
- El CIDR de la VPC debe ser un /16
- Se subdivide automáticamente en 4 subnets de /18 (2 públicas + 2 privadas)
- 1 IGW, 1 NAT GW, 2 Route Tables

### Backend
- S3 con locking nativo (`use_lockfile = true`), sin DynamoDB
- El bloque vive en `backend.tf`, nunca en `versions.tf` ni en `providers.tf`
- Patrón de key: `{team}/{project}/{environment}/terraform.tfstate`
- Si `BACKEND_CONFIG` no está definida, Kiro **te pregunta** el bucket y la region — nunca deja el backend comentado

---

## Hooks

Se crean en `.kiro/hooks/` del workspace durante el [onboarding](#onboarding) — no se
instalan con el Power. El directorio `hooks/` de este repo es la fuente de la que salen.

| Hook | Disparador | Qué hace |
|------|------------|----------|
| Validar Terraform al guardar | Cualquier archivo `.tf` guardado | Ejecuta `terraform fmt -check -diff`, `terraform validate` y `tflint` (si está instalado) |
| Verificar convenciones de Terraform | Antes de escribir cualquier archivo `.tf` | Verifica los tags obligatorios, las versiones pineadas y el `assume_role` obligatorio sobre `terraform-iac` |
| Revisar Terraform contra los estándares | Archivo de steering o skill actualizado | Revisa todos los archivos .tf contra las convenciones vigentes |

---

## Qué NO hace este Power

- ❌ No ejecuta `terraform apply` — eso es cosa de tu pipeline de CI/CD o de una decisión manual
- ❌ No crea módulos nuevos — contacta al platform team si falta alguno
- ❌ No adivina — si falta información, te la pregunta

---

## Resolución de problemas

| Problema | Solución |
|----------|----------|
| "Authentication failed" | Verifica que `GITHUB_TOKEN` esté definido y tenga scope `repo` |
| "Organization not found" | Verifica que el token tenga acceso a la organización `<TuOrg>` |
| "No modules found" | Verifica que los repos de módulos sigan la convención `terraform-aws-module-*` |
| "BACKEND_CONFIG not set" | Exporta la variable o proporciona bucket/region cuando Kiro te lo pida |
| Las tools no aparecen | Asegúrate de que `uvx` esté instalado y el Power activado en Kiro IDE |
| "could not read Username for 'https://github.com'" | `uvx` clona por HTTPS sin prompts. Redirige a SSH: `git config --global url."git@github.com:<TuOrg>/".insteadOf "https://github.com/<TuOrg>/"` |

---

## Arquitectura

```
kiro-power-terraform-standards/     ← Este Power
├── POWER.md                        ← Estás aquí (frontmatter + onboarding)
├── mcp.json                        ← Configuración del MCP server
├── steering/
│   └── terraform-conventions.md    ← Convenciones incluidas automáticamente
├── hooks/                          ← Fuente; Kiro NO los instala desde aquí
│   ├── terraform-validate-save.kiro.hook
│   ├── terraform-tag-check.kiro.hook
│   └── review-terraform-standards.kiro.hook
└── iam/                            ← Políticas del rol que asume Terraform

<tu-workspace>/                     ← Donde acaban los hooks
└── .kiro/hooks/                    ← Creados por el onboarding del Power

terraform-mcp-server/               ← El MCP Server (repo aparte)
└── src/terraform_mcp_server/
    └── server.py                   ← 5 tools: search, get, list, scaffold, backend
```
