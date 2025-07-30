# 🚀 FASE 2 DETALLADA: INSTALACIÓN Y CONFIGURACIÓN LARAVEL

## 🎯 OBJETIVOS DE LA FASE 2
- Instalar Laravel 12 en el subdominio configurado
- Implementar arquitectura hexagonal desde el inicio
- Configurar conexión multi-tenant con PostgreSQL
- Establecer estructura de directorios modular
- Configurar servicios base y middleware
- Preparar sistema de migraciones por schema

**Duración estimada:** 2-3 días  
**Prerequisito:** Fase 1 completada ✅

---

## 📋 PASO 2.1: INSTALACIÓN DE LARAVEL 12
**Tiempo estimado:** 45-60 minutos

### **2.1.1 Preparar directorio de trabajo**
```bash
# Navegar al directorio del subdominio
cd /home/erp.tudominio.com/

# Verificar que tenemos los permisos correctos
ls -la
whoami

# Crear backup del directorio actual (por seguridad)
sudo cp -r public_html public_html_backup
```

### **2.1.2 Instalar Laravel 12**
```bash
# Instalar Laravel en directorio temporal
composer create-project laravel/laravel erp-backend "12.*"

# Verificar que se instaló correctamente
cd erp-backend
php artisan --version

# Debería mostrar: Laravel Framework 12.x.x
```

### **2.1.3 Configurar estructura de directorios**
```bash
# Mover contenido de Laravel al directorio principal
cd /home/erp.tudominio.com/

# Mover archivos de Laravel
sudo mv erp-backend/* ./
sudo mv erp-backend/.[^.]* ./ 2>/dev/null || true

# Limpiar directorio temporal
sudo rm -rf erp-backend

# Ajustar permisos
sudo chown -R cyberpanel:cyberpanel /home/erp.tudominio.com/
sudo chmod -R 755 /home/erp.tudominio.com/
sudo chmod -R 775 storage bootstrap/cache
```

### **2.1.4 Configurar virtual host para Laravel**
```bash
# El public_html debe apuntar a la carpeta public de Laravel
sudo rm -rf public_html
sudo ln -s /home/erp.tudominio.com/public public_html

# Verificar que el enlace simbólico funciona
ls -la public_html
```

### **2.1.5 Probar instalación básica**
```bash
# Crear archivo de prueba
echo "<?php echo 'Laravel 12 instalado correctamente - ' . date('Y-m-d H:i:s'); ?>" > public/test.php

# Probar en navegador: https://erp.tudominio.com/test.php
# También probar: https://erp.tudominio.com (debería mostrar página de Laravel)
```

**✅ Checkpoint:** Laravel 12 instalado y funcionando

---

## ⚙️ PASO 2.2: CONFIGURACIÓN DE ENTORNO
**Tiempo estimado:** 30-45 minutos

### **2.2.1 Configurar archivo .env**
```bash
# Hacer copia de seguridad del .env
cp .env .env.backup

# Editar configuración
nano .env
```

```env
# Configuración de aplicación
APP_NAME="ERP Grupo Fixum"
APP_ENV=development
APP_KEY=
APP_DEBUG=true
APP_URL=https://erp.tudominio.com

# Configuración de base de datos PostgreSQL
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=fixum_erp
DB_USERNAME=fixum_admin
DB_PASSWORD="FixumERP2025!@#"

# Configuración de caché
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=database
SESSION_LIFETIME=120

# Configuración de correo (configurar después)
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="noreply@fixum.com"
MAIL_FROM_NAME="${APP_NAME}"

# Timezone
APP_TIMEZONE='America/Mexico_City'

# Configuración multi-tenant
DEFAULT_TENANT_SCHEMA=system
TENANT_SCHEMAS="kreis,gosan,altaguarda,dtc_coatzacoalcos,maha,milenio,acierta,jaguar,shared"
```

### **2.2.2 Generar clave de aplicación**
```bash
php artisan key:generate

# Verificar que se generó
grep APP_KEY .env
```

### **2.2.3 Probar conexión a base de datos**
```bash
# Test de conexión
php artisan tinker

# Dentro de tinker, ejecutar:
DB::connection()->getPdo();
exit
```

**✅ Checkpoint:** Laravel conectado a PostgreSQL

