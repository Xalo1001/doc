# 🔐 FASE 3 DETALLADA: SISTEMA DE AUTENTICACIÓN Y PERMISOS BASE

## 🎯 OBJETIVOS DE LA FASE 3
- Implementar sistema de autenticación JWT con Laravel Sanctum
- Crear estructura granular de permisos (5 niveles)
- Desarrollar sistema de sesión única por usuario
- Configurar middleware de autenticación multi-empresa
- Establecer sistema de auditoría de accesos
- Crear APIs base para manejo de usuarios y permisos

**Duración estimada:** 3-4 días  
**Prerequisito:** Fase 2 completada ✅

---

## 📊 PASO 3.1: DISEÑO DE BASE DE DATOS - SCHEMA SYSTEM
**Tiempo estimado:** 90-120 minutos

### **3.1.1 Migraciones Fundamentales**

**Tabla de Empresas:**
```php
// 2024_01_01_000001_create_companies_table.php
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->string('code', 20)->unique(); // kreis, gosan, etc.
    $table->string('name');
    $table->string('full_name')->nullable();
    $table->string('schema_name')->unique();
    $table->json('brand_colors'); // {"primary": "#1f2937", "secondary": "#f59e0b"}
    $table->string('logo_path')->nullable();
    $table->boolean('is_active')->default(true);
    $table->text('description')->nullable();
    $table->timestamps();
    $table->softDeletes();
});
```

**Tabla de Usuarios (Global):**
```php
// 2024_01_01_000002_create_users_table.php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('username', 50)->unique();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->string('first_name');
    $table->string('last_name');
    $table->string('phone')->nullable();
    $table->boolean('is_active')->default(true);
    $table->boolean('force_password_change')->default(false);
    $table->timestamp('last_login_at')->nullable();
    $table->string('last_login_ip')->nullable();
    $table->string('avatar_path')->nullable();
    $table->rememberToken();
    $table->timestamps();
    $table->softDeletes();

    $table->index(['username', 'is_active']);
    $table->index(['email', 'is_active']);
});
```

**Relación Usuarios-Empresas:**
```php
// 2024_01_01_000003_create_user_companies_table.php
Schema::create('user_companies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('company_id')->constrained()->cascadeOnDelete();
    $table->boolean('is_default')->default(false);
    $table->json('permissions_override')->nullable(); // Permisos específicos
    $table->timestamps();

    $table->unique(['user_id', 'company_id']);
});
```

### **3.1.2 Sistema de Permisos Granular**

**Módulos del Sistema:**
```php
// 2024_01_01_000004_create_modules_table.php
Schema::create('modules', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->string('icon')->nullable(); // Heroicon name
    $table->string('route_prefix')->nullable();
    $table->integer('sort_order')->default(0);
    $table->boolean('is_active')->default(true);
    $table->foreignId('parent_id')->nullable()->constrained('modules');
    $table->timestamps();
});
```

**Acciones Disponibles:**
```php
// 2024_01_01_000005_create_actions_table.php
Schema::create('actions', function (Blueprint $table) {
    $table->id();
    $table->string('name'); // Ver, Crear, Editar, Autorizar, Eliminar
    $table->string('slug')->unique(); // view, create, edit, authorize, delete
    $table->string('description')->nullable();
    $table->timestamps();
});
```

**Roles/Perfiles por Empresa:**
```php
// 2024_01_01_000006_create_roles_table.php
Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug');
    $table->text('description')->nullable();
    $table->foreignId('company_id')->constrained()->cascadeOnDelete();
    $table->boolean('is_system_role')->default(false); // Admin, Super User
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();

    $table->unique(['slug', 'company_id']);
});
```

**Permisos Granulares:**
```php
// 2024_01_01_000007_create_role_permissions_table.php
Schema::create('role_permissions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('role_id')->constrained()->cascadeOnDelete();
    $table->foreignId('module_id')->constrained()->cascadeOnDelete();
    $table->foreignId('action_id')->constrained()->cascadeOnDelete();
    $table->boolean('granted')->default(false);
    $table->json('conditions')->nullable(); // Condiciones específicas
    $table->timestamps();

    $table->unique(['role_id', 'module_id', 'action_id']);
});
```

**Asignación de Roles:**
```php
// 2024_01_01_000008_create_user_roles_table.php
Schema::create('user_roles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('role_id')->constrained()->cascadeOnDelete();
    $table->foreignId('company_id')->constrained()->cascadeOnDelete();
    $table->timestamp('assigned_at')->default(now());
    $table->foreignId('assigned_by')->nullable()->constrained('users');
    $table->timestamps();

    $table->unique(['user_id', 'role_id', 'company_id']);
});
```

### **3.1.3 Control de Sesiones**

