# RSN N1 — Laboratorio IaC desde VS Code

**Objetivo**: desplegar y analizar siete flujos AWS con CloudFormation (YAML), AWS CLI, PowerShell y Git.

Archivos: `infra/core.yaml` (2 VPC, subredes, peering, S3 Gateway Endpoint, SG, 4 EC2, EFS, ALB y SSM endpoints opcionales), `infra/waf-global.yaml` (WAF CloudFront en us-east-1), `infra/edge.yaml` (CloudFront+OAC+bucket policy), `scripts/deploy.ps1`, `scripts/verify.ps1`, `scripts/cleanup.ps1`, y datos de D01/D02/D03.

## Preparación

1. En Learner Lab: Start Lab, AWS Details, credenciales temporales CLI. Recargar credenciales al reiniciar la sesión.
2. VS Code > Terminal PowerShell: `aws sts get-caller-identity` y `aws configure list`. Configurar región según la región permitida por el laboratorio.
3. Verificar **instance profile** real para EC2 en la consola o por CLI. `LabInstanceProfile` es solo el valor sugerido, no una garantía. Debe permitir S3 y SSM si se quiere utilizar Session Manager.
4. Crear repo Git privado y no subir credenciales ni evidencias privadas.
5. Confirmar permisos, cuotas y coste. La plantilla no se ha ejecutado sobre la cuenta del alumno.

## Comandos (PowerShell)

```powershell
cd .\RSN_N1_VSCode_Lab
aws sts get-caller-identity
aws cloudformation validate-template --region us-east-1 --template-body file://infra/core.yaml
.\scripts\deploy.ps1 -Region us-east-1 -Prefix n1-g03 -InstanceProfile LabInstanceProfile
.\scripts\verify.ps1 -Region us-east-1 -Prefix n1-g03
.\scripts\cleanup.ps1 -Region us-east-1 -Prefix n1-g03
```

Si Learner Lab no admite WAF, usar `-SkipWaf` en los tres scripts. Si no se pueden crear los endpoints de interfaz SSM, usar `-NoSsmEndpoints` en el despliegue, sabiendo que Session Manager y Run Command quedarán sin conectividad privada. WAF global **siempre** se crea en `us-east-1`, aun con la red de la práctica en otra región.

## Diseño de la práctica

- S3 es privado con Block Public Access. D02 se entrega mediante CloudFront y OAC; D01 y D03 solamente se leen desde la VPC corporativa por el S3 Gateway Endpoint, si la identidad IAM está autorizada.
- App, clientes EFS y planta carecen de IP pública. El ALB es público por HTTP 80 **solo en este laboratorio**; en producción se exige HTTPS para el portal.
- EFS: TLS+NFS/TCP 2049 mediante `amazon-efs-utils`, permisos POSIX 2001 (escritor) y 2002 (lector).
- La planta emulada está en una VPC separada y accede al servidor de aplicación por VPC Peering.
- Los cuatro endpoints SSM (dos por VPC) son opcionales y **generan coste por hora**; los S3 Gateway Endpoints no cobran tarifa de endpoint.

## Restricciones

- El perfil EC2 preexistente puede no tener los permisos necesarios; no se crean roles IAM para evitar suponer permisos de creación en Learner Lab.
- Algunos Learner Labs restringen CloudFront/WAF/EFS, endpoints, regiones y tipos de instancia. Verificar antes de desplegar.
- Los intentos de instalar `amazon-efs-utils` en EC2 privadas requieren que las fuentes de paquetes sean alcanzables sin NAT; comprobar los logs de `cloud-init` o habilitar acceso a repositorios a través de rutas/endpoints apropiados. **No se permite montar NFS sin TLS como atajo.**
- Las validaciones CLI no constituyen aceptación operativa ni pruebas de seguridad completas.