---

## 🏗️ PASO 2.3: IMPLEMENTAR ARQUITECTURA HEXAGONAL
**Tiempo estimado:** 60-90 minutos

### **2.3.1 Crear estructura de directorios hexagonal**
```bash
# Crear estructura de dominios
mkdir -p app/Domain/User/{Entities,Repositories,Services}
mkdir -p app/Domain/Company/{Entities,Repositories,Services}
mkdir -p app/Domain/Permission/{Entities,Repositories,Services}
mkdir -p app/Domain/SharedAssets/{Entities,Repositories,Services}
mkdir -p app/Domain/Audit/{Entities,Repositories,Services}

# Crear capa de aplicación
mkdir -p app/Application/Commands/{User,Company,Permission}
mkdir -p app/Application/Queries/{User,Company,Permission}
mkdir -p app/Application/Services/{Auth,Tenant,Audit}

# Crear capa de infraestructura
mkdir -p app/Infrastructure/Repositories/{User,Company,Permission}
mkdir -p app/Infrastructure/Services/{Database,Cache,Mail}
mkdir -p app/Infrastructure/Events
mkdir -p app/Infrastructure/Jobs

# Crear adaptadores/interfaces
mkdir -p app/Interfaces/Http/Controllers/{Api,Web}
mkdir -p app/Interfaces/Http/Requests/{Auth,User,Company}
mkdir -p app/Interfaces/Http/Resources
mkdir -p app/Interfaces/Console/Commands
```

### **2.3.2 Configurar PSR-4 autoloading**
```bash
# Editar composer.json
nano composer.json
```

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/",
            "App\\Domain\\": "app/Domain/",
            "App\\Application\\": "app/Application/",
            "App\\Infrastructure\\": "app/Infrastructure/",
            "App\\Interfaces\\": "app/Interfaces/"
        }
    }
}
```

```bash
# Regenerar autoload
composer dump-autoload
```

### **2.3.3 Crear interfaces base (Ports)**
```bash
# Crear archivo de repositorio base
nano app/Domain/Contracts/RepositoryInterface.php
```

```php
<?php

namespace App\Domain\Contracts;

interface RepositoryInterface
{
    public function find($id);
    public function findOrFail($id);
    public function create(array $data);
    public function update($id, array $data);
    public function delete($id);
    public function all();
    public function paginate($perPage = 15);
}
```

```bash
# Crear interfaz de tenant
nano app/Domain/Contracts/TenantInterface.php
```

```php
<?php

namespace App\Domain\Contracts;

interface TenantInterface
{
    public function switchTenant(string $schema): void;
    public function getCurrentTenant(): ?string;
    public function getTenantSchemas(): array;
    public function isTenantValid(string $schema): bool;
}
```

**✅ Checkpoint:** Estructura hexagonal creada

---

## 🔗 PASO 2.4: CONFIGURAR MULTI-TENANCY
**Tiempo estimado:** 90-120 minutos

### **2.4.1 Crear servicio de Tenant**
```bash
nano app/Application/Services/Tenant/TenantService.php
```

```php
<?php

namespace App\Application\Services\Tenant;

use App\Domain\Contracts\TenantInterface;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Config;

class TenantService implements TenantInterface
{
    private ?string $currentTenant = null;
    private array $tenantSchemas;

    public function __construct()
    {
        $this->tenantSchemas = explode(',', config('app.tenant_schemas', ''));
        $this->currentTenant = config('app.default_tenant_schema', 'system');
    }

    public function switchTenant(string $schema): void
    {
        if (!$this->isTenantValid($schema)) {
            throw new \InvalidArgumentException("Schema '{$schema}' no válido");
        }

        // Cambiar search_path en PostgreSQL
        DB::statement("SET search_path TO {$schema}, public");
        
        $this->currentTenant = $schema;
        
        // Guardar en sesión si está disponible
        if (session()->isStarted()) {
            session(['current_tenant' => $schema]);
        }
    }

    public function getCurrentTenant(): ?string
    {
        // Priorizar sesión sobre propiedad de clase
        if (session()->isStarted() && session('current_tenant')) {
            $this->currentTenant = session('current_tenant');
        }
        
        return $this->currentTenant;
    }

