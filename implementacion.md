# 📋 PLAN DE DESARROLLO POR FASES - ERP GRUPO FIXUM

## 🎯 OBJETIVO DE LAS FASES INICIALES
Crear la **infraestructura base sólida** del ERP antes de desarrollar módulos específicos. Esto incluye: configuración de servidores, bases de datos, autenticación, permisos granulares, y estructura fundamental.

---

## 🏗️ FASE 1: CONFIGURACIÓN DE INFRAESTRUCTURA BASE
**Duración estimada:** 2-3 días  
**Estado:** PostgreSQL ✅ Instalado

### **1.1 Configuración PostgreSQL Multi-Tenant**
```bash
# Crear usuario administrador del ERP
sudo -u postgres createuser --interactive --pwprompt fixum_admin

# Crear base de datos principal
sudo -u postgres createdb fixum_erp -O fixum_admin

# Configurar acceso remoto y seguridad
sudo nano /var/lib/pgsql/data/postgresql.conf
sudo nano /var/lib/pgsql/data/pg_hba.conf
```

### **1.2 Schemas por Empresa**
```sql
-- Conectar como fixum_admin
CREATE SCHEMA kreis;
CREATE SCHEMA gosan;
CREATE SCHEMA altaguarda;
CREATE SCHEMA dtc_coatzacoalcos;
CREATE SCHEMA maha;
CREATE SCHEMA milenio;
CREATE SCHEMA acierta;
CREATE SCHEMA jaguar;
CREATE SCHEMA shared;  -- Para activos TI y traspasos
CREATE SCHEMA system;  -- Para usuarios, permisos, configuraciones
```

### **1.3 Configuración del Subdominio**
- Crear subdominio en CyberPanel: `erp.tudominio.com`
- Configurar SSL automático
- Configurar estructura de directorios
- Configurar PHP 8.2+ y extensiones necesarias

---

## 🚀 FASE 2: INSTALACIÓN Y CONFIGURACIÓN LARAVEL
**Duración estimada:** 2-3 días

### **2.1 Instalación Laravel 12**
```bash
# En el directorio del subdominio
composer create-project laravel/laravel erp-fixum
cd erp-fixum

# Configurar .env para PostgreSQL
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=fixum_erp
DB_USERNAME=fixum_admin
DB_PASSWORD=tu_password
```

### **2.2 Estructura Hexagonal**
```
app/
├── Domain/              # Lógica de negocio pura
│   ├── User/
│   ├── Company/
│   ├── Permission/
│   └── SharedAssets/
├── Application/         # Casos de uso
│   ├── Services/
│   ├── Commands/
│   └── Queries/
├── Infrastructure/      # Implementaciones
│   ├── Repositories/
│   ├── Services/
│   └── Events/
└── Interfaces/         # Adaptadores
    ├── Http/
    ├── Console/
    └── Broadcasting/
```

### **2.3 Configuración Multi-Tenant**
```php
// app/Services/TenantService.php
class TenantService {
    public function switchSchema($company_schema) {
        DB::statement("SET search_path TO {$company_schema}, public");
    }
}

// Middleware para cambio automático de schema
class TenantMiddleware {
    // Lógica de cambio de contexto empresarial
}
```

---

## 🔐 FASE 3: SISTEMA DE AUTENTICACIÓN Y PERMISOS BASE
**Duración estimada:** 3-4 días

### **3.1 Migraciones Fundamentales (Schema: system)**
```php
// Tabla de empresas
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->string('code')->unique(); // kreis, gosan, etc.
    $table->string('name');
    $table->string('schema_name');
    $table->json('brand_colors'); // Colores corporativos
    $table->string('logo_path')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});

// Tabla de usuarios (global)
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('username')->unique();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->string('first_name');
    $table->string('last_name');
    $table->boolean('is_active')->default(true);
    $table->timestamp('last_login_at')->nullable();
    $table->string('last_login_ip')->nullable();
    $table->rememberToken();
    $table->timestamps();
    $table->softDeletes();
});

// Relación usuarios-empresas
Schema::create('user_companies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->foreignId('company_id')->constrained();
    $table->boolean('is_default')->default(false);
    $table->timestamps();
});
```

