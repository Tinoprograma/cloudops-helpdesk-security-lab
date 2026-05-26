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

## Próximo paso

**Día 2 — Networking base (VPC, subnets, primera EC2)**

Consultar `PROJECT_ROADMAP.md` sección "Fase 1, Día 2" para los detalles específicos.