    public function getTenantSchemas(): array
    {
        return $this->tenantSchemas;
    }

    public function isTenantValid(string $schema): bool
    {
        return in_array($schema, $this->tenantSchemas) || $schema === 'system';
    }

    public function getAllValidSchemas(): array
    {
        return array_merge(['system'], $this->tenantSchemas);
    }
}
```

### **2.4.2 Crear middleware de Tenant**
```bash
php artisan make:middleware TenantMiddleware
```

```bash
# Editar el middleware creado
nano app/Http/Middleware/TenantMiddleware.php
```

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Application\Services\Tenant\TenantService;
use Symfony\Component\HttpFoundation\Response;

class TenantMiddleware
{
    private TenantService $tenantService;

    public function __construct(TenantService $tenantService)
    {
        $this->tenantService = $tenantService;
    }

    public function handle(Request $request, Closure $next): Response
    {
        // Obtener tenant de la sesión, header o parámetro
        $tenant = $this->resolveTenant($request);
        
        if ($tenant && $this->tenantService->isTenantValid($tenant)) {
            $this->tenantService->switchTenant($tenant);
        } else {
            // Por defecto usar system schema
            $this->tenantService->switchTenant('system');
        }

        return $next($request);
    }

    private function resolveTenant(Request $request): ?string
    {
        // 1. Buscar en header X-Tenant
        if ($request->hasHeader('X-Tenant')) {
            return $request->header('X-Tenant');
        }

        // 2. Buscar en sesión
        if (session('current_tenant')) {
            return session('current_tenant');
        }

        // 3. Buscar en query parameter
        if ($request->has('tenant')) {
            return $request->get('tenant');
        }

        return null;
    }
}
```

### **2.4.3 Registrar servicios en Service Provider**
```bash
php artisan make:provider TenantServiceProvider
```

```bash
nano app/Providers/TenantServiceProvider.php
```

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Domain\Contracts\TenantInterface;
use App\Application\Services\Tenant\TenantService;

class TenantServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(TenantInterface::class, TenantService::class);
        $this->app->singleton(TenantService::class);
    }

    public function boot(): void
    {
        // Configurar tenant por defecto al inicio
        $tenantService = $this->app->make(TenantService::class);
        $tenantService->switchTenant(config('app.default_tenant_schema', 'system'));
    }
}
```

### **2.4.4 Registrar provider y middleware**
```bash
# Editar config/app.php
nano config/app.php
```

Agregar en el array `providers`:
```php
App\Providers\TenantServiceProvider::class,
```

```bash
# Registrar middleware
nano app/Http/Kernel.php
```

Agregar en `$middleware` (global):
```php
\App\Http\Middleware\TenantMiddleware::class,
```

**✅ Checkpoint:** Multi-tenancy configurado

---

## 🗃️ PASO 2.5: CONFIGURAR MIGRACIONES MULTI-TENANT
**Tiempo estimado:** 60-90 minutos

### **2.5.1 Crear comando para migraciones por schema**
```bash
php artisan make:command MigrateTenants
```

```bash
nano app/Console/Commands/MigrateTenants.php
```

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use App\Application\Services\Tenant\TenantService;

class MigrateTenants extends Command
{
    protected $signature = 'tenant:migrate {--schema= : Specific schema to migrate} {--fresh : Drop all tables and re-run migrations}';
    protected $description = 'Run migrations for all tenant schemas';

    private TenantService $tenantService;

    public function __construct(TenantService $tenantService)
    {
        parent::__construct();
        $this->tenantService = $tenantService;
    }

    public function handle()
    {
        $targetSchema = $this->option('schema');
        $fresh = $this->option('fresh');

        if ($targetSchema) {
            // Migrar solo un schema específico
            $this->migrateSchema($targetSchema, $fresh);
        } else {
            // Migrar todos los schemas
            $schemas = $this->tenantService->getAllValidSchemas();
            
            foreach ($schemas as $schema) {
                $this->migrateSchema($schema, $fresh);
            }
        }

        $this->info('✅ Migraciones completadas');
    }

    private function migrateSchema(string $schema, bool $fresh = false): void
    {
        $this->info("🔄 Migrando schema: {$schema}");

        try {
            // Cambiar al schema
            $this->tenantService->switchTenant($schema);

            // Ejecutar migraciones
            if ($fresh) {
                $this->call('migrate:fresh', ['--force' => true]);
            } else {
                $this->call('migrate', ['--force' => true]);
            }

            $this->info("✅ Schema {$schema} migrado correctamente");

        } catch (\Exception $e) {
            $this->error("❌ Error migrando {$schema}: " . $e->getMessage());
        }
    }
}
```