**Sesiones Activas:**
```php
// 2024_01_01_000009_create_user_sessions_table.php
Schema::create('user_sessions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('session_token')->unique(); // JWT token hash
    $table->string('device_name')->nullable();
    $table->string('ip_address');
    $table->text('user_agent');
    $table->timestamp('last_activity');
    $table->boolean('is_active')->default(true);
    $table->timestamps();

    $table->index(['user_id', 'is_active']);
    $table->index('last_activity');
});
```

---

## 🔑 PASO 3.2: IMPLEMENTAR LARAVEL SANCTUM
**Tiempo estimado:** 60-90 minutos

### **3.2.1 Instalación y Configuración**
```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
```

### **3.2.2 Configuración Personalizada**
```php
// config/sanctum.php - Modificaciones clave
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
    '%s%s',
    'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
    Str::startsWith(app('url')->to('/'), 'https://') ? ','.parse_url(app('url')->to('/'), PHP_URL_HOST) : ''
))),

'expiration' => env('SANCTUM_TOKEN_EXPIRATION', 60 * 24), // 24 horas

'token_prefix' => env('SANCTUM_TOKEN_PREFIX', ''),

// Middleware personalizado
'middleware' => [
    'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
    'encrypt_cookies' => App\Http\Middleware\EncryptCookies::class,
    'validate_csrf_token' => App\Http\Middleware\VerifyCsrfToken::class,
    'tenant' => App\Http\Middleware\TenantMiddleware::class,
],
```

### **3.2.3 Modelo User Personalizado**
```php
// app/Domain/User/Entities/User.php
class User extends Authenticatable implements HasApiTokens
{
    use HasApiTokens, HasFactory, Notifiable, SoftDeletes;

    protected $fillable = [
        'username', 'email', 'password', 'first_name', 'last_name', 
        'phone', 'is_active', 'avatar_path'
    ];

    protected $hidden = ['password', 'remember_token'];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'last_login_at' => 'datetime',
        'is_active' => 'boolean',
        'force_password_change' => 'boolean',
    ];

    // Relaciones
    public function companies()
    {
        return $this->belongsToMany(Company::class, 'user_companies')
                    ->withPivot('is_default', 'permissions_override')
                    ->withTimestamps();
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class, 'user_roles')
                    ->withPivot('company_id', 'assigned_at', 'assigned_by')
                    ->withTimestamps();
    }

    public function activeSessions()
    {
        return $this->hasMany(UserSession::class)->where('is_active', true);
    }

    // Métodos de negocio
    public function getFullNameAttribute()
    {
        return "{$this->first_name} {$this->last_name}";
    }

    public function hasAccessToCompany($companyId)
    {
        return $this->companies()->where('company_id', $companyId)->exists();
    }

    public function getDefaultCompany()
    {
        return $this->companies()->wherePivot('is_default', true)->first();
    }
}
```

---

## 🏢 PASO 3.3: SERVICIOS DE DOMINIO
**Tiempo estimado:** 120-150 minutos

### **3.3.1 Servicio de Autenticación**
```php
// app/Application/Services/Auth/AuthService.php
class AuthService
{
    public function __construct(
        private UserRepository $userRepository,
        private CompanyRepository $companyRepository,
        private SessionService $sessionService,
        private TenantService $tenantService
    ) {}

    public function login(LoginRequest $request): AuthResult
    {
        // 1. Validar credenciales
        $user = $this->validateCredentials($request->username, $request->password);
        
        // 2. Verificar que el usuario esté activo
        if (!$user->is_active) {
            throw new InactiveUserException();
        }

        // 3. Obtener empresas del usuario
        $companies = $user->companies()->where('companies.is_active', true)->get();
        
        if ($companies->isEmpty()) {
            throw new NoCompaniesAssignedException();
        }

        // 4. Si es login completo (con empresa seleccionada)
        if ($request->has('company_id')) {
            return $this->completeLogin($user, $request->company_id, $request);
        }

        // 5. Retornar empresas disponibles para selección
        return new AuthResult(
            success: true,
            user: $user,
            companies: $companies,
            requiresCompanySelection: true
        );
    }

    private function completeLogin(User $user, int $companyId, LoginRequest $request): AuthResult
    {
        // Verificar acceso a la empresa
        if (!$user->hasAccessToCompany($companyId)) {
            throw new UnauthorizedCompanyAccessException();
        }

        // Terminar sesiones anteriores (sesión única)
        $this->sessionService->terminateUserSessions($user->id);

        // Cambiar contexto de tenant
        $company = $this->companyRepository->find($companyId);
        $this->tenantService->switchTenant($company->schema_name);

        // Crear token JWT con contexto
        $token = $user->createToken(
            name: $request->device_name ?? 'web-browser',
            abilities: $this->getUserAbilities($user, $companyId)
        );

        // Registrar sesión activa
        $this->sessionService->createSession($user, $token, $request);

        // Actualizar último login
        $user->update([
            'last_login_at' => now(),
            'last_login_ip' => $request->ip()
        ]);

        return new AuthResult(
            success: true,
            user: $user,
            company: $company,
            token: $token->plainTextToken,
            permissions: $this->getUserPermissions($user, $companyId)
        );
    }

    private function getUserAbilities(User $user, int $companyId): array
    {
        // Lógica para obtener habilidades del usuario en la empresa
        return $this->permissionService->getUserAbilities($user->id, $companyId);
    }
}
```