### **3.2 Sistema de Permisos Granular**
```php
// Módulos del sistema
Schema::create('modules', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->string('icon')->nullable();
    $table->integer('sort_order')->default(0);
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});

// Acciones posibles
Schema::create('actions', function (Blueprint $table) {
    $table->id();
    $table->string('name'); // view, create, edit, authorize, delete
    $table->string('slug')->unique();
    $table->timestamps();
});

// Roles/Perfiles
Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug');
    $table->text('description')->nullable();
    $table->foreignId('company_id')->constrained();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});

// Permisos granulares
Schema::create('role_permissions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('role_id')->constrained();
    $table->foreignId('module_id')->constrained();
    $table->foreignId('action_id')->constrained();
    $table->boolean('granted')->default(false);
    $table->timestamps();
});

// Asignación de roles a usuarios por empresa
Schema::create('user_roles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->foreignId('role_id')->constrained();
    $table->foreignId('company_id')->constrained();
    $table->timestamps();
});
```

### **3.3 Sistema de Sesiones Únicas**
```php
// Control de sesiones activas
Schema::create('user_sessions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->string('session_id')->unique();
    $table->string('ip_address');
    $table->text('user_agent');
    $table->timestamp('last_activity');
    $table->timestamps();
});
```

---

## 🎨 FASE 4: INSTALACIÓN Y CONFIGURACIÓN FRONTEND
**Duración estimada:** 2-3 días

### **4.1 Instalación Vue 3 + Nuxt 3**
```bash
# En directorio separado para el frontend
npx nuxi@latest init erp-frontend
cd erp-frontend

# Instalar dependencias principales
npm install @pinia/nuxt
npm install @nuxtjs/tailwindcss
npm install @headlessui/vue @heroicons/vue
npm install @vueuse/nuxt
npm install axios
```

### **4.2 Configuración Nuxt**
```javascript
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },
  modules: [
    '@pinia/nuxt',
    '@nuxtjs/tailwindcss',
    '@vueuse/nuxt'
  ],
  css: ['~/assets/css/main.css'],
  runtimeConfig: {
    public: {
      apiBase: 'http://localhost:8000/api'
    }
  },
  ssr: true
})
```

### **4.3 Estructura de Directorios Frontend**
```
pages/
├── login.vue
├── dashboard.vue
├── settings/
└── profile.vue

components/
├── Auth/
├── Layout/
├── Forms/
└── UI/

stores/
├── auth.js
├── companies.js
└── permissions.js

middleware/
├── auth.js
├── guest.js
└── tenant.js
```

---

## 🔑 FASE 5: DESARROLLO DEL SISTEMA DE LOGIN
**Duración estimada:** 3-4 días

### **5.1 Backend - API de Autenticación**
```php
// app/Http/Controllers/Auth/AuthController.php
class AuthController extends Controller {
    public function login(LoginRequest $request) {
        // 1. Validar credenciales
        // 2. Obtener empresas del usuario
        // 3. Crear token JWT con contexto
        // 4. Registrar sesión activa
        // 5. Expulsar sesiones anteriores
    }
    
    public function switchCompany(SwitchCompanyRequest $request) {
        // Cambiar contexto empresarial sin re-login
    }
    
    public function logout(Request $request) {
        // Limpiar sesión y tokens
    }
}

// Rutas API
Route::prefix('auth')->group(function () {
    Route::post('login', [AuthController::class, 'login']);
    Route::post('logout', [AuthController::class, 'logout']);
    Route::post('switch-company', [AuthController::class, 'switchCompany']);
    Route::get('me', [AuthController::class, 'me']);
    Route::get('companies', [AuthController::class, 'getUserCompanies']);
});
```

