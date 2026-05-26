# CloudOps Helpdesk \+ Security Lab

**Hoja de ruta del proyecto**

| Campo                    | Valor                                                           |
| :----------------------- | :-------------------------------------------------------------- |
| **Autor**                | Martín Salvo                                                    |
| **Inicio estimado**      | Mayo 2026                                                       |
| **Duración planificada** | 14 días                                                         |
| **Esfuerzo diario**      | 3-4 horas                                                       |
| **Presupuesto**          | USD $0 (AWS Free Tier estricto)                                 |
| **Repositorio**          | github.com/\[usuario\]/cloudops-helpdesk-security-lab *(crear)* |
| **Rol objetivo**         | Jr. IT Support Specialist / SOC Analyst Trainee                 |

---

## 1\. Resumen ejecutivo

Construcción end-to-end de un entorno cloud que simula la infraestructura de IT Support y Security Operations de una pequeña empresa, desplegada íntegramente en AWS Free Tier. El proyecto demuestra capacidades técnicas en gestión de identidades, soporte a usuarios mediante sistema de ticketing, monitoreo de seguridad, y respuesta a incidentes.

El objetivo no es construir una solución de producción, sino producir un **portfolio técnico verificable** que evidencie competencias relevantes para roles de IT Support híbrido con foco en seguridad, alineado al camino profesional hacia SOC Jr.

---

## 2\. Objetivos y motivación

### Objetivo principal

Generar un proyecto público, documentado y demostrable que respalde técnicamente el perfil de "Jr IT Support \+ Security" en entrevistas y postulaciones, cerrando el gap percibido entre teoría (Security+) y ejecución práctica.

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
| Security          | Monitoreo y detección               | CloudWatch Logs \+ queries de threat hunting         |
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
- Habilitación de CloudTrail con logs hacia S3 \+ CloudWatch.  
- Instalación de CloudWatch Agent en endpoints simulados.  
- Simulación controlada de tres tipos de incidentes de seguridad.  
- Detección de incidentes mediante CloudWatch Logs Insights.  
- Configuración de Alarms y notificaciones por SNS.  
- Documentación profesional en formato GitHub (README, fases, incident writeups).  
- Demo video de 2-3 minutos.

### Fuera del alcance (out-of-scope)

- Active Directory tradicional on-premise (se reemplaza por IAM Identity Center, más moderno y alineado con stack startup).  
- SIEM enterprise (Splunk, Sentinel, QRadar): se aproxima conceptualmente con CloudWatch Logs Insights.  
- Wazuh Manager: requiere más recursos que los disponibles en Free Tier.  
- MDM real (Jamf, Intune): fuera del alcance por costos y por no ser objetivo central.  
- Automatización completa con IaC: opcional como bonus en Fase 4, no obligatorio.  
- Multi-region o alta disponibilidad: irrelevante para el propósito del lab.  
- Integración con servicios externos pagos (Datadog, PagerDuty, Slack pagado).  
- Penetration testing real: solo simulación de ataques desde host propio dentro de la VPC.

### Supuestos

- Cuenta AWS recién creada con 12 meses de Free Tier disponibles.  
- Acceso a internet estable durante las 14 horas-día planificadas.  
- Disponibilidad de 3-4 horas diarias durante 14 días consecutivos.  
- Conocimientos previos: Linux básico, redes básicas (TCP/IP, SSH), CompTIA Security+.

---

## 4\. Arquitectura

### Vista general

El lab simula una organización ficticia con tres componentes principales: una capa de identidad centralizada, un sistema de helpdesk operativo, y una capa transversal de seguridad y monitoreo.

![][image1] 

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

### Total de instancias EC2 concurrentes: 3 × t2.micro

Cálculo de horas: 3 instances × 24 hrs × 14 días \= 1.008 horas. Free Tier permite 750 horas/mes. **Acción correctiva**: apagar instancias cuando no se trabaje activamente. Si trabajamos efectivamente 4 hrs/día durante 14 días con instancias prendidas solo en ese horario: 3 × 4 × 14 \= 168 horas. Muy dentro del límite.

---

## 5\. Plan de ejecución

