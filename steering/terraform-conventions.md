---
inclusion: auto
---

# Convenciones Terraform de <TuOrg>

Este documento define las convenciones obligatorias para toda la Infraestructura como Código (IaC) con Terraform gestionada por <TuOrg>. Todos los agentes y colaboradores DEBEN seguir estos estándares sin excepción.

## Requisitos del provider

- **Versión de Terraform**: >= 1.10 (declárala siempre en `required_version`; 1.10+ es obligatorio para el locking nativo de state en S3 con `use_lockfile`)
- **Versión del provider de AWS**: DEBE estar pineada — nunca uses una restricción abierta `>=`, porque permite upgrades de major implícitos
  - **Root modules (proyectos)**: usa una restricción pesimista: `~> 5.0`
  - **Módulos reutilizables**: usa un rango acotado: `>= 5.0, < 7.0`
- Declara **siempre** `required_providers` dentro de un bloque `terraform`, ubicado en `versions.tf`
- **Commitea siempre el `.terraform.lock.hcl`** al control de versiones — fija las versiones exactas de los providers y evita upgrades implícitos. Nunca lo agregues al `.gitignore`.

```hcl
# versions.tf (root module)
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Source de los módulos

Todos los módulos de infraestructura viven en **repositorios privados de GitHub** bajo la organización `<TuOrg>`. Convención de nombres del repositorio:

```
terraform-aws-module-<service>
```

Usa **siempre** el formato de source git con un tag pineado:

```hcl
module "vpc" {
  source = "git::https://github.com/<TuOrg>/terraform-aws-module-vpc.git?ref=v1.0.0"
}
```

Reglas:
- NUNCA uses referencias a ramas (por ejemplo, `ref=main`). Pinea siempre a un tag de versión concreto.
- NUNCA uses el formato de source del Terraform Registry para estos módulos.
- El `<service>` corresponde al servicio de AWS o agrupación lógica (por ejemplo, `vpc`, `ec2`, `rds`, `s3`, `iam`, `eks`).

## Estructura de archivos

Todo proyecto Terraform DEBE seguir esta estructura de archivos:

| Archivo | Propósito |
|---------|-----------|
| `versions.tf` | Bloque `terraform`: `required_version` y `required_providers` |
| `backend.tf` | Bloque `terraform` con el `backend "s3"` — **nunca** en `versions.tf` ni en `providers.tf` |
| `providers.tf` | Configuración del bloque `provider`: region, `assume_role` obligatorio y `default_tags` |
| `variables.tf` | Todas las declaraciones de variables de entrada |
| `terraform.tfvars` | Valores de las variables (específicos del environment) |
| `data.tf` | Todos los data sources |
| `main.tf` | Llamadas a módulos y orquestación de recursos |
| `outputs.tf` | Todas las declaraciones de outputs |

Se permiten archivos adicionales para proyectos complejos (por ejemplo, `locals.tf`), pero los anteriores son obligatorios.

## Política module-only

**CRÍTICO**: SOLO los módulos privados de <TuOrg> pueden aprovisionar recursos de infraestructura.

Flujo de trabajo:
1. Usa **siempre** primero la tool `search_modules` para comprobar si existe un módulo para el servicio requerido.
2. Si el módulo existe, úsalo con el source git correcto y la versión pineada.
3. Si NO existe módulo para el servicio requerido:
   - Informa al usuario de que no hay módulo disponible.
   - NO crees bloques `resource` sueltos.
   - Sugiere al usuario que solicite un módulo nuevo al platform team.

**Excepción**: los `data` sources, los `locals` y los bloques `terraform`/`provider` sí se permiten fuera de los módulos.

## Estándar de tagging

Todos los recursos de AWS DEBEN incluir estos tags:

| Tag | Descripción | Ejemplo |
|-----|-------------|---------|
| `Environment` | Environment de despliegue | `dev`, `staging`, `prod` |
| `Team` | Equipo propietario | `platform`, `backend`, `data` |
| `CostCenter` | Código de imputación de costes | `CC-1234` |
| `ManagedBy` | Siempre `terraform` | `terraform` |

**Implementación**: usa `default_tags` en el bloque provider de AWS para que todos los recursos hereden los tags obligatorios:

```hcl
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::${var.aws_account_id}:role/terraform-iac"
    session_name = "terraform-${var.team}-${var.environment}"
  }

  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
    }
  }
}
```

El bloque `assume_role` no es opcional — ver [Autenticación y seguridad](#autenticación-y-seguridad).

## Convención de nombres

Todos los recursos DEBEN seguir este patrón de nombres:

```
{team}-{project}-{environment}
```

Ejemplos:
- `platform-onboarding-prod`
- `backend-payments-staging`
- `data-analytics-dev`

Usa variables para construir los nombres dinámicamente:

```hcl
locals {
  name_prefix = "${var.team}-${var.project_name}-${var.environment}"
}
```

## Estándar de VPC / networking

Todo despliegue de VPC DEBE seguir esta arquitectura:

### Componentes obligatorios
- 1 VPC
- 2 subnets públicas (una por AZ)
- 2 subnets privadas (una por AZ)
- 1 Internet Gateway
- 1 NAT Gateway (único, en la primera subnet pública)
- 2 Route Tables: 1 pública (compartida por ambas subnets públicas), 1 privada (compartida por ambas subnets privadas)

### Reglas de asignación de CIDR

**El CIDR de la VPC DEBE ser un /16** (65.536 IPs). No se permite ninguna otra máscara.

Cuando el usuario proporcione un CIDR /16 (por ejemplo, `10.0.0.0/16`), subdivídelo automáticamente en **4 subnets de /18** (16.384 IPs cada una, 16.379 utilizables) con este esquema fijo:

| Subnet | Offset | Máscara | Ejemplo (para 10.0.0.0/16) |
|--------|--------|---------|----------------------------|
| public-a (AZ-a)  | +0 (1er bloque) | /18 | 10.0.0.0/18   |
| public-b (AZ-b)  | +1 (2º bloque)  | /18 | 10.0.64.0/18  |
| private-a (AZ-a) | +2 (3er bloque) | /18 | 10.0.128.0/18 |
| private-b (AZ-b) | +3 (4º bloque)  | /18 | 10.0.192.0/18 |

**Lógica de cálculo**: usa `cidrsubnet(vpc_cidr, 2, N)`, donde N es el índice de la subnet (0-3). Esto añade 2 bits a la máscara (16+2=18) y genera 4 bloques iguales.

### Uso del módulo de VPC

Al llamar a `terraform-aws-module-vpc`, el usuario solo necesita proporcionar el CIDR /16. El módulo resuelve todo el subneteo internamente:

```hcl
module "vpc" {
  source = "git::https://github.com/<TuOrg>/terraform-aws-module-vpc.git?ref=<version>"

