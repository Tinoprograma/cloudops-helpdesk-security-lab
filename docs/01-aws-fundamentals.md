# Fase 1 — AWS Fundamentals

**Fecha de ejecución**: Mayo 25, 2026
**Región de trabajo**: us-east-1 (N. Virginia)
**Estado**: ✅ Completado

---

## Resumen ejecutivo

Configuración inicial de cuenta AWS nueva con enfoque en seguridad, monitoreo de costos y preparación para el despliegue del lab de 14 días.

---

## Configuración de seguridad

### Cuenta root
- **MFA habilitado**: ✅ (Google Authenticator)
- **Uso**: Solo emergencias
- **Mejor práctica**: Cuenta raíz protegida y prácticamente nunca usada

### Usuario IAM administrativo
- **Username**: martin-admin
- **Política**: AdministratorAccess
- **MFA habilitado**: ✅ (Google Authenticator)
- **Último login**: Mayo 25, 2026
- **Descripción**: Usuario de trabajo diario para todas las tareas del lab

**Decisión de diseño**: Se utilizó Google Authenticator en lugar de Authy porque es más simple para el caso de uso de lab personal (sin necesidad de backup en la nube).

---

## Monitoreo y alertas de costos

### Budgets configurados

| Budget            | Threshold | Estado |
| ----------------- | --------- | ------ |
| lab-budget-1-usd  | USD $1    | Activo |
| lab-budget-5-usd  | USD $5    | Activo |
| lab-budget-10-usd | USD $10   | Activo |

### Alertas habilitadas

| Alerta                    | Tipo             | Estado                                  |
| ------------------------- | ---------------- | --------------------------------------- |
| AWS Free Tier alerts      | Email            | Entregado                               |
| CloudWatch billing alerts | CloudWatch Alarm | Creado (SNS topic pending confirmation) |
| Cost Explorer             | Dashboard        | Habilitado (datos en 24hr)              |

**Decisión de diseño**: Tres budgets en escalera ($1, $5, $10) para detección temprana de cualquier cargo inesperado. El threshold de $1 es agresivo pero apropiado para Free Tier.

---

## Acceso a Billing

- **IAM user access to Billing information**: Habilitado en Account settings
- **Razón**: Permitir que martin-admin pueda ver datos de costos sin usar root

---

## Credenciales guardadas

✅ Guardadas en Google Password Manager:
- Email y contraseña root (acceso de emergencia)
- Email y contraseña martin-admin (trabajo diario)
- URL de sign-in específica de la cuenta: https://[account-id].signin.aws.amazon.com/console
- Backup codes de MFA (si fueron proporcionados)

---

## Lecciones aprendidas

1. **Orden de operaciones importa**: Habilitar CloudWatch billing alerts desde Billing preferences es un paso previo necesario antes de intentar crear alarms en CloudWatch.

2. **Permisos de Billing necesitan habilitación explícita**: Los usuarios IAM con AdministratorAccess aún necesitan que root habilite "IAM user and role access to Billing information" en Account settings. No es heredado automáticamente.

3. **Cost Explorer necesita 24hr de datos**: El dashboard está habilitado pero no muestra información inicial. Revisar al Día 2.

4. **SNS topic confirmation puede tardar**: El email de confirmación para SNS no llegó inmediatamente. Esto es normal en AWS y se reintentar según sea necesario.

---

## Configuración completada

- ✅ Cuenta AWS segura con MFA en root e IAM user
- ✅ Monitoreo de costos multinivel
- ✅ Usuario administrativo listo para trabajo diario
- ✅ Buenas prácticas de seguridad aplicadas desde el día 1
- ✅ Documentación completada

---

## Infraestructura de red (Día 2)

| Recurso          | Nombre              | Valor                          |
| ---------------- | ------------------- | ------------------------------ |
| VPC              | lab-vpc             | 10.0.0.0/16                    |
| Subnet 1         | lab-public-subnet-1 | 10.0.16.0/20 / us-east-1a      |
| Subnet 2         | lab-public-subnet-2 | 10.0.0.0/20 / us-east-1b       |
| Internet Gateway | lab-igw             | Attached a lab-vpc             |
| Security Group   | lab-sg-ssh-http     | SSH (My IP) + HTTP (0.0.0.0/0) |

**Decisión de diseño**: Se usaron subnets creadas automáticamente por AWS 
al crear la VPC en lugar de crear nuevas, ya que cubrían el mismo propósito.

## Identity layer (Día 3)

### IAM Identity Center

| Recurso    | Detalle                                |
| ---------- | -------------------------------------- |
| Directorio | Identity Center directory              |
| Portal URL | https://d-xxxxxxxxxx.awsapps.com/start |

### Usuarios creados

| Username     | Grupo     | Permission Set    |
| ------------ | --------- | ----------------- |
| ana.garcia   | Marketing | MarketingReadOnly |
| carlos.lopez | Finance   | FinanceBilling    |
| diana.perez  | Sales     | SalesReadOnly     |

### Verificación
Login desde portal URL con usuario ficticio: ✅ Funciona correctamente

---

## Lecciones aprendidas — Fase 1

1. **Orden de operaciones en AWS**: varios servicios requieren 
   habilitación explícita antes de poder usarlos 
   (Billing access para IAM users, CloudWatch billing alerts, 
   Cost Explorer).

2. **VPC crea recursos automáticamente**: al crear una VPC, 
   AWS genera subnets, route tables y otros recursos 
   por defecto que no son visibles hasta investigarlos.

3. **IAM Identity Center vs Active Directory**: IAM Identity 
   Center es la alternativa cloud-native moderna a AD 
   on-premise. Más simple de gestionar, sin servidores 
   que mantener, integrado nativamente con AWS.

4. **My IP en Security G

# Fase 2 — Helpdesk deployment

## Día 4 — Deploy de osTicket

**Fecha**: Mayo 2026
**Estado**: ✅ Completado

### Instancia EC2

| Campo          | Valor                   |
| -------------- | ----------------------- |
| Nombre         | lab-osticket-server     |
| AMI            | Ubuntu Server 24.04 LTS |
| Instance type  | t2.micro                |
| Storage        | 20 GiB                  |
| VPC            | lab-vpc                 |
| Subnet         | lab-public-subnet-1     |
| Security Group | lab-sg-ssh-http         |

### Stack instalado

| Componente | Versión |
| ---------- | ------- |
| Apache     | 2.x     |
| PHP        | 8.5.4   |
| MariaDB    | latest  |
| osTicket   | v1.18.1 |

### URLs

| Recurso         | URL             |
| --------------- | --------------- |
| Panel admin     | http://[IP]/scp |
| Portal usuarios | http://[IP]/    |

*Nota: IP pública cambia cada vez que se reinicia la instancia.*

### Lecciones aprendidas

1. **php-imap no disponible en Ubuntu 24.04**: el paquete 
   cambió de nombre. osTicket funciona sin él para casos 
   de uso básicos de lab.
2. **Email del admin distinto al del helpdesk**: osTicket 
   requiere emails diferentes para el sistema y el admin.
3. **Eliminar /setup post-instalación**: paso de seguridad 
   crítico para evitar reinstalaciones no autorizadas.

### Próximo paso

Día 5 — Configuración operativa de osTicket 
(departamentos, SLAs, formularios, agentes)