# 🚀 SÍNTESIS COMPLETA - ERP GRUPO FIXUM

## 🏢 ESTRUCTURA EMPRESARIAL

### **Consorcio y Empresas:**
- **Grupo Fixum** (entidad padre/consorcio)
- **7 Empresas Operativas:**
  - Kreis
  - Gosan
  - Altaguarda
  - DTC Coatzacoalcos
  - Maha
  - Milenio
  - Acierta
  - Jaguar

### **Características:**
- **Sector:** Industrial con proyectos específicos
- **Identidad visual:** Cada empresa mantiene sus colores corporativos
- **Operación:** Multi-proyecto con recursos compartidos específicos

---

## 🏗️ ARQUITECTURA TÉCNICA

### **Stack Tecnológico Confirmado:**
- **Backend:** Laravel 12 con arquitectura hexagonal
- **Frontend:** Vue 3 + Nuxt 3
- **Base de datos:** PostgreSQL (multi-tenant con schemas)
- **Servidor:** VPS Hostinger + CyberPanel + LiteSpeed
- **Desarrollo:** Directo en servidor con subdominio

### **Arquitectura Hexagonal (DDD):**
- **Dominio:** Lógica de negocio pura (multi-tenant, permisos)
- **Aplicación:** Casos de uso y servicios de negocio
- **Infraestructura:** Laravel, PostgreSQL, APIs externas
- **Adaptadores:** Controllers, Repositories, Events
- **Puertos:** Interfaces que definen contratos

### **Infraestructura Actual:**
- ✅ **AlmaLinux 9.6** (Enterprise Linux)
- ✅ **PostgreSQL 13.20** instalado y funcionando
- ✅ **CyberPanel** para administración
- ✅ **LiteSpeed** como servidor web

---

## 🔐 SEGURIDAD Y MULTI-TENANCY

### **Separación de Datos:**
- **Estricta por empresa:** Cada empresa = schema PostgreSQL independiente
- **Recursos compartidos específicos:**
  - **Activos TI:** Asignables entre proyectos/empresas
  - **Almacén:** Sistema de traspasos entre empresas
  - **Schema compartido:** Para recursos transversales

### **Estructura PostgreSQL:**
```sql
-- Schemas por empresa
CREATE SCHEMA kreis;
CREATE SCHEMA gosan;
CREATE SCHEMA altaguarda;
CREATE SCHEMA dtc_coatzacoalcos;
CREATE SCHEMA maha;
CREATE SCHEMA milenio;
CREATE SCHEMA acierta;
CREATE SCHEMA jaguar;

-- Schema compartido
CREATE SCHEMA shared; -- Activos TI y traspasos
```

### **Sistema de Autenticación:**
- **Login multi-empresa:** Usuario + Contraseña + Selector de empresa
- **Sesión única:** Un usuario = un dispositivo activo (expulsión automática)
- **Switch empresarial:** Dropdown en header sin re-login
- **Tecnología:** JWT + Laravel Sanctum
- **Contexto persistente:** Recordar última empresa seleccionada

### **Permisos Granulares:**
- **5 Niveles de control:**
  1. **Empresa** → Acceso a qué empresas
  2. **Módulo** → Qué secciones del ERP
  3. **Acción** → Visualizar, Crear, Editar, Autorizar, Eliminar
  4. **Proyecto** → Acceso específico por proyecto
  5. **Recurso** → Activos TI, almacén compartido

- **Perfiles personalizables:** Admin puede crear roles específicos
- **Contexto empresarial:** Diferentes permisos por empresa
- **Herencia de permisos:** Sistema jerárquico configurable

---

## 🎨 INTERFAZ DE USUARIO

### **Diseño y UX:**
- **UI Empresarial:** Profesional pero moderna y animada
- **Temas adaptativos:** Oscuro/Claro con switch automático
- **Responsive design:** Mobile-first, multiplataforma
- **PWA capabilities:** Funcionalidades de app nativa
- **Colores corporativos:** Preservar identidad visual de cada empresa

### **Header Inteligente:**
- **Dropdown empresarial:** Cambio de contexto sin reload
- **Indicador visual:** Logo/colores de empresa activa
- **Breadcrumbs contextuales:** "Grupo Fixum > Kreis > Inventario"
- **Búsqueda rápida:** Para empresas con muchas opciones
- **Favoritos:** Empresas más usadas prioritarias

### **Experiencia de Usuario:**
- **Transiciones suaves:** Animaciones profesionales
- **Feedback visual:** Cambios de estado claros
- **Navegación intuitiva:** Estructura lógica por módulos
- **Accesibilidad:** Cumplimiento de estándares web

---

## ⚙️ SISTEMAS INTEGRADOS

### **Aplicación de Almacén Existente:**
- **Migración gradual:** Backend como submódulo al nuevo ERP
- **Sistema OCs:** Evolución de manual a automatizado
- **Preservación:** Mantener funcionalidad durante transición
- **Integración:** Con nuevo sistema de permisos y multi-tenancy

### **Módulo TI (Desarrollo Nuevo):**
- **Recreación completa:** Partir de base Excel existente de AppSheet
- **Funcionalidades principales:**
  - Control de activos TI
  - Gestión de stock tecnológico
  - Sistema de reparaciones
  - Administración de licencias
- **Gestión compartida:** Activos asignables entre proyectos/empresas
- **Workflow:** Solicitud → Aprobación → Asignación → Seguimiento

