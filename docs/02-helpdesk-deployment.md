# Fase 2 — Helpdesk Deployment

**Estado**: 🔄 En progreso
**Días completados**: 4, 5, 6

---

## Día 4 — Deploy de osTicket

**Fecha**: Mayo 2026
**Estado**: ✅ Completado

### Instancia EC2

| Campo          | Valor                   |
| -------------- | ----------------------- |
| Nombre         | lab-osticket-server     |
| AMI            | Ubuntu Server 24.04 LTS |
| Instance type  | t2.micro (Free Tier)    |
| Storage        | 20 GiB EBS              |
| VPC            | lab-vpc                 |
| Subnet         | lab-public-subnet-1     |
| Security Group | lab-sg-ssh-http         |
| Key pair       | lab-key-pair (.pem)     |

### Stack LAMP instalado

| Componente | Versión   | Rol                                              |
| ---------- | --------- | ------------------------------------------------ |
| Ubuntu     | 24.04 LTS | Sistema operativo                                |
| Apache     | 2.x       | Servidor web — recibe solicitudes HTTP           |
| PHP        | 8.5.4     | Lenguaje — ejecuta el código de osTicket         |
| MariaDB    | latest    | Base de datos — almacena tickets y configuración |
| osTicket   | v1.18.1   | Aplicación de helpdesk                           | 

### Configuración de base de datos

| Parámetro             | Valor                                |
| --------------------- | ------------------------------------ |
| Base de datos         | osticket                             | 
| Usuario de aplicación | osticket_user                        |
| Permisos              | Solo sobre la base de datos osticket |
| Acceso                | localhost únicamente                 |

**Decisión de diseño**: Se creó un usuario de DB dedicado (osticket_user) en lugar de usar root. Principio de menor privilegio: si osTicket tuviera una vulnerabilidad, el impacto queda limitado a su propia base de datos.

### URLs del sistema

| Recurso             | URL             |
| ------------------- | --------------- |
| Panel administrador | http://[IP]/scp |
| Portal de usuarios  | http://[IP]/    |

*Nota: la IP pública cambia cada vez que se reinicia la instancia EC2.*

### Proceso de instalación resumido

1. Actualización del sistema (`apt update && apt upgrade`)
2. Instalación del stack LAMP
3. Creación de base de datos y usuario dedicado en MariaDB
4. Descarga y descompresión de osTicket v1.18.1
5. Configuración de Apache VirtualHost
6. Habilitación de módulo rewrite (`a2enmod rewrite`)
7. Asignación de permisos (`chown www-data`)
8. Completar wizard de instalación vía navegador
9. Eliminación de carpeta `/setup` (seguridad post-instalación)
10. Restricción de permisos en ost-config.php (`chmod 0644`)

### Lecciones aprendidas

1. **php-imap no disponible en Ubuntu 24.04**: el paquete fue removido del repositorio principal. osTicket funciona correctamente sin él para casos de uso de lab que no requieran recepción de emails por IMAP.

2. **Eliminar /setup post-instalación es crítico**: la carpeta de setup permite reinstalar osTicket sobre una instalación existente, borrando todos los datos. Su eliminación es un paso de seguridad estándar post-instalación, análogo a deshabilitar endpoints de debug en producción.

3. **Apache corre como www-data**: los archivos de la aplicación deben pertenecer al usuario del sistema operativo que corre Apache. Sin el `chown www-data`, Apache no puede leer los archivos y responde con error 403.

4. **El flujo request-response de una app web**: Apache recibe la solicitud HTTP → pasa archivos .php a PHP → PHP consulta MariaDB → PHP genera HTML → Apache responde al navegador. Entender este flujo permite identificar en qué capa falla una aplicación web.

---

## Día 5 — Configuración operativa de osTicket

**Fecha**: Junio 2026
**Estado**: ✅ Completado
**Modalidad**: 100% interfaz web, sin terminal

### Departamentos configurados