### Fase 1 — Fundamentos AWS (Días 1-3)

**Objetivo de fase**: Establecer cuenta AWS con buenas prácticas y desplegar la infraestructura de red base.

#### Día 1 — Seguridad de cuenta y monitoreo de costos

- [ ] Habilitar MFA en cuenta root (Authenticator app)  
- [ ] Crear usuario IAM administrativo personal con MFA  
- [ ] Cerrar sesión de root y trabajar siempre con el usuario IAM  
- [ ] Configurar AWS Budgets con alertas en USD $1, $5, $10  
- [ ] Habilitar AWS Cost Explorer  
- [ ] Suscribirse a las alertas de Free Tier usage  
- [ ] Documentar configuración inicial en `01-aws-fundamentals.md`

#### Día 2 — Networking base

- [ ] Crear VPC custom (CIDR 10.0.0.0/16)  
- [ ] Crear 2 subnets públicas en distintas AZ  
- [ ] Crear Internet Gateway y asociar a la VPC  
- [ ] Configurar Route Table pública  
- [ ] Crear Security Groups iniciales (SSH, HTTP, HTTPS)  
- [ ] Lanzar primera EC2 t2.micro de prueba (Ubuntu)  
- [ ] Conectarse por SSH usando key pair  
- [ ] Detener instancia de prueba para no consumir horas  
- [ ] Documentar diseño de red

#### Día 3 — Identity layer

- [ ] Habilitar AWS IAM Identity Center en la región  
- [ ] Crear directorio interno  
- [ ] Crear 3 usuarios ficticios con perfiles realistas:  
      - Ana García — Marketing Manager  
      - Carlos López — Finance Analyst  
      - Diana Pérez — Sales Lead  
- [ ] Crear 3 grupos: Marketing, Finance, Sales  
- [ ] Asignar usuarios a grupos  
- [ ] Crear permission sets diferenciados por grupo  
- [ ] Habilitar MFA forzado para todos los usuarios  
- [ ] Probar login con uno de los usuarios ficticios  
- [ ] Documentar arquitectura de identidades

**Entregable Fase 1**: Documento `01-aws-fundamentals.md` con configuración inicial, diagrama de VPC, y arquitectura de IAM Identity Center.

---

### Fase 2 — Helpdesk operativo (Días 4-7)

**Objetivo de fase**: Sistema de tickets funcionando con flujos operativos definidos.

#### Día 4 — Deploy de osTicket