### **3.3.2 Servicio de Permisos**
```php
// app/Application/Services/Permission/PermissionService.php
class PermissionService
{
    public function hasPermission(int $userId, int $companyId, string $module, string $action): bool
    {
        return Cache::remember(
            key: "user_permission_{$userId}_{$companyId}_{$module}_{$action}",
            ttl: 3600, // 1 hora
            callback: fn() => $this->checkPermissionFromDatabase($userId, $companyId, $module, $action)
        );
    }

    private function checkPermissionFromDatabase(int $userId, int $companyId, string $module, string $action): bool
    {
        return DB::table('user_roles')
            ->join('role_permissions', 'user_roles.role_id', '=', 'role_permissions.role_id')
            ->join('modules', 'role_permissions.module_id', '=', 'modules.id')
            ->join('actions', 'role_permissions.action_id', '=', 'actions.id')
            ->where('user_roles.user_id', $userId)
            ->where('user_roles.company_id', $companyId)
            ->where('modules.slug', $module)
            ->where('actions.slug', $action)
            ->where('role_permissions.granted', true)
            ->exists();
    }

    public function getUserPermissions(int $userId, int $companyId): array
    {
        return Cache::remember(
            key: "user_permissions_{$userId}_{$companyId}",
            ttl: 3600,
            callback: fn() => $this->buildPermissionTree($userId, $companyId)
        );
    }

    private function buildPermissionTree(int $userId, int $companyId): array
    {
        $permissions = DB::table('user_roles')
            ->join('role_permissions', 'user_roles.role_id', '=', 'role_permissions.role_id')
            ->join('modules', 'role_permissions.module_id', '=', 'modules.id')
            ->join('actions', 'role_permissions.action_id', '=', 'actions.id')
            ->where('user_roles.user_id', $userId)
            ->where('user_roles.company_id', $companyId)
            ->where('role_permissions.granted', true)
            ->select('modules.slug as module', 'actions.slug as action')
            ->get();

        $tree = [];
        foreach ($permissions as $permission) {
            $tree[$permission->module][] = $permission->action;
        }

        return $tree;
    }
}
```

### **3.3.3 Servicio de Sesiones**
```php
// app/Application/Services/Auth/SessionService.php
class SessionService
{
    public function createSession(User $user, $token, Request $request): UserSession
    {
        return UserSession::create([
            'user_id' => $user->id,
            'session_token' => hash('sha256', $token->plainTextToken),
            'device_name' => $request->device_name ?? $this->detectDevice($request),
            'ip_address' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'last_activity' => now(),
            'is_active' => true
        ]);
    }

    public function terminateUserSessions(int $userId): void
    {
        // Desactivar sesiones en BD
        UserSession::where('user_id', $userId)
            ->where('is_active', true)
            ->update(['is_active' => false]);

        // Revocar tokens de Sanctum
        PersonalAccessToken::where('tokenable_id', $userId)
            ->where('tokenable_type', User::class)
            ->delete();
    }

    public function updateActivity(string $tokenHash): void
    {
        UserSession::where('session_token', $tokenHash)
            ->update(['last_activity' => now()]);
    }
}
```

---

## 🛡️ PASO 3.4: MIDDLEWARE DE SEGURIDAD
**Tiempo estimado:** 60-90 minutos

### **3.4.1 Middleware de Autenticación Mejorado**
```php
// app/Http/Middleware/CustomAuthMiddleware.php
class CustomAuthMiddleware
{
    public function handle(Request $request, Closure $next, ...$guards)
    {
        // 1. Verificar token JWT
        if (!$token = $request->bearerToken()) {
            return $this->unauthorized('Token no proporcionado');
        }

        // 2. Validar token con Sanctum
        $user = $this->validateToken($token);
        if (!$user) {
            return $this->unauthorized('Token inválido');
        }

        // 3. Verificar que el usuario esté activo
        if (!$user->is_active) {
            return $this->unauthorized('Usuario inactivo');
        }

        // 4. Verificar sesión única
        if (!$this->isValidSession($user, $token)) {
            return $this->unauthorized('Sesión expirada');
        }

        // 5. Actualizar actividad
        $this->updateSessionActivity($token);

        // 6. Establecer contexto
        Auth::setUser($user);
        $request->merge(['current_user' => $user]);

        return $next($request);
    }

    private function isValidSession(User $user, string $token): bool
    {
        $tokenHash = hash('sha256', $token);
        
        return UserSession::where('user_id', $user->id)
            ->where('session_token', $tokenHash)
            ->where('is_active', true)
            ->where('last_activity', '>', now()->subHours(24))
            ->exists();
    }
}
```

