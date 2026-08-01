# Políticas IAM

El Power obliga a que **todo bloque `provider "aws"` lleve `assume_role`** sobre un rol
dedicado (`terraform-iac` por defecto). Aquí están las políticas que hacen falta para que ese
rol exista, sea asumible y pueda crear exactamente lo que los módulos necesitan — nada más.

Son de **mínimo privilegio para este laboratorio** (VPC + EC2 con SSM). Si añades módulos de
RDS, EKS o lo que sea, tendrás que ampliar `terraform-iac-policy.json`.

## Archivos

| Archivo | Se adjunta a | Para qué |
|---|---|---|
| `terraform-iac-policy.json` | El rol `terraform-iac` | Permisos de creación: VPC, subnets, IGW, NAT, route tables, security groups, instancias EC2 y el instance profile con SSM |
| `terraform-iac-trust.json` | Trust policy del rol | Quién puede asumirlo — variante usuario IAM (uso local) |
| `terraform-iac-trust-oidc.json` | Trust policy del rol | Variante federación OIDC con GitHub Actions (CI/CD) |
| `tfstate-bucket-access.json` | El **principal que ejecuta Terraform**, no el rol | Leer/escribir el state y el lockfile en S3 |

## Placeholders

| Placeholder | Ejemplo |
|---|---|
| `__ACCOUNT_ID__` | `123456789012` |
| `__REGION__` | `us-east-1` |
| `__IAC_PRINCIPAL__` | `terraform-runner` (el usuario IAM que asume el rol) |
| `__STATE_BUCKET__` | `mi-org-tfstate-sdlc` |
| `__GITHUB_ORG__` | `mi-org` |

```bash
ACCOUNT_ID=123456789012
REGION=us-east-1
IAC_PRINCIPAL=terraform-runner
STATE_BUCKET=mi-org-tfstate-sdlc

for f in *.json; do
  sed -i '' \
    -e "s/__ACCOUNT_ID__/$ACCOUNT_ID/g" \
    -e "s/__REGION__/$REGION/g" \
    -e "s/__IAC_PRINCIPAL__/$IAC_PRINCIPAL/g" \
    -e "s/__STATE_BUCKET__/$STATE_BUCKET/g" "$f"
done
```

## Montaje

```bash
# 1. El principal que va a ejecutar Terraform
aws iam create-user --user-name "$IAC_PRINCIPAL"
aws iam create-access-key --user-name "$IAC_PRINCIPAL"   # guarda las claves

# 2. El rol y sus permisos
aws iam create-role --role-name terraform-iac \
  --assume-role-policy-document file://terraform-iac-trust.json

aws iam put-role-policy --role-name terraform-iac \
  --policy-name terraform-iac-lab \
  --policy-document file://terraform-iac-policy.json

# 3. Acceso al bucket de state — va en el USUARIO, no en el rol (ver abajo)
aws iam put-user-policy --user-name "$IAC_PRINCIPAL" \
  --policy-name tfstate-bucket-access \
  --policy-document file://tfstate-bucket-access.json

# 4. Permitir que el usuario asuma el rol
cat > /tmp/allow-assume.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"sts:AssumeRole",
"Resource":"arn:aws:iam::$ACCOUNT_ID:role/terraform-iac"}]}
EOF
aws iam put-user-policy --user-name "$IAC_PRINCIPAL" \
  --policy-name allow-assume-terraform-iac \
  --policy-document file:///tmp/allow-assume.json

# 5. Perfil local
aws configure --profile terraform-runner   # con las claves del paso 1
export AWS_PROFILE=terraform-runner
```

Y en `terraform.tfvars` del proyecto:

```hcl
aws_account_id = "123456789012"
```

## Dos cosas que no son obvias

**El backend de S3 NO usa el rol.** El `assume_role` vive en el bloque `provider`, pero el
backend se autentica antes de que exista provider alguno, con la cadena de credenciales por
defecto. Por eso `tfstate-bucket-access.json` se adjunta al **usuario**, no al rol: si lo
pones en el rol, el `plan` falla con `AccessDenied` sobre `terraform.tfstate.tflock` aunque
el rol tenga permisos de sobra.

Y ojo con el `.tflock`: `use_lockfile = true` **crea y borra** un objeto por cada operación,
así que `s3:DeleteObject` es obligatorio, y las acciones de objeto necesitan el ARN con `/*`
— `arn:aws:s3:::bucket` a secas solo sirve para `s3:ListBucket`.

**El usuario root no puede asumir roles.** No es un problema de permisos: AWS lo prohíbe por
diseño. Si lo intentas verás `No valid credential sources found` o un `AccessDenied` de STS
que no se arregla con ninguna política. Hay que crear un usuario IAM (paso 1).