### **5.2 Frontend - Páginas de Login**
```vue
<!-- pages/login.vue -->
<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-50 dark:bg-gray-900">
    <div class="max-w-md w-full space-y-8">
      <div>
        <h2 class="text-center text-3xl font-extrabold text-gray-900 dark:text-white">
          ERP Grupo Fixum
        </h2>
      </div>
      <form @submit.prevent="handleLogin" class="mt-8 space-y-6">
        <!-- Username -->
        <div>
          <input v-model="form.username" type="text" required 
                 class="appearance-none rounded-md relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
                 placeholder="Usuario">
        </div>
        
        <!-- Password -->
        <div>
          <input v-model="form.password" type="password" required 
                 class="appearance-none rounded-md relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
                 placeholder="Contraseña">
        </div>
        
        <!-- Company Selection (shown after credentials validation) -->
        <div v-if="companies.length > 0">
          <select v-model="form.company_id" required
                  class="appearance-none rounded-md relative block w-full px-3 py-2 border border-gray-300 text-gray-900 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500">
            <option value="">Seleccionar empresa</option>
            <option v-for="company in companies" :key="company.id" :value="company.id">
              {{ company.name }}
            </option>
          </select>
        </div>
        
        <button type="submit" :disabled="loading"
                class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50">
          {{ loading ? 'Iniciando sesión...' : 'Iniciar Sesión' }}
        </button>
      </form>
    </div>
  </div>
</template>
```

### **5.3 Store de Autenticación (Pinia)**
```javascript
// stores/auth.js
export const useAuthStore = defineStore('auth', {
  state: () => ({
    user: null,
    token: null,
    currentCompany: null,
    userCompanies: [],
    permissions: []
  }),
  
  actions: {
    async login(credentials) {
      // Lógica de login
    },
    
    async switchCompany(companyId) {
      // Cambiar empresa sin re-login
    },
    
    async logout() {
      // Limpiar estado y redireccionar
    },
    
    hasPermission(module, action) {
      // Verificar permisos granulares
    }
  }
})
```

---

## 📊 FASE 6: DASHBOARD BASE Y NAVEGACIÓN
**Duración estimada:** 2-3 días

### **6.1 Layout Principal**
```vue
<!-- layouts/default.vue -->
<template>
  <div class="min-h-screen bg-gray-100 dark:bg-gray-900">
    <!-- Header con dropdown de empresas -->
    <header class="bg-white dark:bg-gray-800 shadow">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between h-16">
          <!-- Logo y navegación -->
          <div class="flex">
            <div class="flex-shrink-0 flex items-center">
              <h1 class="text-xl font-bold text-gray-900 dark:text-white">
                ERP Grupo Fixum
              </h1>
            </div>
          </div>
          
          <!-- Dropdown de empresas y usuario -->
          <div class="flex items-center space-x-4">
            <!-- Company Switcher -->
            <CompanySwitcher />
            
            <!-- Theme Toggle -->
            <ThemeToggle />
            
            <!-- User Menu -->
            <UserDropdown />
          </div>
        </div>
      </div>
    </header>
    
    <!-- Main Content -->
    <main>
      <slot />
    </main>
  </div>
</template>
```

