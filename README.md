# Demo: OIDC vs Access Keys en GitHub Actions → AWS ECR

Demuestra la diferencia entre autenticarse a AWS desde GitHub Actions usando **OIDC (buenas prácticas)** vs **Access Keys de larga duración (malas prácticas)**.

## Estructura

```
├── index.js                              # API Node.js (Express)
├── package.json
├── Dockerfile
└── .github/workflows/
    ├── deploy-oidc.yml                   # ✅ OIDC - credenciales temporales
    └── deploy-access-keys.yml            # ❌ Access Keys - credenciales permanentes
```

## Comparación

| Aspecto | ✅ OIDC | ❌ Access Keys |
|---------|---------|----------------|
| Credenciales almacenadas | Ninguna | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Duración del acceso | Temporal (~1h) | Permanente hasta rotación manual |
| Si se filtran | Token ya expirado, sin impacto | Acceso total hasta que se revoquen |
| Rotación | Automática (cada ejecución) | Manual |
| Principio de menor privilegio | Scoped por repo/branch/environment | Acceso amplio del IAM User |

## Prerequisitos AWS

### Para el workflow OIDC

1. **Crear Identity Provider en IAM:**
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. **Crear IAM Role** con trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": {
         "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
       },
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {
           "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
         },
         "StringLike": {
           "token.actions.githubusercontent.com:sub": "repo:TU_ORG/TU_REPO:*"
         }
       }
     }]
   }
   ```

3. **Adjuntar policy** al rol para ECR push:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": [
         "ecr:GetAuthorizationToken",
         "ecr:BatchCheckLayerAvailability",
         "ecr:PutImage",
         "ecr:InitiateLayerUpload",
         "ecr:UploadLayerPart",
         "ecr:CompleteLayerUpload"
       ],
       "Resource": "*"
     }]
   }
   ```

4. **Configurar en GitHub:** Variable `AWS_ROLE_ARN` con el ARN del rol creado.

### Para el workflow Access Keys

1. Crear un IAM User con permisos de ECR.
2. Guardar en GitHub Secrets: `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY`.

## Cómo funciona OIDC

```
GitHub Actions                         AWS
     │                                  │
     ├─ 1. Solicita token OIDC ────────►│
     │     (firmado por GitHub)          │
     │                                  │
     │◄─ 2. AWS valida token ───────────┤
     │     (verifica issuer, audience,   │
     │      subject/repo)                │
     │                                  │
     │◄─ 3. Retorna credenciales ───────┤
     │     temporales (STS)              │
     │                                  │
     ├─ 4. Usa creds para push ECR ────►│
     │                                  │
```

## Ejecución local (solo la API)

```bash
npm install
npm start
# GET http://localhost:3000/health
```