Los departamentos definen quién atiende cada tipo de ticket y permiten el ruteo automático de solicitudes.

| Departamento      | Tipo   | Propósito                                             |
| ----------------- | ------ | ----------------------------------------------------- |
| IT Support        | Public | Soporte técnico general, gestión de accesos, hardware |
| Security          | Public | Incidentes de seguridad, violaciones de política      |
| HR                | Public | Solicitudes de RRHH, onboarding, offboarding          |
| Support (Default) | Public | Consultas generales sin categoría específica          |
| Maintenance       | Public | Mantenimiento de sistemas (preexistente)              |
| Sales             | Public | Área comercial (preexistente)                         |

### SLAs configurados

Los SLAs (Service Level Agreements) definen el tiempo máximo de respuesta según la urgencia. Son el principal KPI de un equipo de helpdesk.

| SLA      | Grace Period | Schedule        | Caso de uso                                              |
| -------- | ------------ | --------------- | -------------------------------------------------------- |
| Critical | 2 horas      | 24/7            | Caída de sistema productivo, brecha de seguridad activa  |
| High     | 4 horas      | 24/7            | Usuario sin acceso a sistema crítico, incidente activo   |
| Standard | 24 horas     | Lun-Vie 8am-5pm | Solicitudes de acceso, problemas de hardware no urgentes |
| Low      | 72 horas     | Lun-Vie 8am-5pm | Consultas generales, mejoras, solicitudes no urgentes    |

**Decisión de diseño**: Critical y High tienen schedule 24/7 porque en entornos bancarios regulados (como BIND) los incidentes críticos no esperan horario comercial. Un usuario sin acceso a sistemas de pagos o un incidente de seguridad activo requieren respuesta inmediata independientemente del horario.

### Help Topics configurados

Los Help Topics son las categorías que ven los usuarios al crear un ticket. Cada uno está vinculado a un departamento y SLA, permitiendo routing automático.

| Help Topic        | Departamento | SLA      | Descripción                                     |
| ----------------- | ------------ | -------- | ----------------------------------------------- |
| Access Request    | IT Support   | Standard | Solicitudes de acceso a sistemas o aplicaciones |
| Account Lockout   | IT Support   | High     | Usuarios bloqueados que no pueden ingresar      |
| Hardware Issue    | IT Support   | Standard | Problemas con equipos físicos                   |
| Security Incident | Security     | Critical | Incidentes de seguridad activos                 |
| General Inquiry   | Support      | Low      | Consultas generales sin urgencia                |

**Decisión de diseño**: Account Lockout tiene SLA High porque en trabajo remoto un usuario sin acceso es un empleado improductivo. En banca, puede impactar operaciones críticas del negocio.

### Agentes configurados

| Agente       | Username     | Departamento | Rol           | Propósito                                   |
| ------------ | ------------ | ------------ | ------------- | ------------------------------------------- |
| Martin Salvo | admin#05     | Todos        | Administrator | Administrador del sistema, acceso completo  |
| Junior Agent | junior.agent | IT Support   | View only     | Simulación de agente con permisos limitados |

**Decisión de diseño**: El agente junior tiene rol "View only" para simular el principio de menor privilegio en el equipo de soporte. En producción, un agente N1 no debería poder modificar configuraciones del sistema ni acceder a datos de otros departamentos.

### Formulario custom — Access Request Form

Formulario personalizado para estandarizar las solicitudes de acceso. Garantiza que cada solicitud tenga la información mínima necesaria para ser procesada correctamente y auditada.

| Campo                  | Tipo         | Req | Variable      | Descripción                              |
| ---------------------- | ------------ | --- | ------------- | ---------------------------------------- |
| System Name            | Short Answer | ✅ | system_name   | Sistema o aplicación solicitada          |
| Business Justification | Long Answer  | ✅ | justification | Razón de negocio que justifica el acceso |
| Manager Name           | Short Answer | ✅ | manager_name  | Manager que aprueba la solicitud         |
| Access Level           | Choices      | ✅ | access_level  | Read Only / Read Write / Admin           |