### **2.5.2 Crear migración base para todas las tablas**
```bash
php artisan make:migration create_base_tables
```

```bash
nano database/migrations/[timestamp]_create_base_tables.php
```

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Esta migración se ejecutará en cada schema
        
        // Tabla de configuraciones por tenant
        Schema::create('tenant_settings', function (Blueprint $table) {
            $table->id();
            $table->string('key')->unique();
            $table->text('value')->nullable();
            $table->string('type')->default('string'); // string, json, boolean, integer
            $table->text('description')->nullable();
            $table->timestamps();
        });

        // Tabla de logs de actividad por tenant
        Schema::create('activity_logs', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id')->nullable();
            $table->string('action');
            $table->string('resource');
            $table->json('old_values')->nullable();
            $table->json('new_values')->nullable();
            $table->string('ip_address')->nullable();
            $table->text('user_agent')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('activity_logs');
        Schema::dropIfExists('tenant_settings');
    }
};
```

### **2.5.3 Crear seeder base**
```bash
php artisan make:seeder TenantBaseSeeder
```

```bash
nano database/seeders/TenantBaseSeeder.php
```

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;

class TenantBaseSeeder extends Seeder
{
    public function run(): void
    {
        // Configuraciones básicas por tenant
        $settings = [
            [
                'key' => 'company_name',
                'value' => $this->getCompanyName(),
                'type' => 'string',
                'description' => 'Nombre de la empresa'
            ],
            [
                'key' => 'timezone',
                'value' => 'America/Mexico_City',
                'type' => 'string',
                'description' => 'Zona horaria de la empresa'
            ],
            [
                'key' => 'date_format',
                'value' => 'd/m/Y',
                'type' => 'string',
                'description' => 'Formato de fecha'
            ]
        ];

        foreach ($settings as $setting) {
            DB::table('tenant_settings')->updateOrInsert(
                ['key' => $setting['key']],
                array_merge($setting, [
                    'created_at' => now(),
                    'updated_at' => now()
                ])
            );
        }
    }

    private function getCompanyName(): string
    {
        $schema = DB::select('SELECT current_schema()')[0]->current_schema;
        
        $companies = [
            'kreis' => 'Kreis',
            'gosan' => 'Gosan',
            'altaguarda' => 'Altaguarda',
            'dtc_coatzacoalcos' => 'DTC Coatzacoalcos',
            'maha' => 'Maha',
            'milenio' => 'Milenio',
            'acierta' => 'Acierta',
            'jaguar' => 'Jaguar',
            'shared' => 'Recursos Compartidos',
            'system' => 'Sistema'
        ];

        return $companies[$schema] ?? 'Grupo Fixum';
    }
}
```

**✅ Checkpoint:** Sistema de migraciones multi-tenant listo

---

## 🧪 PASO 2.6: TESTING Y VERIFICACIÓN
**Tiempo estimado:** 45-60 minutos

### **2.6.1 Ejecutar migraciones de prueba**
```bash
# Migrar todas las tablas base
php artisan tenant:migrate

# Verificar que se crearon las tablas en cada schema
php artisan tinker
```

```php
// En tinker
use App\Application\Services\Tenant\TenantService;

$tenant = app(TenantService::class);

// Probar cambio de schema
$tenant->switchTenant('kreis');
DB::select('SELECT current_schema()');

$tenant->switchTenant('system');
DB::select('SELECT current_schema()');

// Verificar tablas
DB::select("SELECT tablename FROM pg_tables WHERE schemaname = 'system'");

exit
```

### **2.6.2 Crear controlador de prueba**
```bash
php artisan make:controller TestController
```