### **6.2 Dashboard Principal**
```vue
<!-- pages/dashboard.vue -->
<template>
  <div class="py-6">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <!-- Bienvenida contextual -->
      <div class="mb-8">
        <h1 class="text-2xl font-semibold text-gray-900 dark:text-white">
          Bienvenido a {{ currentCompany?.name }}
        </h1>
        <p class="mt-1 text-sm text-gray-600 dark:text-gray-400">
          Panel de control - {{ new Date().toLocaleDateString() }}
        </p>
      </div>
      
      <!-- Widgets básicos -->
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
        <StatsCard title="Usuarios Activos" :value="stats.activeUsers" />
        <StatsCard title="Empresas" :value="stats.companies" />
        <StatsCard title="Módulos" :value="stats.modules" />
        <StatsCard title="Sesiones" :value="stats.sessions" />
      </div>
      
      <!-- Información del sistema -->
      <div class="bg-white dark:bg-gray-800 overflow-hidden shadow rounded-lg">
        <div class="px-4 py-5 sm:p-6">
          <h3 class="text-lg leading-6 font-medium text-gray-900 dark:text-white">
            Estado del Sistema
          </h3>
          <div class="mt-2 max-w-xl text-sm text-gray-500 dark:text-gray-400">
            <p>El sistema está funcionando correctamente.</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

---

## 🔧 FASE 7: CONFIGURACIONES DEL SISTEMA
**Duración estimada:** 2-3 días

### **7.1 Gestión de Empresas**
- CRUD de empresas
- Configuración de colores corporativos
- Logos por empresa
- Activación/desactivación

### **7.2 Gestión de Usuarios**
- CRUD de usuarios
- Asignación de empresas
- Gestión de roles por empresa
- Bloqueo/desbloqueo de usuarios

### **7.3 Gestión de Permisos**
- Crear/editar roles
- Asignación granular de permisos
- Templates de roles comunes
- Previsualización de permisos

### **7.4 Configuraciones Generales**
- Configuración de sistema
- Parámetros por empresa
- Configuración de notificaciones
- Logs de auditoría

---

## 📋 FASE 8: SISTEMA DE AUDITORÍA Y LOGS
**Duración estimada:** 2 días

### **8.1 Middleware de Auditoría**
```php
class AuditMiddleware {
    public function handle($request, Closure $next) {
        $response = $next($request);
        
        // Registrar toda actividad
        AuditLog::create([
            'user_id' => auth()->id(),
            'company_id' => session('current_company_id'),
            'action' => $request->method(),
            'resource' => $request->path(),
            'ip_address' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'request_data' => $request->all(),
            'response_status' => $response->status(),
        ]);
        
        return $response;
    }
}
```

### **8.2 Dashboard de Auditoría**
- Logs de acceso por usuario
- Filtros por empresa, fecha, acción
- Exportación de logs
- Alertas de seguridad

---

## ✅ FASE 9: TESTING Y OPTIMIZACIÓN
**Duración estimada:** 2-3 días

### **9.1 Testing Automatizado**
- Unit tests para servicios críticos
- Feature tests para autenticación
- Tests de permisos
- Tests de multi-tenancy

### **9.2 Optimización**
- Optimización de queries PostgreSQL
- Caché de permisos
- Optimización del frontend
- Configuración de producción

---

## 🚀 FASE 10: DEPLOYMENT Y DOCUMENTACIÓN
**Duración estimada:** 1-2 días

### **10.1 Configuración de Producción**
- Variables de entorno
- Configuración de SSL
- Backup automático
- Monitoreo básico

### **10.2 Documentación**
- Manual de usuario básico
- Documentación técnica
- Guía de instalación
- Archivo MD auto-generado de BD

---

## 📊 RESUMEN DE ENTREGAS POR FASE

| Fase | Entregable | Duración |
|------|------------|----------|
| 1 | PostgreSQL configurado con schemas | 2-3 días |
| 2 | Laravel 12 con arquitectura hexagonal | 2-3 días |
| 3 | Sistema de permisos granular | 3-4 días |
| 4 | Frontend Vue 3 + Nuxt 3 | 2-3 días |
| 5 | Login multi-empresa funcional | 3-4 días |
| 6 | Dashboard base con navegación | 2-3 días |
| 7 | Configuraciones del sistema | 2-3 días |
| 8 | Auditoría y logs | 2 días |
| 9 | Testing y optimización | 2-3 días |
| 10 | Deployment y documentación | 1-2 días |

**DURACIÓN TOTAL ESTIMADA: 21-30 días**

---

## 🎯 CRITERIOS DE ACEPTACIÓN

Cada fase debe cumplir:
- ✅ **Funcionalidad completa** según especificación
- ✅ **Testing básico** funcionando
- ✅ **Documentación actualizada**
- ✅ **Sin errores críticos**
- ✅ **Revisión de seguridad** aprobada

Una vez completadas estas 10 fases, tendremos la **base sólida** para comenzar a desarrollar módulos específicos del ERP (inventario, compras, proyectos, etc.).