**Decisión de diseño**: El campo Access Level con opciones predefinidas evita solicitudes ambiguas y reduce el tiempo de análisis. En BIND, la falta de estandarización en solicitudes de acceso era una fuente frecuente de retrabajos y escalamientos innecesarios.

### Flujo de routing de un ticket

```
Usuario crea ticket
        ↓
Selecciona Help Topic (ej: Account Lockout)
        ↓
osTicket asigna automáticamente:
  → Departamento: IT Support
  → SLA: High (4 horas)
  → Si es Access Request: muestra Access Request Form
        ↓
Ticket aparece en bandeja del agente asignado
        ↓
Agente triagea, resuelve y documenta
        ↓
Usuario recibe notificación de resolución
```

### Lecciones aprendidas

1. **Los formularios custom requieren variable names obligatorios**: osTicket no guarda un campo como requerido si no tiene variable name asignado. Este requisito no está documentado claramente en la UI y genera comportamientos inesperados.

2. **La duplicación de campos es un bug conocido de osTicket**: al intentar guardar un formulario sin completar todos los campos requeridos, osTicket duplica los campos en lugar de mostrar un error claro. Solución: completar todos los variable names antes de hacer Save.

3. **Help Topics como punto de entrada del routing**: la configuración de Help Topics es el único punto donde se vinculan departamento, SLA y formulario en una sola acción. Un Help Topic mal configurado rompe todo el flujo de asignación automática.

4. **Principio de menor privilegio en agentes**: configurar roles diferenciados desde el inicio es una buena práctica operativa y de seguridad. Un agente N1 con acceso de administrador es un riesgo de integridad de datos y configuración del sistema.

---

## Día 6 — Integración de email con Amazon SES

**Fecha**: Junio 2026
**Estado**: ✅ Completado (con limitación documentada)
**Herramientas**: AWS SES, swaks, MariaDB

### ¿Qué es Amazon SES?

Amazon Simple Email Service (SES) es el servicio de envío de emails de AWS. Las empresas lo usan para notificaciones transaccionales (confirmaciones de tickets, alertas de sistema) porque escala a millones de emails y tiene métricas de deliverability integradas. Es la alternativa enterprise a usar Gmail o SMTP de terceros.

**Sandbox mode**: las cuentas nuevas de AWS operan en sandbox, donde SES solo puede enviar emails a direcciones explícitamente verificadas. Esto previene el abuso del servicio. Salir del sandbox requiere una solicitud formal a AWS.

### Configuración realizada

#### Identidad verificada en SES

| Campo             | Valor                     |
| ----------------- | ------------------------- |
| Tipo de identidad | Email address             |
| Email verificado  | martin.salvo616@gmail.com |
| Región            | us-east-1                 |
| Estado            | Verified                  |
| Modo              | Sandbox                   |

#### Credenciales SMTP

| Campo         | Valor                              |
| ------------- | ---------------------------------- |
| Usuario IAM   | osticket-smtp-user-2               |
| SMTP Endpoint | email-smtp.us-east-1.amazonaws.com |
| Puerto        | 587                                |
| Encriptación  | STARTTLS / TLS 1.3                 |

**Nota de seguridad**: Durante la sesión las credenciales originales (osticket-smtp-user) fueron expuestas accidentalmente en un canal de comunicación. Se aplicó el procedimiento estándar de respuesta: el usuario original fue eliminado inmediatamente y se generaron nuevas credenciales (osticket-smtp-user-2). Las credenciales comprometidas nunca fueron usadas maliciosamente dado el contexto de lab, pero el procedimiento correcto se aplica independientemente del contexto.

### Verificación funcional con swaks

swaks (Swiss Army Knife for SMTP) es una herramienta CLI para testear servidores SMTP. Se usó para verificar la funcionalidad de SES directamente desde la EC2.

