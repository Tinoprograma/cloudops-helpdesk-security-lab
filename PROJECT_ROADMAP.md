# CloudOps Helpdesk + Security Lab

**Hoja de ruta del proyecto**

| Campo                    | Valor                                                           |
| :----------------------- | :-------------------------------------------------------------- |
| **Autor**                | Martín Salvo                                                    |
| **Inicio estimado**      | Mayo 2026                                                       |
| **Duración planificada** | 14 días                                                         |
| **Esfuerzo diario**      | 3-4 horas                                                       |
| **Presupuesto**          | USD $0 (AWS Free Tier estricto)                                 |
| **Repositorio**          | github.com/Tinoprograma/cloudops-helpdesk-security-lab          |
| **Rol objetivo**         | Jr. IT Support Specialist / SOC Analyst Trainee                 |

---

## 1\. Resumen ejecutivo

Construcción end-to-end de un entorno cloud que simula la infraestructura de IT Support y Security Operations de una pequeña empresa, desplegada íntegramente en AWS Free Tier. El proyecto demuestra capacidades técnicas en gestión de identidades, soporte a usuarios mediante sistema de ticketing, monitoreo de seguridad, y respuesta a incidentes.

El objetivo no es construir una solución de producción, sino producir un **portfolio técnico verificable** que evidencie competencias relevantes para roles de IT Support híbrido con foco en seguridad, alineado al camino profesional hacia SOC Jr.

---

## 2\. Objetivos y motivación

### Objetivo principal

Generar un proyecto público, documentado y demostrable que respalde técnicamente el perfil de "Jr IT Support + Security" en entrevistas y postulaciones, cerrando el gap percibido entre teoría (Security+) y ejecución práctica.

### Objetivos secundarios

1. Adquirir familiaridad operativa con AWS, una de las plataformas más demandadas en el mercado.
2. Producir contenido publicable (repo GitHub, demo video, post de LinkedIn) que aumente visibilidad profesional.
3. Generar material concreto para narrativas STAR en entrevistas futuras.
4. Establecer base técnica reutilizable para certificaciones posteriores (AWS Cloud Practitioner, CySA+).

### Competencias que el proyecto demuestra

| Categoría         | Competencia                         | Cómo se evidencia                                    |
| :---------------- | :---------------------------------- | :--------------------------------------------------- |
| IT Support        | Gestión de tickets y SLA            | Sistema osTicket configurado y operado               |
| IT Support        | Documentación técnica               | Procedimientos operativos por categoría de incidente |
| Identity & Access | Gestión moderna de identidades      | AWS IAM Identity Center con usuarios, grupos, MFA    |
| Identity & Access | Control de accesos granular         | Permission sets, principio de menor privilegio       |
| Cloud             | Operación básica de infraestructura | EC2, VPC, Security Groups, IAM                       |
| Security          | Auditoría y trazabilidad            | CloudTrail habilitado y consultado                   |
| Security          | Monitoreo y detección               | CloudWatch Logs + queries de threat hunting          |
| Security          | Respuesta a incidentes              | 3 incident writeups completos con playbook           |
| Soft              | Gestión de proyecto autónomo        | Plan ejecutado en plazo y entregables documentados   |

---

## 3\. Alcance del proyecto

### Dentro del alcance (in-scope)

- Configuración inicial de cuenta AWS con buenas prácticas de seguridad (MFA, billing alerts, IAM user).
- Diseño y despliegue de VPC con subnets, security groups e IAM roles.
- Configuración de AWS IAM Identity Center con usuarios y grupos simulados.
- Despliegue manual de osTicket sobre EC2 Linux con MariaDB y Apache.
- Configuración operativa de osTicket: departamentos, SLAs, formularios, agentes, email vía SES.
- Simulación de 10-15 tickets variados resueltos con documentación de resolución.
- Habilitación de CloudTrail con logs hacia S3 + CloudWatch.
- Instalación de CloudWatch Agent en endpoints simulados.
- Simulación controlada de tres tipos de incidentes de seguridad.
- Detección de incidentes mediante CloudWatch Logs Insights.
- Configuración de Alarms y notificaciones por SNS.
- Documentación profesional en formato GitHub (README, fases, incident writeups).