```bash
nano app/Http/Controllers/TestController.php
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Application\Services\Tenant\TenantService;
use Illuminate\Support\Facades\DB;

class TestController extends Controller
{
    private TenantService $tenantService;

    public function __construct(TenantService $tenantService)
    {
        $this->tenantService = $tenantService;
    }

    public function testTenant(Request $request)
    {
        $currentSchema = DB::select('SELECT current_schema()')[0]->current_schema;
        
        return response()->json([
            'success' => true,
            'current_schema' => $currentSchema,
            'available_schemas' => $this->tenantService->getAllValidSchemas(),
            'session_tenant' => session('current_tenant'),
            'service_tenant' => $this->tenantService->getCurrentTenant(),
            'timestamp' => now()
        ]);
    }

    public function switchTenant(Request $request, string $schema)
    {
        try {
            $this->tenantService->switchTenant($schema);
            
            $newSchema = DB::select('SELECT current_schema()')[0]->current_schema;
            
            return response()->json([
                'success' => true,
                'message' => "Cambiado al schema: {$schema}",
                'current_schema' => $newSchema,
                'session_tenant' => session('current_tenant')
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage()
            ], 400);
        }
    }
}
```

### **2.6.3 Crear rutas de prueba**
```bash
nano routes/web.php
```

Agregar al final:
```php
// Rutas de prueba para desarrollo
if (app()->environment(['local', 'development'])) {
    Route::get('/test/tenant', [App\Http\Controllers\TestController::class, 'testTenant']);
    Route::post('/test/tenant/{schema}', [App\Http\Controllers\TestController::class, 'switchTenant']);
}
```

### **2.6.4 Probar en navegador**
```bash
# Probar la ruta de test
# https://erp.tudominio.com/test/tenant

# También probar cambio de tenant con POST
# https://erp.tudominio.com/test/tenant/kreis
```

**✅ Checkpoint:** Todo funcionando correctamente

---

## 📝 PASO 2.7: CONFIGURACIÓN ADICIONAL
**Tiempo estimado:** 30-45 minutos

### **2.7.1 Configurar .gitignore para desarrollo**
```bash
nano .gitignore
```

```gitignore
# Archivos de Laravel
/node_modules
/public/hot
/public/storage
/storage/*.key
/vendor
.env
.env.backup
.phpunit.result.cache
docker-compose.override.yml
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
/.idea
/.vscode

# Archivos específicos del proyecto
/backups
/public/test.php
*.log
```

### **2.7.2 Crear archivo de configuración personalizada**
```bash
nano config/tenant.php
```

```php
<?php

return [
    /*
    |--------------------------------------------------------------------------
    | Multi-Tenant Configuration
    |--------------------------------------------------------------------------
    */

    'default_schema' => env('DEFAULT_TENANT_SCHEMA', 'system'),
    
    'tenant_schemas' => explode(',', env('TENANT_SCHEMAS', '')),
    
    'companies' => [
        'kreis' => [
            'name' => 'Kreis',
            'schema' => 'kreis',
            'colors' => [
                'primary' => '#1f2937',
                'secondary' => '#f59e0b'
            ]
        ],
        'gosan' => [
            'name' => 'Gosan',
            'schema' => 'gosan',
            'colors' => [
                'primary' => '#065f46',
                'secondary' => '#10b981'
            ]
        ],
        'altaguarda' => [
            'name' => 'Altaguarda',
            'schema' => 'altaguarda',
            'colors' => [
                'primary' => '#7c2d12',
                'secondary' => '#ea580c'
            ]
        ],
        'dtc_coatzacoalcos' => [
            'name' => 'DTC Coatzacoalcos',
            'schema' => 'dtc_coatzacoalcos',
            'colors' => [
                'primary' => '#1e40af',
                'secondary' => '#3b82f6'
            ]
        ],
        'maha' => [
            'name' => 'Maha',
            'schema' => 'maha',
            'colors' => [
                'primary' => '#7c1d6f',
                'secondary' => '#d946ef'
            ]
        ],
        'milenio' => [
            'name' => 'Milenio',
            'schema' => 'milenio',
            'colors' => [
                'primary' => '#991b1b',
                'secondary' => '#ef4444'
            ]
        ],
        'acierta' => [
            'name' => 'Acierta',
            'schema' => 'acierta',
            'colors' => [
                'primary' => '#0c4a6e',
                'secondary' => '#0ea5e9'
            ]
        ],
        'jaguar' => [
            'name' => 'Jaguar',
            'schema' => 'jaguar',
            'colors' => [
                'primary' => '#365314',
                'secondary' => '#65a30d'
            ]
        ]
    ],

    'shared_resources' => [
        'schema' => 'shared',
        'tables' => [
            'assets',
            'transfers'
        ]
    ]
];
```