### **3.4.2 Middleware de Permisos**
```php
// app/Http/Middleware/PermissionMiddleware.php
class PermissionMiddleware
{
    public function handle(Request $request, Closure $next, string $module, string $action)
    {
        $user = Auth::user();
        $companyId = session('current_company_id');

        if (!$this->permissionService->hasPermission($user->id, $companyId, $module, $action)) {
            abort(403, 'No tienes permisos para realizar esta acción');
        }

        return $next($request);
    }
}
```

---

## 🌐 PASO 3.5: CONTROLADORES DE API
**Tiempo estimado:** 90-120 minutos

### **3.5.1 Controlador de Autenticación**
```php
// app/Interfaces/Http/Controllers/Api/AuthController.php
class AuthController extends Controller
{
    public function __construct(
        private AuthService $authService,
        private CompanyRepository $companyRepository
    ) {}

    public function login(LoginRequest $request)
    {
        try {
            $result = $this->authService->login($request);

            if ($result->requiresCompanySelection) {
                return response()->json([
                    'success' => true,
                    'message' => 'Selecciona una empresa',
                    'requires_company_selection' => true,
                    'user' => new UserResource($result->user),
                    'companies' => CompanyResource::collection($result->companies)
                ]);
            }

            return response()->json([
                'success' => true,
                'message' => 'Login exitoso',
                'user' => new UserResource($result->user),
                'company' => new CompanyResource($result->company),
                'token' => $result->token,
                'permissions' => $result->permissions,
                'expires_in' => config('sanctum.expiration') * 60
            ]);

        } catch (InvalidCredentialsException $e) {
            return response()->json([
                'success' => false,
                'message' => 'Credenciales inválidas'
            ], 401);
        } catch (InactiveUserException $e) {
            return response()->json([
                'success' => false,
                'message' => 'Usuario inactivo'
            ], 403);
        }
    }

    public function switchCompany(SwitchCompanyRequest $request)
    {
        try {
            $user = Auth::user();
            $result = $this->authService->switchCompany($user, $request->company_id);

            return response()->json([
                'success' => true,
                'message' => 'Empresa cambiada exitosamente',
                'company' => new CompanyResource($result->company),
                'permissions' => $result->permissions
            ]);

        } catch (UnauthorizedCompanyAccessException $e) {
            return response()->json([
                'success' => false,
                'message' => 'No tienes acceso a esta empresa'
            ], 403);
        }
    }

    public function me(Request $request)
    {
        $user = Auth::user();
        $companyId = session('current_company_id');

        return response()->json([
            'success' => true,
            'user' => new UserResource($user),
            'current_company' => $companyId ? new CompanyResource($this->companyRepository->find($companyId)) : null,
            'permissions' => $companyId ? $this->permissionService->getUserPermissions($user->id, $companyId) : []
        ]);
    }

    public function logout(Request $request)
    {
        $user = Auth::user();
        
        // Terminar sesión actual
        $this->sessionService->terminateCurrentSession($user, $request->bearerToken());
        
        return response()->json([
            'success' => true,
            'message' => 'Sesión cerrada exitosamente'
        ]);
    }
}
```

---

## 📊 PASO 3.6: SEEDERS Y DATOS INICIALES
**Tiempo estimado:** 60-90 minutos

### **3.6.1 Seeder de Empresas**
```php
// database/seeders/CompanySeeder.php
class CompanySeeder extends Seeder
{
    public function run(): void
    {
        $companies = [
            [
                'code' => 'kreis',
                'name' => 'Kreis',
                'full_name' => 'Kreis Industrial S.A. de C.V.',
                'schema_name' => 'kreis',
                'brand_colors' => ['primary' => '#1f2937', 'secondary' => '#f59e0b']
            ],
            [
                'code' => 'gosan',
                'name' => 'Gosan',
                'full_name' => 'Gosan Construcciones S.A. de C.V.',
                'schema_name' => 'gosan',
                'brand_colors' => ['primary' => '#065f46', 'secondary' => '#10b981']
            ],
            // ... resto de empresas
        ];

        foreach ($companies as $company) {
            Company::create(array_merge($company, [
                'is_active' => true,
                'created_at' => now(),
                'updated_at' => now()
            ]));
        }
    }
}
```

