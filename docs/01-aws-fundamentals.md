# Fase 1 — AWS Fundamentals

**Fecha de ejecución**: Mayo 25, 2026
**Región de trabajo**: us-east-1 (N. Virginia)
**Estado**: ✅ Completado

---

## Resumen ejecutivo

Configuración inicial de cuenta AWS nueva con enfoque en seguridad, monitoreo de costos y preparación para el despliegue del lab de 14 días.

---

## Día 1 — Seguridad de cuenta y monitoreo de costos

### Configuración de seguridad

#### Cuenta root
- **MFA habilitado**: ✅ (Google Authenticator)
- **Uso**: Solo emergencias
- **Mejor práctica**: Cuenta raíz protegida y prácticamente nunca usada

#### Usuario IAM administrativo
- **Username**: martin-admin
- **Política**: AdministratorAccess
- **MFA habilitado**: ✅ (Google Authenticator)
- **Último login**: Mayo 25, 2026
- **Descripción**: Usuario de trabajo diario para todas las tareas del lab

**Decisión de diseño**: Se utilizó Google Authenticator en lugar de Authy porque es más simple para el caso de uso de lab personal (sin necesidad de backup en la nube). Authy tiene problemas de compatibilidad con el formato de números telefónicos argentinos en su proceso de registro.

---

### Monitoreo y alertas de costos

#### Budgets configurados

| Budget            | Threshold | Estado |
| ----------------- | --------- | ------ |
| lab-budget-1-usd  | USD $1    | Activo |
| lab-budget-5-usd  | USD $5    | Activo |
| lab-budget-10-usd | USD $10   | Activo |

**Decisión de diseño**: Tres budgets en escalera ($1, $5, $10) para detección temprana de cualquier cargo inesperado. El threshold de $1 es agresivo pero apropiado para Free Tier.

#### Alertas habilitadas

| Alerta                    | Tipo             | Estado     |
| ------------------------- | ---------------- | ---------- |
| AWS Free Tier alerts      | Email            | Entregado  |
| CloudWatch billing alerts | CloudWatch Alarm | Creado     |
| Cost Explorer             | Dashboard        | Habilitado |

---

### Acceso a Billing

- **IAM user access to Billing information**: ✅ Habilitado en Account settings
- **Razón**: Por defecto, los usuarios IAM con AdministratorAccess no pueden ver datos de billing. Requiere habilitación explícita desde la cuenta root en Account settings.

---

### Credenciales guardadas

✅ Guardadas en Google Password Manager:
- Email y contraseña root (acceso de emergencia)
- Email y contraseña martin-admin (trabajo diario)
- URL de sign-in específica de la cuenta
- Backup codes de MFA

---

## Día 2 — Infraestructura de red

### Recursos creados

| Recurso          | Nombre              | Valor                          |
| ---------------- | ------------------- | ------------------------------ |
| VPC              | lab-vpc             | 10.0.0.0/16                    |
| Subnet 1         | lab-public-subnet-1 | 10.0.16.0/20 / us-east-1a      |
| Subnet 2         | lab-public-subnet-2 | 10.0.0.0/20 / us-east-1b       |
| Internet Gateway | lab-igw             | Attached a lab-vpc             |
| Route Table      | Main route table    | 0.0.0.0/0 → lab-igw            |
| Security Group   | lab-sg-ssh-http     | SSH (My IP) + HTTP (0.0.0.0/0) |

### Decisiones de diseño

**Subnets**: Al crear la VPC, AWS generó automáticamente 4 subnets con rangos en el espacio 10.0.0.0/16. Se renombraron dos de ellas en lugar de crear nuevas, ya que cubrían el mismo propósito y evitaba conflictos de CIDR.

**Security Group**: SSH restringido a la IP del administrador (My IP) siguiendo el principio de menor exposición. HTTP abierto a 0.0.0.0/0 para que osTicket sea accesible desde el navegador.

**Nota operativa**: La IP del administrador es dinámica (ISP argentino). Si SSH falla al reconectar, el primer paso es verificar si la IP cambió y actualizar la regla del Security Group.

---

## Día 3 — Identity Layer

### AWS IAM Identity Center

| Recurso            | Detalle                              |
| ------------------ | ------------------------------------ |
| Tipo de directorio | Identity Center directory            |
| Región             | us-east-1                            |
| Portal URL         | [reemplazar con URL real del portal] |

### Usuarios creados

| Username     | Departamento | Grupo     | Permission Set    |
| ------------ | ------------ | --------- | ----------------- |
| ana.garcia   | Marketing    | Marketing | MarketingReadOnly |
| carlos.lopez | Finance      | Finance   | FinanceBilling    |
| diana.perez  | Sales        | Sales     | SalesReadOnly     |

### Permission Sets configurados

| Permission Set    | Política base  | Nivel de acceso                              |
| ----------------- | -------------- | -------------------------------------------- |
| MarketingReadOnly | ReadOnlyAccess | Solo lectura en toda la cuenta               |
| FinanceBilling    | Billing        | Acceso a información de costos y facturación |
| SalesReadOnly     | ReadOnlyAccess | Solo lectura en toda la cuenta               |

### Verificación

- Login desde portal URL con usuario ficticio: ✅ Funciona correctamente
- MFA forzado para todos los usuarios: ✅ Configurado
- Reset de contraseña via email (alias Gmail): ✅ Funciona con formato usuario+alias@gmail.com

### Decisión de diseño

Se eligió IAM Identity Center en lugar de Active Directory on-premise porque es la solución cloud-native moderna para gestión de identidades en entornos AWS. No requiere servidores adicionales, se integra nativamente con AWS, y refleja el stack real de startups y empresas tech que trabajan 100% en cloud. AD on-premise sería relevante en entornos híbridos o legacy.

---

## Lecciones aprendidas — Fase 1

1. **Orden de operaciones en AWS**: varios servicios requieren habilitación explícita antes de poder usarlos. CloudWatch billing alerts debe activarse desde Billing preferences antes de crear alarms. Cost Explorer tarda 24hr en mostrar datos iniciales.

2. **Permisos de Billing no son heredados**: los usuarios IAM con AdministratorAccess aún necesitan que root habilite "IAM user and role access to Billing information" en Account settings. No se hereda automáticamente por diseño de seguridad de AWS.

3. **VPC crea recursos automáticamente**: al crear una VPC, AWS genera subnets, route tables y otros recursos por defecto en el espacio de direcciones definido. Estos recursos no son visibles en la pantalla de creación pero existen y pueden causar conflictos al intentar crear subnets con CIDRs superpuestos.

4. **My IP en Security Groups**: restringir SSH a la IP específica del administrador es una práctica crítica de seguridad. En Argentina, los ISPs asignan IPs dinámicas que pueden cambiar. Si SSH falla inesperadamente, verificar si la IP cambió es el primer paso de troubleshooting.

5. **IAM Identity Center vs Active Directory**: IAM Identity Center es más simple de gestionar para entornos cloud-native. Sin servidores que mantener, sin licencias adicionales, y con integración nativa a servicios AWS. AD on-premise tiene sentido en entornos híbridos donde hay workloads on-premise que necesitan autenticación centralizada.

---

## Estado final — Fase 1

- ✅ Cuenta AWS segura con MFA en root e IAM user
- ✅ Monitoreo de costos multinivel (budgets + alarms + free tier alerts)
- ✅ Red base completa (VPC, subnets, IGW, route table, security group)
- ✅ Identity layer operativo (IAM Identity Center con 3 usuarios y grupos)
- ✅ Documentación completada

**Costo acumulado al cierre de Fase 1**: USD $0.00
