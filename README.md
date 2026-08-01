# kiro-power-terraform-standards

Un **Kiro Power** que convierte tus estándares de Terraform en gobernanza automática: el agente
de IA no puede generar infraestructura que se salte tus convenciones, porque las reglas viajan
con el Power (steering), las tools solo saben hablar con tus módulos privados (MCP server) y
los hooks verifican antes de escribir y al guardar.

Está pensado como **plantilla**: clónalo, sustituye los placeholders por los datos de tu
organización y publícalo en tu propio repo.

## Qué hay dentro

```
POWER.md                          Manifiesto del Power (frontmatter + onboarding)
mcp.json                          Declaración del MCP server que aporta las tools
steering/
└── terraform-conventions.md      Convenciones inyectadas en cada interacción del agente
hooks/
├── terraform-validate-save.kiro.hook     fmt + validate + tflint al guardar un .tf
├── terraform-tag-check.kiro.hook         verificación previa a escribir un .tf
└── review-terraform-standards.kiro.hook  re-auditoría cuando cambia el steering
iam/
└── *.json                            Políticas de mínimo privilegio del rol que asume Terraform
```

> Los archivos de `hooks/` **no** los instala Kiro: son la fuente de la que la sección
> *Onboarding* de `POWER.md` los copia al `.kiro/hooks/` de cada workspace.

## Qué hay que personalizar

Todos los placeholders son literales, así que un buscar/reemplazar basta:

| Placeholder | Qué es | Dónde aparece |
|-------------|--------|---------------|
| `<TuOrg>` | Tu organización de GitHub y el nombre que quieras mostrar | `POWER.md`, `mcp.json`, `steering/` |
| `terraform-iac` | Nombre del rol IAM que asume Terraform | `mcp.json` (`IAC_ROLE_NAME`), `steering/`, hooks |
| `terraform-aws-module-` | Prefijo de los repos de tus módulos | `mcp.json` (`MODULE_PREFIX`) |
| `terraform-mcp-server` | Repo del MCP server que acompaña al Power | `mcp.json` |

```bash
grep -rl '<TuOrg>' . | xargs sed -i '' 's/<TuOrg>/mi-org/g'
```

Revisa también las convenciones de `steering/terraform-conventions.md` (tagging, patrón de
nombres, subneteo de VPC, estándar de backend): son opinionadas a propósito, cámbialas por
las tuyas.

El rol IAM que el Power exige en cada `provider "aws"` no se crea solo: en [`iam/`](./iam)
están las políticas de mínimo privilegio y los pasos para montarlo.

## Requisitos

- Kiro IDE
- Terraform >= 1.10 (necesario para el locking nativo de state en S3, `use_lockfile`)
- `uvx` (`pip install uv`) para arrancar el MCP server
- Un PAT de GitHub con scope `repo` en `GITHUB_TOKEN` si tus módulos son privados
- El MCP server que aporta las tools: `terraform-mcp-server`

## Instalación

En Kiro IDE → **Kiro Powers** → **Add Custom Power** → pega la URL del repo.

> Si el repositorio es **privado**, la instalación por URL no funciona: Kiro no puede
> autenticarse y el clone falla en silencio aunque muestre un mensaje de éxito. La vía
> soportada es clonar el repo y usar **Import power from a folder**
> ([documentación oficial](https://kiro.dev/docs/powers/installation/)).

El detalle completo —onboarding, ejemplos de prompts, tools disponibles, convenciones que se
aplican y resolución de problemas— está en [`POWER.md`](./POWER.md).

## Licencia

MIT