### **2.7.3 Optimizar para desarrollo**
```bash
# Configurar caché de configuración
php artisan config:cache

# Limpiar caché si es necesario
php artisan config:clear

# Optimizar autoload
composer dump-autoload -o

# Verificar rutas
php artisan route:list
```

**✅ Checkpoint:** Configuración completa

---

## 🔍 PASO 2.8: VERIFICACIÓN FINAL Y DOCUMENTACIÓN
**Tiempo estimado:** 30 minutos

### **2.8.1 Test final completo**
```bash
# Crear script de verificación
nano test_phase2.php
```

```php
<?php
require_once 'vendor/autoload.php';

$app = require_once 'bootstrap/app.php';
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

// Test 1: Laravel funcionando
echo "🧪 Testing Laravel installation...\n";
echo "Laravel Version: " . app()->version() . "\n";

// Test 2: PostgreSQL connection
echo "\n🧪 Testing PostgreSQL connection...\n";
try {
    $pdo = DB::connection()->getPdo();
    echo "✅ PostgreSQL connected\n";
} catch (Exception $e) {
    echo "❌ PostgreSQL error: " . $e->getMessage() . "\n";
}

// Test 3: Tenant service
echo "\n🧪 Testing Tenant Service...\n";
$tenantService = app(\App\Application\Services\Tenant\TenantService::class);
echo "Current tenant: " . $tenantService->getCurrentTenant() . "\n";
echo "Available schemas: " . implode(', ', $tenantService->getAllValidSchemas()) . "\n";

// Test 4: Schema switching
echo "\n🧪 Testing schema switching...\n";
$schemas = ['system', 'kreis', 'gosan'];
foreach ($schemas as $schema) {
    try {
        $tenantService->switchTenant($schema);
        $current = DB::select('SELECT current_schema()')[0]->current_schema;
        echo "✅ Switched to {$schema}: {$current}\n";
    } catch (Exception $e) {
        echo "❌ Error switching to {$schema}: " . $e->getMessage() . "\n";
    }
}

echo "\n🎉 Phase 2 verification completed!\n";
```

```bash
php test_phase2.php
```

### **2.8.2 Generar documentación automática**
```bash
# Crear comando para documentar la BD
php artisan make:command GenerateDbDocs
```