### **3.6.2 Seeder de Módulos y Acciones**
```php
// database/seeders/ModuleSeeder.php
class ModuleSeeder extends Seeder
{
    public function run(): void
    {
        // Acciones básicas
        $actions = [
            ['name' => 'Ver', 'slug' => 'view'],
            ['name' => 'Crear', 'slug' => 'create'],
            ['name' => 'Editar', 'slug' => 'edit'],
            ['name' => 'Autorizar', 'slug' => 'authorize'],
            ['name' => 'Eliminar', 'slug' => 'delete']
        ];

        foreach ($actions as $action) {
            Action::create($action);
        }

        // Módulos básicos
        $modules = [
            ['name' => 'Dashboard', 'slug' => 'dashboard', 'icon' => 'home'],
            ['name' => 'Usuarios', 'slug' => 'users', 'icon' => 'users'],
            ['name' => 'Empresas', 'slug' => 'companies', 'icon' => 'building-office'],
            ['name' => 'Configuración', 'slug' => 'settings', 'icon' => 'cog'],
        ];

        foreach ($modules as $index => $module) {
            Module::create(array_merge($module, [
                'sort_order' => $index + 1,
                'is_active' => true
            ]));
        }
    }
}
```

### **3.6.3 Seeder de Usuario Super Admin**
```php
// database/seeders/SuperAdminSeeder.php
class SuperAdminSeeder extends Seeder
{
    public function run(): void
    {
        // Crear usuario super admin
        $user = User::create([
            'username' => 'superadmin',
            'email' => 'admin@grupofixum.com',
            'password' => Hash::make('FixumAdmin2025!'),
            'first_name' => 'Super',
            'last_name' => 'Administrator',
            'is_active' => true,
            'email_verified_at' => now()
        ]);

        // Asignar a todas las empresas
        $companies = Company::all();
        foreach ($companies as $company) {
            $user->companies()->attach($company->id, [
                'is_default' => $company->code === 'kreis'
            ]);

            // Crear rol super admin para cada empresa
            $role = Role::create([
                'name' => 'Super Administrador',
                'slug' => 'super-admin',
                'company_id' => $company->id,
                'is_system_role' => true,
                'description' => 'Acceso completo al sistema'
            ]);

            // Asignar todos los permisos
            $this->assignAllPermissions($role);

            // Asignar rol al usuario
            $user->roles()->attach($role->id, [
                'company_id' => $company->id,
                'assigned_at' => now()
            ]);
        }
    }

    private function assignAllPermissions(Role $role): void
    {
        $modules = Module::all();
        $actions = Action::all();

        foreach ($modules as $module) {
            foreach ($actions as $action) {
                RolePermission::create([
                    'role_id' => $role->id,
                    'module_id' => $module->id,
                    'action_id' => $action->id,
                    'granted' => true
                ]);
            }
        }
    }
}
```

---

## 🧪 PASO 3.7: TESTING Y VALIDACIÓN
**Tiempo estimado:** 60-90 minutos

### **3.7.1 Tests de Autenticación**
```php
// tests/Feature/AuthTest.php
class AuthTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_login_with_valid_credentials()
    {
        $this->seed([CompanySeeder::class, SuperAdminSeeder::class]);
        
        $response = $this->postJson('/api/auth/login', [
            'username' => 'superadmin',
            'password' => 'FixumAdmin2025!',
            'device_name' => 'test-device'
        ]);

        $response->assertStatus(200)
                ->assertJsonStructure([
                    'success',
                    'requires_company_selection',
                    'companies'
                ]);
    }

    public function test_user_cannot_login_with_invalid_credentials()
    {
        $response = $this->postJson('/api/auth/login', [
            'username' => 'wrong',
            'password' => 'wrong'
        ]);

        $response->assertStatus(401);
    }

    public function test_user_can_complete_login_with_company_selection()
    {
        $this->seed([CompanySeeder::class, SuperAdminSeeder::class]);
        $company = Company::first();
        
        $response = $this->postJson('/api/auth/login', [
            'username' => 'superadmin',
            'password' => 'FixumAdmin2025!',
            'company_id' => $company->id,
            'device_name' => 'test-device'
        ]);

        $response->assertStatus(200)
                ->assertJsonStructure([
                    'success',
                    'user',
                    'company',
                    'token',
                    'permissions'
                ]);
    }
}
```

### **3.7.2 API de Testing**
```php
// routes/api.php - Rutas de desarrollo
if (app()->environment(['local', 'development'])) {
    Route::prefix('dev')->group(function () {
        Route::get('/test-permissions/{userId}/{companyId}', function ($userId, $companyId) {
            $permissionService = app(PermissionService::class);
            return $permissionService->getUserPermissions($userId, $companyId);
        });
        
        Route::get('/test-session-cleanup', function () {
            $count = UserSession::where('last_activity', '<', now()->subHours(24))->count();
            UserSession::where('last_activity', '<', now()->subHours(24))->update(['is_active' => false]);
            return ['cleaned_sessions' => $count];
        });
    });
}
```