- [ ] Lanzar EC2 t2.micro dedicada para osTicket  
- [ ] Asignar Elastic IP (atención: solo gratis si está asociada)  
- [ ] Instalar stack LAMP: Apache, PHP, MariaDB  
- [ ] Crear base de datos para osTicket  
- [ ] Descargar e instalar osTicket última versión  
- [ ] Completar wizard de instalación  
- [ ] Verificar acceso al panel admin desde navegador  
- [ ] Configurar SSL básico (Let's Encrypt opcional)

#### Día 5 — Configuración operativa

- [ ] Crear departamentos: IT Support, Security, HR  
- [ ] Configurar SLAs:  
      - Critical: 2 horas  
      - High: 4 horas  
      - Standard: 24 horas  
      - Low: 72 horas  
- [ ] Crear formularios custom:  
      - Access Request (sistema, justificación, manager)  
      - Hardware Issue (device, síntomas)  
      - General Inquiry  
- [ ] Crear 2 agentes: vos como admin, "Junior Agent" como técnico  
- [ ] Definir help topics y priorities

#### Día 6 — Email integration

- [ ] Verificar dominio o usar email personal verificado en SES  
- [ ] Configurar SES en sandbox mode  
- [ ] Generar SMTP credentials  
- [ ] Configurar osTicket para enviar/recibir vía SES  
- [ ] Test: enviar email a la dirección configurada y verificar que crea ticket  
- [ ] Configurar auto-responder y plantillas de respuesta

#### Día 7 — Operación simulada

- [ ] Generar 10-15 tickets diversos desde "usuarios" ficticios:  
      - "No puedo acceder a Salesforce" (Access Request)  
      - "Mi laptop hace ruido extraño" (Hardware Issue)  
      - "Necesito VPN para viaje" (Access Request)  
      - "Bloqueé mi cuenta" (Account Lockout)  
      - "Solicito Notion Pro" (Software Request)  
      - Etc.  
- [ ] Atender cada ticket end-to-end:  
      - Triagear por SLA  
      - Asignar a categoría  
      - Documentar resolución  
      - Cerrar con resumen  
- [ ] Tomar screenshots de tickets resueltos  
- [ ] Documentar 5 procedimientos operativos estándar

**Entregable Fase 2**: Documento `02-helpdesk-deployment.md` con arquitectura, screenshots, ejemplos de tickets, y procedimientos operativos.

---

### Fase 3 — Security operations (Días 8-11)

**Objetivo de fase**: Capa de monitoreo activa y tres incident writeups completos.

#### Día 8 — Audit logging

- [ ] Crear bucket S3 dedicado para logs  
- [ ] Habilitar CloudTrail con trail multi-region  
- [ ] Configurar log delivery a S3 y a CloudWatch Logs  
- [ ] Configurar log file validation  
- [ ] Explorar Event History en la consola  
- [ ] Identificar eventos importantes: ConsoleLogin, CreateUser, AttachPolicy  
- [ ] Documentar política de retención

#### Día 9 — Endpoint monitoring

- [ ] Lanzar Endpoint A y Endpoint B (EC2 t2.micro Ubuntu)  
- [ ] Lanzar "Attacker host" (EC2 t2.micro Kali Linux o Ubuntu con herramientas)  
- [ ] Crear IAM role con permisos para CloudWatch Agent  
- [ ] Instalar y configurar CloudWatch Agent en Endpoints A y B  
- [ ] Configurar envío de:  
      - `/var/log/auth.log` (intentos SSH)  
      - `/var/log/syslog` (sistema)  
      - Métricas básicas (CPU, memoria)  
- [ ] Crear log groups con retention de 7 días  
- [ ] Verificar que los logs lleguen a CloudWatch

#### Día 10 — Simulación de incidentes

- [ ] **Incidente 1: SSH Brute Force**  
      - Desde Attacker host, usar hydra o medusa contra Endpoint B  
      - Generar 100+ intentos fallidos en pocos minutos  
      - Verificar que los logs lleguen a CloudWatch  
- [ ] **Incidente 2: Privilege Escalation sospechosa**  
      - Login legítimo en Endpoint A  
      - Ejecutar comandos sudo no esperados (crear usuario nuevo)  
      - Modificar archivos sensibles (/etc/passwd, /etc/sudoers)  
- [ ] **Incidente 3: Acceso desde IP no esperada**  
      - Configurar VPN o usar IP externa para acceder  
      - Loguear actividad inusual  
- [ ] Documentar timestamps exactos de cada incidente

#### Día 11 — Detección y respuesta

- [ ] Construir queries CloudWatch Logs Insights:  
      - Detectar SSH brute force (filter por failed authentication, count \> 10 por IP)  
      - Detectar comandos sudo sospechosos  
      - Detectar logins desde nuevas IPs  
- [ ] Crear Alarms basadas en log metric filters  
- [ ] Configurar SNS topic con tu email  
- [ ] Verificar que las Alarms disparen notificaciones  
- [ ] Escribir 3 incident writeups en formato profesional:  
      - Executive summary  
      - Timeline  
      - Detection queries usadas  
      - Indicators of Compromise (IoCs)  
      - Acciones de remediación  
      - Lecciones aprendidas

**Entregable Fase 3**: Documento `03-security-operations.md` con 3 incident response writeups completos.

---

### Fase 4 — Portfolio y publicación (Días 12-14)

**Objetivo de fase**: Material publicable y presentable en entrevistas.

#### Día 12 — README principal

- [ ] Crear estructura del repositorio:  
        
      /  
        
      ├── README.md  
        
      ├── docs/  
        
      │   ├── 01-aws-fundamentals.md  
        
      │   ├── 02-helpdesk-deployment.md  
        
      │   ├── 03-security-operations.md  
        
      │   └── 04-incidents/  
        
      │       ├── incident-01-ssh-brute-force.md  
        
      │       ├── incident-02-privilege-escalation.md  
        
      │       └── incident-03-unusual-access.md  
        
      ├── diagrams/  
        
      │   └── architecture.png  
        
      ├── screenshots/  
        
      └── scripts/  
        
- [ ] Escribir README principal con:  
      - Project overview  
      - Architecture diagram  
      - Tech stack  
      - Phases summary  
      - Key learnings  
      - Cost analysis (USD $0)  
      - Links a cada documento  
- [ ] Push inicial al repo público

#### Día 13 — Infrastructure as Code (opcional pero recomendado)

- [ ] Aprender CloudFormation o Terraform básico  
- [ ] Convertir al menos la VPC y Security Groups a IaC  
- [ ] Incluir templates en `/scripts/`  
- [ ] Documentar cómo redeplegar el lab desde cero

#### Día 14 — Demo y publicación

- [ ] Grabar demo video con Loom (2-3 minutos):  
      - Mostrar arquitectura  
      - Demo de osTicket con un ticket abierto  
      - Demo de CloudWatch Logs Insights detectando un incidente  
      - Cierre con summary  
- [ ] Publicar repo público en GitHub  
- [ ] Escribir post de LinkedIn con:  
      - Resumen del proyecto  
      - 3 cosas aprendidas  
      - Link al repo y video  
- [ ] Actualizar CV con sección "Proyectos"  
- [ ] Actualizar perfil de LinkedIn con el proyecto destacado

**Entregable Fase 4**: Repo público, demo video, post de LinkedIn, CV actualizado.

---

## 6\. Restricciones y consideraciones

### Restricciones técnicas

- **AWS Free Tier estricto**: cualquier servicio fuera de Free Tier requiere aprobación explícita antes de uso.  
- **3 instancias EC2 t2.micro concurrentes máximo**: para mantenerse dentro de las 750 horas mensuales gratuitas, las instancias deben apagarse cuando no se está trabajando activamente.  
- **Sin NAT Gateway**: cuesta USD $32/mes. Las EC2 estarán en subnet pública con security groups restrictivos.  
- **Sin Elastic IPs huérfanas**: cobran si no están asociadas a una EC2 corriendo.

### Restricciones de tiempo

- 14 días calendario, 3-4 horas diarias.  
- La Fase 4 (portfolio) NO debe sacrificarse. Si las Fases 1-3 se extienden, se reduce el alcance de las simulaciones (menos tickets, menos incidentes) pero la documentación se mantiene completa.

### Restricciones de scope

- **No** agregar features mid-proyecto: ideas nuevas se anotan en backlog para post-proyecto.  
- **No** profundizar más allá de lo planificado en cada fase: la profundidad técnica se gana iterando, no en el primer lab.

---

## 7\. Entregables finales

Al cierre del proyecto, debe existir:

1. **Repositorio GitHub público** con toda la documentación, scripts y diagramas.  
2. **README profesional** que sirva como landing del proyecto.  
3. **3 incident response writeups** con formato profesional.  
4. **Demo video de 2-3 minutos** alojado en YouTube o Loom público.  
5. **Post de LinkedIn** publicado con visibilidad al proyecto.  
6. **CV actualizado** con sección Proyectos referenciando el repo.  
7. **Diagrama de arquitectura** en formato exportable (PNG/SVG).  
8. **(Opcional)** Templates de IaC (CloudFormation o Terraform).

---

## 8\. Criterios de éxito

El proyecto se considera exitoso si al día 14:

| Criterio      | Métrica                                                                      |
| :------------ | :--------------------------------------------------------------------------- |
| Funcionalidad | osTicket operativo, IAM Identity Center configurado, 3 incidentes detectados |
| Documentación | 4 documentos de fase \+ 3 incident writeups \+ README                        |
| Costo         | Cargo total en AWS \= USD $0                                                 |
| Visibilidad   | Repo público \+ demo video \+ post de LinkedIn publicados                    |
| Conocimiento  | Capacidad de explicar cada componente sin notas en una entrevista            |
| Reusabilidad  | Posibilidad de redespliegue desde cero usando la documentación               |

---

## 9\. Gestión de riesgos

### Riesgos identificados y mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
| :--------------------------------------------- | :---- | :---- | :----------------------------------------------------------------------------------------------------- |
| Cargo inesperado en AWS                        | Media | Alto  | Billing alerts en $1/$5/$10, revisión diaria de Cost Explorer, apagado de EC2 al finalizar cada sesión |
| Olvido de Elastic IP huérfana                  | Media | Bajo  | Checklist al cerrar sesión: verificar EIPs asociadas o liberar                                         |
| Falla de osTicket por mala configuración       | Media | Medio | Backup de la base de datos antes de cambios mayores, snapshot de EBS antes de Fase 3                   |
| Bloqueo de cuenta AWS por actividad sospechosa | Baja  | Alto  | NO ejecutar herramientas de ataque contra IPs fuera de la propia VPC, no realizar port scans externos  |
| Pérdida de SSH key pair                        | Baja  | Medio | Backup del .pem en gestor de contraseñas                                                               |
| Cuenta IAM Identity Center con problemas       | Baja  | Medio | Documentar credenciales root en gestor de contraseñas, MFA con backup codes                            |
| Sobre-extensión del scope                      | Alta  | Medio | Decisión firme: features nuevas → backlog, no se ejecutan en este proyecto                             |
| Falla en el ritmo diario                       | Media | Medio | Si se pierde un día, se mantiene fecha de cierre y se reduce scope de  Fase 3 (no de Fase 4\)          |

---

## 10\. Tear-down post-proyecto

Después de publicar y presentar el lab, los recursos AWS pueden eliminarse para liberar cupos Free Tier futuros y eliminar cualquier riesgo residual:

- [ ] Terminar todas las EC2 (no solo apagar)  
- [ ] Liberar Elastic IPs  
- [ ] Eliminar snapshots manuales innecesarios  
- [ ] Vaciar y eliminar bucket S3 de CloudTrail  
- [ ] Eliminar CloudWatch Log Groups  
- [ ] Deshabilitar CloudTrail  
- [ ] Verificar Cost Explorer al día siguiente  
- [ ] El repositorio GitHub permanece como portfolio

---

## 11\. Próximos pasos post-proyecto

Una vez finalizado el lab, los siguientes proyectos extienden la curva de aprendizaje:

1. **AWS Cloud Practitioner Certification** (1-2 semanas adicionales).  
2. **Lab v2 con IaC completo**: re-implementar todo con Terraform desde cero.  
3. **Lab v3 con SIEM real**: agregar Wazuh Manager en una EC2 más grande (fuera de Free Tier, costo \~USD $15-25 si se mantiene 1 mes).  
4. **Lab v4 cross-account**: simular environment con security account separada.  
5. **Contribución open source**: aportar al proyecto osTicket o a documentación de AWS Security.

---

## 12\. Referencias y recursos

### Documentación oficial

- AWS Free Tier: [https://aws.amazon.com/free](https://aws.amazon.com/free)  
- AWS IAM Identity Center: [https://docs.aws.amazon.com/singlesignon](https://docs.aws.amazon.com/singlesignon)  
- AWS CloudTrail User Guide: [https://docs.aws.amazon.com/awscloudtrail](https://docs.aws.amazon.com/awscloudtrail)  
- AWS CloudWatch Logs Insights: [https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)  
- osTicket Documentation: [https://docs.osticket.com](https://docs.osticket.com)

### Comunidades y aprendizaje

- r/AWSCertifications (Reddit)  
- r/cybersecurity (Reddit)  
- ISC2 Community  
- TryHackMe (paths gratuitos de Cloud y SOC)

### Herramientas

- draw.io / excalidraw.com para diagramas  
- Loom para grabación de demo  
- GitHub para hosting del repo

---

## Control de cambios

| Versión | Fecha     | Cambios         |
| :------ | :-------- | :-------------- |
| 1.0     | Mayo 2026 | Versión inicial |

---

*Documento de referencia para la ejecución del proyecto. Revisar al inicio de cada fase y actualizar el control de cambios si se modifica el alcance.*  


[image1]: <data:image/png;base64,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>