---

## 🔔 NOTIFICACIONES Y COMUNICACIÓN

### **Sistema de Notificaciones Inteligente:**
- **Tiempo real:** WebSockets (Laravel Broadcasting + Pusher/Socket.io)
- **Multi-canal:** In-app, email, SMS (configurable por usuario)
- **Workflow automático:** Notificaciones por roles y departamentos
- **Ejemplo de flujo:** Requisición autorizada → Notifica a creador + Compras
- **Persistencia:** Historial de notificaciones leídas/no leídas
- **Configuración:** Cada usuario define qué notificaciones recibir

### **Casos de Uso:**
- Aprobaciones de requisiciones
- Cambios de estado en proyectos
- Asignación de activos TI
- Vencimientos de licencias
- Alertas de inventario bajo
- Notificaciones de sistema

---

## 📊 AUDITORÍA Y MONITOREO

### **Sistema de Auditoría Completo:**
- **Logs detallados:** Cada acción CRUD con contexto empresarial
- **Trazabilidad completa:** Quién, cuándo, qué, desde dónde, por qué
- **Soft deletes obligatorio:** No eliminación real, solo bloqueo/desactivación
- **Tracking de cambios:** Before/After en campos críticos
- **Dashboard de auditoría:** Filtros avanzados para debugging y compliance
- **Alertas de seguridad:** Detección de accesos sospechosos

### **Monitoreo del Servidor Integrado:**
- **Recursos en tiempo real:** CPU, RAM, disco dentro del ERP
- **Usuarios activos:** Dashboard de sesiones en línea
- **Performance metrics:** Tiempo de respuesta, queries lentas
- **Alertas automáticas:** Notificaciones por sobrecarga de recursos

### **Gestión de Usuarios:**
- **Control granular:** Bloqueo temporal/permanente (no eliminación)
- **Sesión única:** Expulsión automática de sesiones duplicadas
- **Historial de accesos:** Por empresa, módulo y proyecto
- **Roles dinámicos:** Asignación flexible de permisos

---

## 📄 EXPORTACIÓN Y REPORTES

### **Generación de Documentos:**
- **PDFs dinámicos:** Laravel-dompdf con plantillas personalizables por empresa
- **Excel empresarial:** Laravel-Excel con formatos corporativos
- **Reportes contextuales:** Basados en permisos y empresa activa
- **Templates:** Logos, colores y formatos por empresa

### **Tipos de Reportes:**
- Inventarios por empresa/proyecto
- Estados financieros consolidados
- Reportes de activos TI
- Auditorías de accesos
- Métricas de performance

---

## 📚 DOCUMENTACIÓN Y DESARROLLO

### **Documentación Automática:**
- **Archivo MD de referencia:** Auto-generado con:
  - Todas las tablas de PostgreSQL
  - Columnas, tipos de datos e índices
  - Relaciones y foreign keys
  - Seeders y datos de prueba
- **Actualización automática:** En cada migración de Laravel
- **Referencia técnica:** Para desarrollo y mantenimiento futuro

### **Metodología de Desarrollo:**
- **Gradual e iterativo:** Frontend y Backend en paralelo
- **Módulo por módulo:** Implementación escalonada por prioridades
- **Testing automático:** En cada feature desarrollada
- **Integración continua:** Deploy automático en servidor
- **Versionado:** Control de cambios y rollbacks

---

## 🔧 CONFIGURACIÓN ACTUAL DEL SERVIDOR

### **Estado de la Infraestructura:**
- **Sistema:** AlmaLinux 9.6 (Sage Margay)
- **Panel:** CyberPanel 2.4.1 → 2.4.2 (pendiente actualización)
- **Web Server:** LiteSpeed
- **Base de datos:** PostgreSQL 13.20 ✅ FUNCIONANDO
- **Recursos:** 7.5GB RAM, 99GB Disco (17% usado)
- **Accesos:**
  - CyberPanel: https://31.97.141.107:8090
  - phpMyAdmin: https://31.97.141.107:8090/dataBases/phpMyAdmin
  - Rainloop: https://31.97.141.107:8090/rainloop

### **Próximos Pasos Técnicos:**
1. **Configurar PostgreSQL** (usuarios, bases de datos, schemas)
2. **Instalar Laravel 12** con estructura hexagonal
3. **Setup Vue 3 + Nuxt 3** con SSR
4. **Desarrollar autenticación** multi-empresa
5. **Crear dashboard base** con switch de empresas

---

## 🎯 OBJETIVOS DEL PROYECTO

### **Metas Principales:**
- **Modernización:** Reemplazar ERP obsoleto con solución contemporánea
- **Consolidación:** Unificar gestión de 7 empresas bajo una plataforma
- **Eficiencia:** Automatizar procesos manuales actuales
- **Seguridad:** Implementar controles empresariales robustos
- **Escalabilidad:** Arquitectura preparada para crecimiento futuro

### **Impacto Esperado:**
- Reducción de tiempo en procesos administrativos
- Mayor control y trazabilidad de operaciones
- Mejor colaboración entre empresas del grupo
- Informes consolidados para toma de decisiones
- Modernización de la imagen tecnológica del grupo

---

*Documento generado: 30 de julio, 2025*  
*Estado: PostgreSQL instalado ✅ - Listo para fase de desarrollo*