**Comando ejecutado:**
```bash
swaks --to martin.salvo616@gmail.com \
      --from martin.salvo616@gmail.com \
      --server email-smtp.us-east-1.amazonaws.com \
      --port 587 \
      --auth LOGIN \
      --auth-user [SMTP_USERNAME] \
      --auth-password [SMTP_PASSWORD] \
      --tls \
      --header "Subject: CloudOps Lab - SES Test" \
      --body "SES integration test from CloudOps Helpdesk Lab."
```

**Resultado:**

| Paso | Estado |
| --------------------------- | ----------------------------------------- |
| Conexión TCP al servidor    | Connected                                 |
| Negociación STARTTLS        | TLS started (TLSv1.3, AES-256-GCM-SHA384) |
| Verificación de certificado | CA verification passed                    |
| Autenticación LOGIN         | 235 Authentication successful             |
| MAIL FROM                   | 250 Ok                                    |
| RCPT TO                     | 250 Ok                                    |
| Entrega del mensaje         | 250 Ok (Message-ID confirmado por SES)    |

**Email recibido en bandeja de spam**: esperado en sandbox mode. SES no tiene reputación de dominio establecida. En producción se resuelve configurando registros SPF, DKIM y DMARC en el DNS del dominio.

### Problema encontrado: integración UI osTicket-SES

#### Descripción

Al intentar configurar las credenciales SMTP en la interfaz de osTicket (Emails → Emails → Outgoing SMTP → Basic Authentication), la UI mostraba el error "PROTOCOL Required / Host, Port & Protocol Required" y no permitía ingresar username y password.

#### Investigación

1. Se exploró la configuración en Remote Mailbox — la UI de osTicket requiere que el Remote Mailbox tenga un Protocol seleccionado antes de aceptar credenciales SMTP en Outgoing, aunque Remote Mailbox esté deshabilitado.
2. Se investigó la base de datos MariaDB para entender la estructura de almacenamiento de configuración SMTP (`SHOW TABLES`, `DESCRIBE ost_email_account`, `SELECT` en tablas relevantes).
3. Se intentó insertar la configuración SMTP directamente en la DB (`ost_email_account`, `ost_config`), pero osTicket almacena las credenciales con encriptación interna no documentada.

#### Causa raíz

Bug de validación cruzada en osTicket v1.18.1: la configuración de credenciales en "Outgoing SMTP" valida el estado del "Remote Mailbox" aunque sean configuraciones independientes. Cuando Remote Mailbox no tiene Protocol seleccionado, la validación falla incluso si Remote Mailbox está deshabilitado.

#### Decisión técnica

Se priorizó verificar la funcionalidad a nivel de infraestructura mediante swaks, confirmando que la integración SES → EC2 funciona correctamente. Continuar forzando la UI hubiera consumido tiempo sin valor adicional de aprendizaje.

**Workaround**: tickets creados manualmente desde la interfaz web. La integración UI queda documentada como limitación conocida de la versión.

**Resolución futura**: actualizar osTicket a versión posterior a v1.18.1, o usar la API REST de osTicket para automatizar la creación de tickets desde scripts externos como alternativa a la integración de email.

### Lecciones aprendidas

1. **Rotación de credenciales ante exposición**: el procedimiento es siempre revocar primero, generar nuevas después, independientemente del contexto (lab o producción). No asumir que una exposición es inofensiva.

2. **Verificar funcionalidad en capas**: cuando la interfaz bloquea una configuración, verificar directamente la capa de infraestructura. swaks confirmó que SES funciona aunque la UI de osTicket tenga un bug. Saber en qué capa está el problema ahorra tiempo y evita conclusiones incorrectas.

3. **SPF, DKIM y DMARC en producción**: la entrega a spam en sandbox mode ilustra por qué la reputación de dominio importa en producción. En cualquier rol IT que incluya gestión de sistemas de notificaciones, configurar correctamente los registros de autenticación de email es parte del trabajo.