---

## 🔐 PASO 3.8: CONFIGURACIÓN DE SEGURIDAD AVANZADA
**Tiempo estimado:** 45-60 minutos

### **3.8.1 Rate Limiting Personalizado**
```php
// app/Http/Middleware/CustomRateLimiter.php
class CustomRateLimiter
{
    public function handle(Request $request, Closure $next, $maxAttempts = 5, $decayMinutes = 1)
    {
        $key = $this->resolveRequestSignature($request);
        
        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            $seconds = RateLimiter::availableIn($key);
            
            // Log intento sospechoso
            Log::warning('Rate limit exceeded', [
                'ip' => $request->ip(),
                'user_agent' => $request->userAgent(),
                'endpoint' => $request->path(),
                'retry_after' => $seconds
            ]);
            
            return response()->json([
                'success' => false,
                'message' => 'Demasiados intentos. Intenta de nuevo en ' . $seconds . ' segundos.',
                'retry_after' => $seconds
            ], 429);
        }
        
        RateLimiter::hit($key, $decayMinutes * 60);
        
        return $next($request);
    }
    
    private function resolveRequestSignature(Request $request): string
    {
        return sha1(
            $request->method() .
            '|' . $request->server('SERVER_NAME') .
            '|' . $request->path() .
            '|' . $request->ip()
        );
    }
}
```

### **3.8.2 Middleware de Auditoría**
```php
// app/Http/Middleware/AuditMiddleware.php
class AuditMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);
        
        // Solo auditar ciertas rutas
        if ($this->shouldAudit($request)) {
            dispatch(new LogUserActivity(
                userId: Auth::id(),
                action: $request->method() . ' ' . $request->path(),
                resource: $request->path(),
                ipAddress: $request->ip(),
                userAgent: $request->userAgent(),
                requestData: $this->sanitizeData($request->all()),
                responseStatus: $response->status(),
                companyId: session('current_company_id')
            ));
        }
        
        return $response;
    }
    
    private function shouldAudit(Request $request): bool
    {
        $auditRoutes = ['api/auth', 'api/users', 'api/companies', 'api/roles'];
        
        return collect($auditRoutes)->some(function ($route) use ($request) {
            return str_starts_with($request->path(), $route);
        });
    }
    
    private function sanitizeData(array $data): array
    {
        // Remover campos sensibles
        $sensitive = ['password', 'password_confirmation', 'token'];
        
        return collect($data)->except($sensitive)->toArray();
    }
}
```

### **3.8.3 Job de Limpieza de Sesiones**
```php
// app/Jobs/CleanupExpiredSessions.php
class CleanupExpiredSessions implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function handle(): void
    {
        $expiredSessions = UserSession::where('last_activity', '<', now()->subHours(24))
                                     ->where('is_active', true)
                                     ->get();
        
        foreach ($expiredSessions as $session) {
            // Desactivar sesión
            $session->update(['is_active' => false]);
            
            // Revocar token asociado
            PersonalAccessToken::where('tokenable_id', $session->user_id)
                              ->where('tokenable_type', User::class)
                              ->where('token', 'LIKE', '%' . substr($session->session_token, 0, 10) . '%')
                              ->delete();
        }
        
        Log::info('Cleaned up expired sessions', ['count' => $expiredSessions->count()]);
    }
}
```

---

## 📋 PASO 3.9: RUTAS Y RECURSOS API
**Tiempo estimado:** 60-75 minutos

### **3.9.1 Rutas de Autenticación**
```php
// routes/api.php
Route::prefix('auth')->group(function () {
    // Rutas públicas
    Route::post('login', [AuthController::class, 'login'])
         ->middleware(['throttle:5,1']); // 5 intentos por minuto
    
    Route::post('register', [AuthController::class, 'register'])
         ->middleware(['throttle:3,5']); // 3 intentos cada 5 minutos
    
    // Rutas protegidas
    Route::middleware(['auth:sanctum', 'tenant'])->group(function () {
        Route::get('me', [AuthController::class, 'me']);
        Route::post('logout', [AuthController::class, 'logout']);
        Route::post('switch-company', [AuthController::class, 'switchCompany']);
        Route::get('companies', [AuthController::class, 'getUserCompanies']);
        Route::post('refresh', [AuthController::class, 'refreshToken']);
    });
});

// Rutas protegidas por permisos
Route::middleware(['auth:sanctum', 'tenant', 'audit'])->group(function () {
    // Usuarios
    Route::apiResource('users', UserController::class)->middleware('permission:users,view');
    Route::post('users/{user}/toggle-status', [UserController::class, 'toggleStatus'])
         ->middleware('permission:users,edit');
    
    // Empresas
    Route::apiResource('companies', CompanyController::class)->middleware('permission:companies,view');
    
    // Roles y permisos
    Route::apiResource('roles', RoleController::class)->middleware('permission:roles,view');
    Route::get('permissions/tree', [PermissionController::class, 'getPermissionTree']);
    Route::post('roles/{role}/permissions', [RoleController::class, 'updatePermissions'])
         ->middleware('permission:roles,edit');
});
```