  vpc_cidr     = var.vpc_cidr # Debe ser un /16
  environment  = var.environment
  team         = var.team
  cost_center  = var.cost_center
  project_name = var.project_name
  aws_region   = var.aws_region
}
```

El módulo automáticamente:
- Calcula 4 subnets de /18 con `cidrsubnet(vpc_cidr, 2, N)`
- Crea 2 subnets públicas + 2 privadas repartidas en 2 AZs
- Aprovisiona 1 Internet Gateway, 1 NAT Gateway y 2 Route Tables
- Aplica los tags obligatorios (Environment, Team, CostCenter, ManagedBy) a todos los recursos

### Validación

- Si el usuario proporciona un CIDR que NO es un /16, recházalo y pide un CIDR /16 válido.
- Usa siempre el módulo tal cual — NO pases los CIDR de las subnets manualmente, el módulo los calcula internamente.

## Estándar de backend

Todo el state de Terraform DEBE almacenarse remotamente en Amazon S3 con **locking nativo de S3** (sin DynamoDB — el locking con DynamoDB está deprecado).

El bloque va **siempre en su propio archivo `backend.tf`**, nunca dentro de `versions.tf` ni de `providers.tf`.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket       = "<resolved-bucket>"
    key          = "{team}/{project}/{environment}/terraform.tfstate"
    region       = "<resolved-region>"
    encrypt      = true
    use_lockfile = true
  }
}
```