```bash
nano app/Console/Commands/GenerateDbDocs.php
```

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class GenerateDbDocs extends Command
{
    protected $signature = 'docs:database';
    protected $description = 'Generate database documentation';

    public function handle()
    {
        $this->info('📚 Generating database documentation...');
        
        $schemas = DB::select("SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT IN ('information_schema', 'pg_catalog', 'pg_toast')");
        
        $docs = "# 📊 ERP GRUPO FIXUM - DATABASE DOCUMENTATION\n\n";
        $docs .= "Generated: " . now() . "\n\n";
        
        foreach ($schemas as $schema) {
            $schemaName = $schema->schema_name;
            $docs .= "## Schema: {$schemaName}\n\n";
            
            // Get tables in schema
            $tables = DB::select("SELECT tablename FROM pg_tables WHERE schemaname = ?", [$schemaName]);
            
            foreach ($tables as $table) {
                $tableName = $table->tablename;
                $docs .= "### Table: {$tableName}\n";
                
                // Get columns
                $columns = DB::select("
                    SELECT column_name, data_type, is_nullable, column_default 
                    FROM information_schema.columns 
                    WHERE table_schema = ? AND table_name = ?
                    ORDER BY ordinal_position
                ", [$schemaName, $tableName]);
                
                $docs .= "| Column | Type | Nullable | Default |\n";
                $docs .= "|--------|------|----------|----------|\n";
                
                foreach ($columns as $column) {
                    $docs .= "| {$column->column_name} | {$column->data_type} | {$column->is_nullable} | {$column->column_default} |\n";
                }
                
                $docs .= "\n";
            }
            
            $docs .= "\n";
        }
        
        // Save documentation
        file_put_contents(base_path('DATABASE_DOCS.md'), $docs);
        
        $this->info('✅ Database documentation generated: DATABASE_DOCS.md');
    }
}
```

```bash
# Generar documentación inicial
php artisan docs:database
```

### **2.8.3 Crear archivo de configuración de entorno para producción**
```bash
# Crear template para producción
cp .env .env.production.template

nano .env.production.template
```

```env
# CONFIGURACIÓN PARA PRODUCCIÓN
APP_NAME="ERP Grupo Fixum"
APP_ENV=production
APP_KEY=base64:GENERAR_NUEVA_CLAVE
APP_DEBUG=false
APP_URL=https://erp.tudominio.com

LOG_CHANNEL=daily
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=fixum_erp
DB_USERNAME=fixum_admin
DB_PASSWORD=CAMBIAR_PASSWORD_PRODUCCION

BROADCAST_DRIVER=log
CACHE_DRIVER=redis
FILESYSTEM_DISK=local
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=tu-servidor-smtp
MAIL_PORT=587
MAIL_USERNAME=noreply@tudominio.com
MAIL_PASSWORD=tu-password-smtp
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="noreply@tudominio.com"
MAIL_FROM_NAME="${APP_NAME}"

APP_TIMEZONE='America/Mexico_City'

DEFAULT_TENANT_SCHEMA=system
TENANT_SCHEMAS="kreis,gosan,altaguarda,dtc_coatzacoalcos,maha,milenio,acierta,jaguar,shared"
```

---

## 📋 CHECKLIST FINAL FASE 2

### **✅ Instalación Laravel:**
- [ ] Laravel 12 instalado correctamente
- [ ] Estructura de directorios configurada
- [ ] Virtual host apuntando a /public
- [ ] .env configurado con PostgreSQL
- [ ] Clave de aplicación generada

### **✅ Arquitectura Hexagonal:**
- [ ] Estructura de directorios creada
- [ ] PSR-4 autoloading configurado
- [ ] Interfaces base implementadas
- [ ] Separación de capas establecida

### **✅ Multi-Tenancy:**
- [ ] TenantService implementado
- [ ] TenantMiddleware funcionando
- [ ] Cambio de schemas dinámico
- [ ] Configuración por tenant

### **✅ Sistema de Migraciones:**
- [ ] Comando tenant:migrate creado
- [ ] Migraciones base ejecutadas
- [ ] Seeders por tenant funcionando
- [ ] Tablas creadas en todos los schemas

### **✅ Testing y Verificación:**
- [ ] Controlador de prueba funcionando
- [ ] Rutas de test accesibles
- [ ] Switch de tenants operativo
- [ ] Documentación de BD generada

---

## 🎯 ENTREGABLES DE LA FASE 2

1. **Laravel 12** completamente instalado y configurado
2. **Arquitectura hexagonal** implementada y funcional
3. **Sistema multi-tenant** con PostgreSQL schemas
4. **Middleware de tenant** funcionando automáticamente
5. **Comando de migraciones** por schema
6. **Rutas y controladores de prueba** operativos
7. **Documentación técnica** auto-generada
8. **Configuración de producción** preparada

---

## 🚀 PRÓXIMO PASO: FASE 3

Una vez completada la Fase 2, estaremos listos para:
- Crear el sistema de autenticación multi-empresa
- Implementar las migraciones de usuarios y permisos
- Desarrollar los modelos de dominio
- Configurar Laravel Sanctum para JWT
- Crear los controladores de autenticación

**La base técnica estará 100% sólida para construir todo el sistema de permisos y autenticación.**

### **¿Verificación antes de continuar?**
Antes de pasar a la Fase 3, es recomendable ejecutar:

```bash
# Test completo de la Fase 2
php test_phase2.php

# Verificar rutas
php artisan route:list

# Verificar servicios
php artisan tinker
# $tenant = app(\App\Application\Services\Tenant\TenantService::class);
# $tenant->switchTenant('kreis');
# DB::select('SELECT current_schema()');
```

**¿Todo funcionando correctamente? ¿Continuamos con la Fase 3?** 🎯