4. **Documentar limitaciones honestamente**: los equipos técnicos confían más en documentación que incluye problemas encontrados, su causa, y las decisiones tomadas, que en documentación que solo muestra éxitos.

5. **swaks como herramienta de diagnóstico SMTP**: permite verificar conectividad, autenticación TLS y entrega en un solo comando. Útil en cualquier troubleshooting de sistemas de email empresarial.

---

## Estado actual — Fase 2

| Día   | Contenido                                                  | Estado      |
| ----- | ---------------------------------------------------------- | ----------- |
| Día 4 | Deploy de osTicket en EC2                                  |  Completado |
| Día 5 | Configuración operativa (SLAs, departamentos, formularios) |  Completado |
| Día 6 | Integración email con SES                                  |  Completado |
| Día 7 | Operación simulada (tickets reales)                        |  Completado |

**Costo acumulado**: USD $0.2

---
# Día 7 — Operación simulada

**Fecha de ejecución**: Junio 8, 2026
**Estado**: ✅ Completado
**Sistema**: osTicket v1.18.1 — CloudOps Lab Helpdesk

---

## Resumen ejecutivo

Simulación completa de una jornada de trabajo de un agente de IT Support. Se crearon y resolvieron 10 tickets cubriendo las 5 categorías configuradas en el sistema, atendidos en orden de prioridad SLA. Cada ticket incluye nota interna de investigación y respuesta profesional al usuario.

---

## Métricas de la sesión