### Fuera del alcance (out-of-scope)

- Active Directory tradicional on-premise (se reemplaza por IAM Identity Center, más moderno y alineado con stack startup).
- SIEM enterprise (Splunk, Sentinel, QRadar): se aproxima conceptualmente con CloudWatch Logs Insights.
- Wazuh Manager: requiere más recursos que los disponibles en Free Tier.
- MDM real (Jamf, Intune): fuera del alcance por costos y por no ser objetivo central.
- Automatización completa con IaC: opcional como bonus en Fase 4, no obligatorio.
- Multi-region o alta disponibilidad: irrelevante para el propósito del lab.
- Integración con servicios externos pagos (Datadog, PagerDuty, Slack pagado).
- Penetration testing real: solo simulación de ataques desde host propio dentro de la VPC.

---

## 4\. Arquitectura

### Componentes

| Componente          | Tipo         | Propósito                               | Costo Free Tier        |
| :------------------ | :----------- | :-------------------------------------- | :--------------------- |
| VPC                 | Networking   | Aislamiento de red del lab              | Gratis                 |
| EC2 osTicket        | t2.micro     | Servidor del sistema de helpdesk        | 750 hrs/mes gratis     |
| EC2 Endpoint A      | t2.micro     | Endpoint simulado, target de monitoreo  | (compartido del cupo)  |
| EC2 Endpoint B      | t2.micro     | Endpoint simulado, target de ataques    | (compartido del cupo)  |
| IAM Identity Center | Identity     | Gestión centralizada de usuarios        | Gratis                 |
| CloudTrail          | Audit        | Audit log de toda la actividad AWS      | Primer trail gratis    |
| CloudWatch Logs     | Logging      | Centralización de logs de los endpoints | 5 GB/mes gratis        |
| CloudWatch Alarms   | Alerting     | Alertas sobre eventos sospechosos       | 10 alarms gratis       |
| SNS                 | Notification | Envío de alertas por email              | 1000 emails/mes gratis |
| SES                 | Email        | Email de osTicket para tickets          | 200 emails/día gratis  |
| S3                  | Storage      | Bucket para logs de CloudTrail          | 5 GB gratis            |

---

## 5\. Restricciones técnicas

- **AWS Free Tier estricto**: cualquier servicio fuera de Free Tier requiere aprobación explícita antes de uso.
- **3 instancias EC2 t2.micro concurrentes máximo**: las instancias deben apagarse cuando no se está trabajando activamente.
- **Sin NAT Gateway**: cuesta USD $32/mes. Las EC2 estarán en subnet pública con security groups restrictivos.
- **Sin Elastic IPs huérfanas**: cobran si no están asociadas a una EC2 corriendo.

---

## 6\. Gestión de riesgos

| Riesgo | Probabilidad | Impacto | Mitigación |
| :----- | :----------- | :------ | :--------- |
| Cargo inesperado en AWS | Media | Alto | Billing alerts en $1/$5/$10, revisión diaria de Cost Explorer |
| Olvido de Elastic IP huérfana | Media | Bajo | Checklist al cerrar sesión |
| Falla de osTicket por mala configuración | Media | Medio | Backup de la base de datos antes de cambios mayores |
| Bloqueo de cuenta AWS por actividad sospechosa | Baja | Alto | NO ejecutar herramientas de ataque contra IPs fuera de la propia VPC |
| Pérdida de SSH key pair | Baja | Medio | Backup del .pem en gestor de contraseñas |

---

## 7\. Tear-down post-proyecto

- [x] Terminar todas las EC2
- [x] Liberar Elastic IPs
- [x] Vaciar y eliminar bucket S3 de CloudTrail
- [x] Eliminar CloudWatch Log Groups
- [x] Deshabilitar CloudTrail
- [x] Verificar Cost Explorer al día siguiente
- [x] Repositorio GitHub permanece como portfolio

---

## Control de cambios

| Versión | Fecha     | Cambios         |
| :------ | :-------- | :-------------- |
| 1.0     | Mayo 2026 | Versión inicial |

---

*Documento de referencia para la ejecución del proyecto.*