### **3.9.2 Recursos API (Resources)**
```php
// app/Http/Resources/UserResource.php
class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'username' => $this->username,
            'email' => $this->email,
            'first_name' => $this->first_name,
            'last_name' => $this->last_name,
            'full_name' => $this->full_name,
            'phone' => $this->phone,
            'is_active' => $this->is_active,
            'avatar_url' => $this->avatar_path ? Storage::url($this->avatar_path) : null,
            'last_login_at' => $this->last_login_at?->format('Y-m-d H:i:s'),
            'created_at' => $this->created_at->format('Y-m-d H:i:s'),
            
            // Información contextual (solo si está disponible)
            'companies' => $this->whenLoaded('companies', function () {
                return CompanyResource::collection($this->companies);
            }),
            
            'current_company_roles' => $this->when(
                session('current_company_id'),
                function () {
                    return $this->roles()
                               ->where('company_id', session('current_company_id'))
                               ->with('company')
                               ->get()
                               ->map(fn($role) => [
                                   'id' => $role->id,
                                   'name' => $role->name,
                                   'slug' => $role->slug
                               ]);
                }
            )
        ];
    }
}
```

```php
// app/Http/Resources/CompanyResource.php
class CompanyResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'code' => $this->code,
            'name' => $this->name,
            'full_name' => $this->full_name,
            'brand_colors' => $this->brand_colors,
            'logo_url' => $this->logo_path ? Storage::url($this->logo_path) : null,
            'is_active' => $this->is_active,
            
            // Información adicional solo para administradores
            $this->mergeWhen(auth()->user()?->can('view-company-details'), [
                'schema_name' => $this->schema_name,
                'description' => $this->description,
                'created_at' => $this->created_at->format('Y-m-d H:i:s'),
                'users_count' => $this->whenCounted('users'),
            ])
        ];
    }
}
```

---

## 🎯 PASO 3.10: COMANDOS ARTISAN PERSONALIZADOS
**Tiempo estimado:** 45-60 minutos

### **3.10.1 Comando para Crear Super Admin**
```php
// app/Console/Commands/CreateSuperAdmin.php
class CreateSuperAdmin extends Command
{
    protected $signature = 'fixum:create-super-admin 
                           {username : Username for the admin} 
                           {email : Email for the admin} 
                           {--password= : Password (will be generated if not provided)}';
    
    protected $description = 'Create a super administrator user with access to all companies';

    public function handle(): void
    {
        $username = $this->argument('username');
        $email = $this->argument('email');
        $password = $this->option('password') ?: Str::random(12);

        // Verificar si el usuario ya existe
        if (User::where('username', $username)->orWhere('email', $email)->exists()) {
            $this->error('❌ Usuario ya existe con ese username o email');
            return;
        }

        $this->info("🔄 Creando super administrador...");

        // Crear usuario
        $user = User::create([
            'username' => $username,
            'email' => $email,
            'password' => Hash::make($password),
            'first_name' => 'Super',
            'last_name' => 'Administrator',
            'is_active' => true,
            'email_verified_at' => now()
        ]);

        // Asignar a todas las empresas activas
        $companies = Company::where('is_active', true)->get();
        
        foreach ($companies as $index => $company) {
            $user->companies()->attach($company->id, [
                'is_default' => $index === 0 // Primera empresa como default
            ]);

            // Buscar o crear rol super admin
            $role = Role::firstOrCreate([
                'slug' => 'super-admin',
                'company_id' => $company->id
            ], [
                'name' => 'Super Administrador',
                'is_system_role' => true,
                'description' => 'Acceso completo al sistema'
            ]);

            // Asignar todos los permisos si es un rol nuevo
            if ($role->wasRecentlyCreated) {
                $this->assignAllPermissions($role);
            }

            // Asignar rol al usuario
            $user->roles()->attach($role->id, [
                'company_id' => $company->id,
                'assigned_at' => now()
            ]);
        }

        $this->info("✅ Super administrador creado exitosamente:");
        $this->table(
            ['Campo', 'Valor'],
            [
                ['Username', $username],
                ['Email', $email],
                ['Password', $password],
                ['Empresas', $companies->count()],
                ['ID', $user->id]
            ]
        );

        $this->warn("⚠️  IMPORTANTE: Guarda estas credenciales de forma segura");
    }

    private function assignAllPermissions(Role $role): void
    {
        $modules = Module::where('is_active', true)->get();
        $actions = Action::all();

        foreach ($modules as $module) {
            foreach ($actions as $action) {
                RolePermission::create([
                    'role_id' => $role->id,
                    'module_id' => $module->id,
                    'action_id' => $action->id,
                    'granted' => true
                ]);
            }
        }
    }
}
```