| Métrica | Valor |
| ------------------------- | ------------------------------------- |
| Tickets creados           | 10                                    |
| Tickets resueltos         | 10                                    |
| Tasa de resolución        | 100%                                  |
| Agente                    | Martin Salvo (admin#05)               |
| Usuarios simulados        | Ana García, Carlos López, Diana Pérez |
| Tiempo total de operación | ~35 minutos                           |

---

## Tickets atendidos — por orden de prioridad SLA

### 🔴 Critical (SLA: 2 horas / 24x7)

#### Ticket #356463 — Phishing email impersonating IT department
- **Usuario**: Carlos López — Finance
- **Categoría**: Security Incident
- **Descripción**: Usuario recibió email de dominio falso `c0mpany.com` solicitando credenciales
- **Investigación**:
  - Identificado dominio typosquatting (c0mpany vs company)
  - Verificado que usuario no clickeó ningún link
  - Revisados logs del email gateway — 3 empleados adicionales recibieron el mismo email
  - Bloqueado dominio remitente en email gateway
  - Sin credenciales comprometidas
- **Resolución**: Dominio bloqueado, email eliminado de todos los buzones, alerta de seguridad enviada a la empresa
- **Estado**: ✅ Resuelto

---

#### Ticket #149712 — Unauthorized login attempt
- **Usuario**: Ana García — Marketing
- **Categoría**: Security Incident
- **Descripción**: Intento de login desde IP extranjera a las 3am, usuario no lo inició
- **Investigación**:
  - IP origen: 185.220.101.45 (Tor exit node, Romania)
  - Intento fallido — contraseña incorrecta utilizada
  - Sin acceso no autorizado confirmado
  - Usuario ya había cambiado la contraseña de forma proactiva
  - MFA no estaba habilitado en la cuenta
- **Resolución**: Cuenta segura confirmada, recomendación de habilitar MFA, monitoreo adicional activado
- **Estado**: ✅ Resuelto

---

### 🟠 High (SLA: 4 horas / 24x7)

#### Ticket #757352 — CRM Account Lockout
- **Usuario**: Diana Pérez — Sales
- **Categoría**: Account Lockout
- **Descripción**: Cuenta bloqueada en CRM, usuario no recuerda haber cambiado contraseña
- **Investigación**:
  - Cuenta bloqueada tras 3 intentos fallidos a las 8:42am
  - Sin actividad sospechosa — probable olvido de contraseña post-weekend
  - Identidad verificada via directorio de empleados
- **Resolución**: Cuenta desbloqueada, contraseña temporal establecida, MFA recomendado
- **Estado**: ✅ Resuelto

---

### 🟡 Standard (SLA: 24 horas / Lun-Vie)

#### Ticket #766382 — Power BI access request
- **Usuario**: Carlos López — Finance
- **Categoría**: Access Request
- **Descripción**: Solicitud de acceso Read/Write a Power BI para reportes mensuales
- **Investigación**:
  - Aprobación de manager Diego Fernández verificada vía email
  - Licencia actual: M365 Basic (insuficiente para Power BI)
  - Licencias Power BI Pro disponibles en el departamento Finance
- **Resolución**: Licencia Power BI Pro asignada, usuario agregado al grupo de seguridad Finance-Reports
- **Estado**: ✅ Resuelto

---

#### Ticket #271156 — Laptop noise and slow
- **Usuario**: Diana Pérez — Sales
- **Categoría**: Hardware Issue
- **Descripción**: Laptop con ruido de ventilador, sobrecalentamiento y lentitud por 2 días
- **Investigación**:
  - CPU al 94%, disco al 89% de capacidad (solo 4GB libres)
  - Ventilador a RPM máximo por thermal throttling
  - Última actualización de OS: hace 47 días
  - Múltiples procesos en background consumiendo recursos
- **Resolución**: Limpieza de archivos temporales (liberados 8GB), procesos innecesarios terminados, OS y drivers actualizados
- **Estado**: ✅ Resuelto

---

#### Ticket #141828 — Admin access Sales dashboard
- **Usuario**: Diana Pérez — Sales
- **Categoría**: Access Request
- **Descripción**: Solicitud de acceso Admin al Sales Dashboard para gestionar reportes del equipo
- **Investigación**:
  - Aprobación de manager Laura Gómez verificada vía email
  - Acceso actual: Read/Write
  - Admin access justificado por responsabilidades de team lead
  - Principio de menor privilegio aplicado — revisión confirmó que Admin es necesario
- **Resolución**: Permisos actualizados a Admin, cambio registrado en access management registry
- **Estado**: ✅ Resuelto

---

#### Ticket #533983 — Monitor colors distorted
- **Usuario**: Carlos López — Finance
- **Categoría**: Hardware Issue
- **Descripción**: Monitor externo con colores distorsionados, reconectar cable no resolvió
- **Investigación**:
  - Probado con cable diferente — problema persiste
  - Probado monitor con laptop diferente — problema persiste
  - Conclusión: falla de hardware en el monitor
  - Monitor tiene 4 años de uso
  - Sin repuesto inmediato disponible
- **Resolución**: Solicitud de reemplazo enviada a procurement (ETA 3-5 días hábiles), usuario continúa trabajando en pantalla del laptop
- **Estado**: ✅ Resuelto

---

### 🟢 Low (SLA: 72 horas / Lun-Vie)

#### Ticket #338277 — VPN setup for remote work
- **Usuario**: Ana García — Marketing
- **Categoría**: General Inquiry
- **Descripción**: Instrucciones para configurar VPN en laptop personal para home office
- **Investigación**:
  - Cuenta activa verificada
  - Licencia de VPN confirmada para trabajo remoto
  - Preparadas instrucciones paso a paso para Windows
- **Resolución**: Instrucciones detalladas enviadas (descarga, instalación, conexión, verificación)
- **Estado**: ✅ Resuelto

---

#### Ticket #788865 — Microsoft 365 license upgrade
- **Usuario**: Carlos López — Finance
- **Categoría**: General Inquiry
- **Descripción**: Solicitud de upgrade de licencia M365 para acceder a Power Automate
- **Investigación**:
  - Licencia actual: M365 Basic (insuficiente para Power Automate)
  - Power Automate requiere M365 Business Standard o superior
  - Cuota de upgrade disponible en departamento Finance
  - No requiere aprobación de manager per política IT
- **Resolución**: Licencia actualizada a M365 Business Standard, acceso a Power Automate provisionado automáticamente
- **Estado**: ✅ Resuelto

---

#### Ticket #307624 — Locked out of Salesforce
- **Usuario**: Ana García — Marketing
- **Categoría**: Account Lockout
- **Descripción**: Bloqueada de Salesforce, email de reset no llega, reunión con cliente en 2 horas
- **Investigación**:
  - Cuenta bloqueada tras 5 intentos fallidos a las 8:47am
  - Email de reset enviado a carpeta de spam (confirmado con usuario)
  - Sin actividad sospechosa detectada
- **Resolución**: Cuenta desbloqueada, nuevo email de reset enviado, MFA recomendado
- **Estado**: ✅ Resuelto

---

## Proceso de atención aplicado

Para cada ticket se siguió este flujo estándar:

```
1. Recepción y triaje por SLA y prioridad
        ↓
2. Asignación del ticket al agente (admin#05)
        ↓
3. Investigación documentada en Internal Note
   (pasos realizados, hallazgos, herramientas usadas)
        ↓
4. Resolución del problema
        ↓
5. Comunicación profesional al usuario (Post Reply)
   (qué pasó, qué se hizo, recomendaciones)
        ↓
6. Cierre del ticket como Resolved
```

---

## Patrones identificados en la operación

**Seguridad:**
- 2 de 10 tickets fueron incidentes de seguridad — 20% del volumen total
- Ninguna cuenta fue comprometida en los incidentes
- MFA no estaba habilitado en las cuentas afectadas — oportunidad de mejora sistémica

**Accesos:**
- 2 solicitudes de acceso procesadas siguiendo principio de menor privilegio
- Ambas con verificación de aprobación de manager antes de provisionar

**Hardware:**
- 2 tickets de hardware: uno resuelto remotamente (limpieza/optimización), uno requirió reemplazo físico
- Diagnóstico diferencial aplicado (monitor: cable → GPU → hardware del monitor)

**Account Lockouts:**
- 2 lockouts en un mismo día sugiere oportunidad para campaña de educación sobre gestión de contraseñas y habilitación de MFA

---

## Lecciones aprendidas

1. **Orden de atención por SLA es crítico**: los tickets Critical deben atenderse primero independientemente del orden de llegada. Un incidente de seguridad activo tiene prioridad sobre cualquier solicitud de acceso o problema de hardware.

2. **Internal Notes como registro de auditoría**: documentar cada paso de investigación en las notas internas es tan importante como resolver el problema. En entornos regulados (como banca), el audit trail es un requisito de compliance.

3. **Verificación de identidad antes de cualquier cambio de acceso**: siempre confirmar la identidad del usuario y la aprobación del manager antes de provisionar o modificar permisos. Aplicado en todos los tickets de Access Request.

4. **Diagnóstico diferencial en hardware**: para problemas de hardware, aislar variables sistemáticamente (cambiar cable, probar con otro equipo) permite identificar la causa raíz antes de escalar o reemplazar equipos innecesariamente.

5. **MFA como recomendación recurrente**: 3 de los 10 tickets tuvieron MFA como recomendación de mejora. Esto indica que una campaña de habilitación de MFA reduciría el volumen de tickets de seguridad y lockouts.

6. **Permisos de agente en osTicket**: los agentes necesitan Extended Access explícito a cada departamento para ver los tickets correspondientes. Sin esta configuración, los tickets existen en el sistema pero son invisibles para el agente.

---

## Estado final

- ✅ 10 tickets creados cubriendo 5 categorías distintas
- ✅ Atendidos en orden correcto de prioridad SLA
- ✅ 100% de tickets resueltos en la sesión
- ✅ Documentación interna completa en cada ticket
- ✅ Comunicaciones profesionales al usuario en cada caso

**Próxima fase**: Fase 3 — Security Operations (Días 8-11)