Reglas:
- **Patrón de key**: `{team}/{project}/{environment}/terraform.tfstate` — siempre.
- **Cifrado**: `encrypt = true` es obligatorio (server-side encryption de S3 en reposo).
- **Locking**: `use_lockfile = true` es obligatorio (requiere Terraform >= 1.10).
- **Separación de environments**: producción usa su propio bucket dedicado; los environments no productivos (dev, qa, staging) comparten el bucket SDLC. Nunca guardes state de prod y no-prod en el mismo bucket.
- **Archivo**: el bloque `backend "s3"` vive en `backend.tf`. Un proyecto sin `backend.tf` está incompleto.
- **Resolución**: usa siempre la tool `get_backend_config` para resolver bucket, key y region.
- **Nunca dejes el backend sin resolver.** Si `get_backend_config` responde `needs_user_input`, **detente y pregunta al usuario** el bucket y la region en ese mismo turno, y vuelve a llamar a la tool con los parámetros `bucket` y `region`. Está PROHIBIDO:
  - dejar el bloque `backend` comentado,
  - escribirlo con placeholders (`<bucket>`, `TODO`, `CHANGE_ME`),
  - omitir `backend.tf` y seguir adelante,
  - o terminar el turno diciendo que el usuario lo complete después.

  Que falte una variable de entorno es motivo para **preguntar**, no para degradar el estándar.
- Los buckets de state DEBEN tener versioning habilitado y políticas IAM restrictivas (solo lectura para la mayoría de usuarios; escritura únicamente desde CI/CD).

## Autenticación y seguridad

- **Nunca uses access keys estáticas** (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` de larga duración) en código, tfvars ni bloques provider.
- **El `assume_role` es OBLIGATORIO en todo bloque provider de `aws`.** Un provider sin él es una
  violación del estándar: no lo generes, y señálalo para corregirlo cuando lo encuentres.
  No hay excepción para demos, ejecuciones locales ni environments desechables.
- Todas las ejecuciones de Terraform se autentican asumiendo el rol IAM dedicado **`terraform-iac`**:

```hcl
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::${var.aws_account_id}:role/terraform-iac"
    session_name = "terraform-${var.team}-${var.environment}"
  }

  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
    }
  }
}
```

- `aws_account_id` es una **variable obligatoria del root module** (12 dígitos, validada, sin default) en todos los proyectos — alimenta el `role_arn` de arriba. Va en `variables.tf` y `terraform.tfvars`, nunca hardcodeada en el bloque provider.
- **CI/CD**: los pipelines federan vía OIDC (por ejemplo, GitHub Actions → IAM) e intercambian tokens de corta duración por credenciales temporales sobre el rol `terraform-iac`. Sin secretos almacenados.
- **Ejecuciones locales**: los desarrolladores asumen `terraform-iac` con `aws sts assume-role` o con un perfil del AWS CLI configurado con `role_arn`; las credenciales son siempre temporales.
- El rol `terraform-iac` sigue least privilege: solo los permisos necesarios para gestionar los recursos de los módulos aprobados.
- **Secretos**: nunca hardcodees secretos en archivos `.tf` o `.tfvars`, y nunca guardes valores secretos en el state. Usa AWS Secrets Manager y referencia los secretos vía data sources.
- Todo output que exponga un valor sensible DEBE marcarse con `sensitive = true`.

## Análisis estático y CI/CD

- Todo pipeline DEBE ejecutar: `terraform fmt -check`, `terraform validate`, `tflint` (con el [ruleset de AWS](https://github.com/terraform-linters/tflint-ruleset-aws)) y `checkov`.
- El CI DEBE fallar cuando falte una restricción de versión del provider o no esté pineada (tflint lo detecta).
- `terraform apply` se ejecuta solo desde CI/CD o como acción manual explícita y revisada — nunca automáticamente desde el IDE.