### **3.10.2 Comando de Limpieza de Sesiones**
```php
// app/Console/Commands/CleanExpiredSessions.php
class CleanExpiredSessions extends Command
{
    protected $signature = 'fixum:clean-sessions {--dry-run : Show what would be cleaned without actually doing it}';
    protected $description = 'Clean expired user sessions and tokens';

    public function handle(): void
    {
        $dryRun = $this->option('dry-run');
        
        $expiredSessions = UserSession::where('last_activity', '<', now()->subHours(24))
                                     ->where('is_active', true);
        
        $count = $expiredSessions->count();
        
        if ($count === 0) {
            $this->info('✅ No hay sesiones expiradas para limpiar');
            return;
        }

        if ($dryRun) {
            $this->table(
                ['ID', 'Usuario', 'IP', 'Última Actividad'],
                $expiredSessions->with('user:id,username,email')
                               ->get()
                               ->map(fn($session) => [
                                   $session->id,
                                   $session->user->username,
                                   $session->ip_address,
                                   $session->last_activity->diffForHumans()
                               ])
            );
            
            $this->warn("🔍 DRY RUN: Se limpiarían {$count} sesiones");
            return;
        }

        if ($this->confirm("¿Limpiar {$count} sesiones expiradas?")) {
            // Limpiar sesiones
            $expiredSessions->update(['is_active' => false]);
            
            // Limpiar tokens asociados
            PersonalAccessToken::whereIn('tokenable_id', 
                $expiredSessions->pluck('user_id')
            )->where('tokenable_type', User::class)->delete();
            
            $this->info("✅ {$count} sesiones limpiadas exitosamente");
        }
    }
}
```

---

## 📊 CHECKLIST FINAL FASE 3

### **✅ Base de Datos:**
- [ ] Migración de empresas completada
- [ ] Migración de usuarios y relaciones
- [ ] Sistema de permisos granular (5 niveles)
- [ ] Tablas de sesiones y auditoría
- [ ] Índices de performance configurados

### **✅ Autenticación:**
- [ ] Laravel Sanctum configurado
- [ ] JWT tokens funcionando
- [ ] Sesión única por usuario
- [ ] Login multi-empresa implementado
- [ ] Middleware de autenticación

### **✅ Autorización:**
- [ ] Servicio de permisos funcional
- [ ] Cache de permisos implementado
- [ ] Middleware de permisos granular
- [ ] Roles y permisos por empresa

### **✅ Seguridad:**
- [ ] Rate limiting configurado
- [ ] Auditoría de acciones
- [ ] Limpieza automática de sesiones
- [ ] Logs de seguridad

### **✅ APIs:**
- [ ] Endpoints de autenticación
- [ ] Resources para respuestas
- [ ] Manejo de errores
- [ ] Documentación básica

### **✅ Testing:**
- [ ] Tests de autenticación
- [ ] Tests de permisos
- [ ] Rutas de desarrollo
- [ ] Comandos de utilidad

---

## 🎯 ENTREGABLES DE LA FASE 3

1. **Sistema de autenticación JWT** completo y funcional
2. **Permisos granulares** a 5 niveles implementados  
3. **Sesión única** por usuario con expulsión automática
4. **APIs de autenticación** documentadas y probadas
5. **Middleware de seguridad** robusto
6. **Sistema de auditoría** básico funcionando
7. **Comandos administrativos** para gestión
8. **Tests automatizados** para funcionalidad crítica

---

## 🚀 PRÓXIMO PASO: FASE 4

Una vez completada la Fase 3, estaremos listos para:
- Instalar y configurar Vue 3 + Nuxt 3
- Crear las páginas de login multi-empresa
- Desarrollar el dashboard base
- Implementar el cambio de empresa en frontend
- Configurar el sistema de temas (oscuro/claro)

**La infraestructura de backend estará 100% sólida para construir la interfaz de usuario moderna y responsiva.**

### **Verificación antes de continuar:**
```bash
# Ejecutar migraciones
php artisan tenant:migrate

# Crear super admin
php artisan fixum:create-super-admin admin admin@grupofixum.com

# Ejecutar seeders
php artisan db:seed --class=CompanySeeder
php artisan db:seed --class=ModuleSeeder

# Test de API
curl -X POST https://erp.tudominio.com/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"generated_password"}'
```

**¿Todo funcionando? ¿Continuamos con la Fase 4 del frontend?** 🎨