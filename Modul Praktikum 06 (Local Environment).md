# WEEK 6: API Development & Authentication
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 6
- **Durasi**: 100 menit  
- **Topik**: Advanced API Development dengan Laravel Sanctum Authentication
- **Target**: Mahasiswa Semester 6
- **Platform**: Laptop/PC (Windows, macOS, atau Linux)

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Mengimplementasikan Laravel Sanctum untuk API authentication dan authorization
2. Membangun comprehensive RESTful API dengan proper versioning dan documentation
3. Menerapkan middleware untuk rate limiting dan request validation
4. Mengembangkan token-based authentication dengan secure token management
5. Mengimplementasikan role-based access control (RBAC) untuk API endpoints
6. Membangun API testing framework dengan automated test suites
7. Menerapkan API security best practices dan vulnerability protection

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1-5 telah completed dengan CRUD operations functional
- Laravel application dengan working database relationships
- Basic API endpoints dari Week 4 sudah implemented
- Understanding konsep REST API principles dan HTTP methods

**Environment Verification:**
```bash
# Pastikan berada di direktori project Laravel

# Verifikasi existing API endpoints dan database
php artisan route:list | grep api
php artisan tinker --execute="
echo 'Users count: ' . \App\Models\User::count() . PHP_EOL;
echo 'Posts count: ' . \App\Models\Post::count() . PHP_EOL;
echo 'Existing API routes: ' . \Route::getRoutes()->count() . PHP_EOL;
"

# Check Laravel version compatibility dengan Sanctum
php artisan --version
composer show laravel/sanctum
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Laravel Sanctum Installation & Configuration (20 menit)**

##### Step 1.1: Install Laravel Sanctum Package
```bash
# Install Laravel Sanctum (compatible with Laravel 12)
composer require laravel/sanctum

# (Opsional) Publish Sanctum configuration jika perlu kustomisasi
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

# Verify published files
ls config/sanctum.php
ls database/migrations
```

##### Step 1.2: Configure Sanctum Settings

**Update config/sanctum.php dengan custom settings `config/sanctum.php`:**
```php
<?php

use Laravel\Sanctum\Sanctum;
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;
use Illuminate\Cookie\Middleware\EncryptCookies;

return [
    /*
    |--------------------------------------------------------------------------
    | Stateful Domains - Praktikum Cloud Computing ITK
    |--------------------------------------------------------------------------
    |
    | Requests dari domains ini akan receive stateful API authentication
    | using Laravel's default session cookies. Typically, frontend domains.
    |
    */
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,localhost:8000,127.0.0.1,127.0.0.1:8000,::1',
        Sanctum::currentApplicationUrlWithPort()
    ))),

    /*
    |--------------------------------------------------------------------------
    | Sanctum Guards - Multi-Guard Authentication
    |--------------------------------------------------------------------------
    |
    | Array of authentication guards yang dapat menggunakan Sanctum tokens.
    | Default guard adalah 'web' untuk browser sessions dan 'sanctum' untuk API.
    |
    */
    'guard' => ['web'],

    /*
    |--------------------------------------------------------------------------
    | Expiration Minutes - Token Lifetime Management
    |--------------------------------------------------------------------------
    |
    | Berapa lama (dalam menit) tokens tetap valid. Default null = forever.
    | Untuk production, sebaiknya set expiration time untuk security.
    |
    */
    'expiration' => env('SANCTUM_TOKEN_EXPIRATION', 1440), // 24 hours

    /*
    |--------------------------------------------------------------------------
    | Token Prefix - Untuk identifikasi token format
    |--------------------------------------------------------------------------
    |
    | Prefix yang ditambahkan ke generated tokens untuk identifikasi.
    | Berguna untuk debugging dan token management.
    |
    */
    //'token_prefix' => env('SANCTUM_TOKEN_PREFIX', 'ccitk'),

    /*
    |--------------------------------------------------------------------------
    | Middleware Configuration
    |--------------------------------------------------------------------------
    |
    | Middleware yang akan digunakan untuk memverifikasi incoming requests
    | yang menggunakan Sanctum authentication.
    |
    */
    'middleware' => [
        'verify_csrf_token' => Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
    ],

    /*
    |--------------------------------------------------------------------------
    | Rate Limiting Configuration
    |--------------------------------------------------------------------------
    |
    | Konfigurasi rate limiting untuk API endpoints.
    | Dapat dikustomisasi berdasarkan authenticated vs guest users.
    |
    */
    'api_rate_limit' => [
        'authenticated' => env('SANCTUM_RATE_LIMIT_AUTH', '200,1'), // 200 requests per minute
        'guest' => env('SANCTUM_RATE_LIMIT_GUEST', '60,1'),        // 60 requests per minute
    ],
];
```

##### Step 1.3: Update Environment Configuration

**Update environment configuration `.env`:**
```bash
# === Sanctum Configuration ===
SANCTUM_STATEFUL_DOMAINS=localhost,localhost:8000,127.0.0.1:8000
SANCTUM_TOKEN_EXPIRATION=1440
SANCTUM_TOKEN_PREFIX=ccitk
SANCTUM_RATE_LIMIT_AUTH=200,1
SANCTUM_RATE_LIMIT_GUEST=60,1

# === API Configuration ===
API_VERSION=v1
API_PREFIX=api
API_DEBUG=true
```

##### Step 1.4: Edit file migrations `2025_09_29_000001_create_personal_access_tokens_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('personal_access_tokens', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('tokenable_type');
            $table->unsignedBigInteger('tokenable_id');
            $table->string('name');
            $table->string('token', 64)->unique();
            $table->text('abilities')->nullable();
            $table->timestamp('last_used_at')->nullable();
            $table->timestamp('expires_at')->nullable();
            $table->timestamps();

            $table->index(['tokenable_type', 'tokenable_id'], 'personal_access_tokens_tokenable_type_tokenable_id_index');
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('personal_access_tokens');
    }
};
```

##### Step 1.5: Run Sanctum Migrations
```bash
# Run migration untuk personal access tokens table
php artisan migrate

# Jika personal access tokens table sudah terbuat, maka dihapus terlebih dahulu lalu migrate ulang

# Verify migration success
mysql -u laravel_user -p laravel_app -e "DESCRIBE personal_access_tokens;"
```

##### Step 1.6: Configure User Model untuk Sanctum

**Update User model dengan Sanctum integration `app/Models/User.php`:**
```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens; // Import Sanctum trait

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable; // Add HasApiTokens

    /**
     * Atribut yang dapat diisi secara mass assignment
     */
    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
        'bio',
        'avatar',
        'is_active',
        'email_verified_at', // Tambah untuk email verification
    ];

    /**
     * Atribut yang disembunyikan dari serialization
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];

    /**
     * Casting atribut ke tipe data yang sesuai
     */
    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
        'is_active' => 'boolean',
    ];

    /**
     * Sanctum: Create token dengan custom abilities dan expiration
     */
    public function createApiToken(string $name, array $abilities = ['*'], \DateTimeInterface $expiresAt = null)
    {
        $token = $this->createToken($name, $abilities, $expiresAt);
        
        // Log token creation untuk audit
        \Log::info('API token created', [
            'user_id' => $this->id,
            'token_name' => $name,
            'abilities' => $abilities,
            'expires_at' => $expiresAt?->toISOString(),
            'created_at' => now()->toISOString()
        ]);
        
        return $token;
    }

    /**
     * Sanctum: Get user's active tokens dengan information
     */
    public function getActiveTokensAttribute()
    {
        return $this->tokens()
                   ->where('expires_at', '>', now())
                   ->orWhereNull('expires_at')
                   ->get()
                   ->map(function ($token) {
                       return [
                           'id' => $token->id,
                           'name' => $token->name,
                           'abilities' => $token->abilities,
                           'last_used_at' => $token->last_used_at,
                           'expires_at' => $token->expires_at,
                           'created_at' => $token->created_at,
                       ];
                   });
    }

    /**
     * Sanctum: Revoke all user tokens (logout from all devices)
     */
    public function revokeAllTokens()
    {
        $tokenCount = $this->tokens()->count();
        $this->tokens()->delete();
        
        \Log::info('All API tokens revoked', [
            'user_id' => $this->id,
            'revoked_count' => $tokenCount,
            'revoked_at' => now()->toISOString()
        ]);
        
        return $tokenCount;
    }

    /**
     * Check if user has specific API ability
     */
    public function hasApiAbility(string $ability): bool
    {
        $token = $this->currentAccessToken();
        
        if (!$token) {
            return false;
        }
        
        return in_array('*', $token->abilities) || in_array($ability, $token->abilities);
    }

    /**
     * Get current token information
     */
    public function getCurrentTokenInfo()
    {
        $token = $this->currentAccessToken();
        
        if (!$token) {
            return null;
        }
        
        return [
            'id' => $token->id,
            'name' => $token->name,
            'abilities' => $token->abilities,
            'last_used_at' => $token->last_used_at,
            'created_at' => $token->created_at,
        ];
    }

    // Existing relationships dan methods dari week sebelumnya
    public function posts() { return $this->hasMany(Post::class); }
    public function comments() { return $this->hasMany(Comment::class); }
    public function mediaUploads() { return $this->hasMany(Media::class, 'uploaded_by'); }
    
    // Existing scopes dan methods
    public function scopeRole($query, $role) { return $query->where('role', $role); }
    public function scopeActive($query) { return $query->where('is_active', true); }
    public function hasRole($role) { return $this->role === $role; }
    public function canManagePosts() { return in_array($this->role, ['admin', 'editor', 'author']); }
    public function canManageUsers() { return $this->role === 'admin'; }
}
```

##### Step 1.7: Define API Rate Limiter (throttle:api)

**Tambahkan konfigurasi rate limiter `api` di `app/Providers/AppServiceProvider.php`:**
```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Http\Request;
use Illuminate\Cache\RateLimiting\Limit;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        //
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        // Define 'api' rate limiter used by 'throttle:api'
        RateLimiter::for('api', function (Request $request) {
            $userId = optional($request->user())->id;
            // Default 60 per minute if not configured
            $limit = config('sanctum.api_rate_limit.authenticated', '200,1');
            if (!$userId) {
                $limit = config('sanctum.api_rate_limit.guest', '60,1');
            }

            $max = 60; $decay = 1;
            if (is_string($limit) && str_contains($limit, ',')) {
                [$max, $decay] = explode(',', $limit);
            }
            $max = (int) $max; $decay = (int) $decay;

            return Limit::perMinutes($decay, $max)->by($userId ?: $request->ip());
        });
    }
}
```

#### **Bagian 2: Authentication Controllers & API Endpoints (25 menit)**

##### Step 2.1: Create Authentication Controllers
```bash
# Generate controllers untuk authentication
php artisan make:controller Api/AuthController
php artisan make:controller Api/UserController --api --model=User

# Generate middleware untuk role-based access
php artisan make:middleware CheckApiRole
php artisan make:middleware ApiRateLimit

# Verify controllers created
ls app/Http/Controllers/Api/
ls app/Http/Middleware/
```

##### Step 2.2: Implement AuthController dengan Comprehensive Features

**Isi file `app/Http/Controllers/Api/AuthController.php`:**
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    /**
     * Register new user account
     */
    public function register(Request $request)
    {
        // Rate limiting untuk registration attempts
        $key = 'register:' . $request->ip();
        if (RateLimiter::tooManyAttempts($key, 5)) {
            $seconds = RateLimiter::availableIn($key);
            return response()->json([
                'success' => false,
                'message' => 'Too many registration attempts. Please try again in ' . $seconds . ' seconds.',
                'retry_after' => $seconds
            ], 429);
        }

        // Validation rules untuk registration
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
            'role' => 'nullable|in:subscriber,author,editor', // Admin tidak bisa register via API
            'bio' => 'nullable|string|max:1000',
        ]);

        if ($validator->fails()) {
            RateLimiter::hit($key, 300); // 5 minute penalty untuk invalid attempts
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors()
            ], 422);
        }

        try {
            // Create new user
            $user = User::create([
                'name' => $request->name,
                'email' => $request->email,
                'password' => Hash::make($request->password),
                'role' => $request->role ?? 'subscriber',
                'bio' => $request->bio,
                'is_active' => true,
                'email_verified_at' => null, // Require email verification
            ]);

            // Create default API token dengan limited abilities
            $abilities = $this->getDefaultAbilitiesForRole($user->role);
            $token = $user->createApiToken(
                'registration-token',
                $abilities,
                now()->addDays(1) // 24 hours untuk initial token
            );

            // Clear rate limiter on successful registration
            RateLimiter::clear($key);

            // Return user data dan token
            return response()->json([
                'success' => true,
                'message' => 'User registered successfully',
                'data' => [
                    'user' => [
                        'id' => $user->id,
                        'name' => $user->name,
                        'email' => $user->email,
                        'role' => $user->role,
                        'bio' => $user->bio,
                        'email_verified_at' => $user->email_verified_at,
                        'created_at' => $user->created_at,
                    ],
                    'token' => [
                        'access_token' => $token->plainTextToken,
                        'token_type' => 'Bearer',
                        'abilities' => $abilities,
                        'expires_at' => $token->accessToken->expires_at?->toISOString(),
                    ]
                ],
                'meta' => [
                    'requires_email_verification' => true,
                    'default_abilities' => $abilities,
                ]
            ], 201);

        } catch (\Exception $e) {
            \Log::error('User registration failed', [
                'email' => $request->email,
                'error' => $e->getMessage(),
                'ip' => $request->ip()
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Registration failed. Please try again.',
            ], 500);
        }
    }

    /**
     * Login user dan generate API token
     */
    public function login(Request $request)
    {
        // Rate limiting untuk login attempts
        $key = 'login:' . $request->ip();
        if (RateLimiter::tooManyAttempts($key, 10)) {
            $seconds = RateLimiter::availableIn($key);
            return response()->json([
                'success' => false,
                'message' => 'Too many login attempts. Please try again in ' . $seconds . ' seconds.',
                'retry_after' => $seconds
            ], 429);
        }

        // Validation
        $validator = Validator::make($request->all(), [
            'email' => 'required|email',
            'password' => 'required|string',
            'device_name' => 'nullable|string|max:255',
            'remember_me' => 'boolean',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors()
            ], 422);
        }

        // Find user dan verify credentials
        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            RateLimiter::hit($key, 300); // 5 minute penalty
            
            return response()->json([
                'success' => false,
                'message' => 'Invalid credentials',
            ], 401);
        }

        // Check if user is active
        if (!$user->is_active) {
            return response()->json([
                'success' => false,
                'message' => 'Account is deactivated. Please contact administrator.',
            ], 403);
        }

        try {
            // Determine token expiration based on remember_me
            $expiresAt = $request->boolean('remember_me') 
                ? now()->addDays(30)  // 30 days untuk remember me
                : now()->addDay();    // 1 day untuk normal login

            // Get abilities berdasarkan user role
            $abilities = $this->getAbilitiesForRole($user->role);

            // Create API token
            $deviceName = $request->device_name ?? 'Unknown Device';
            $token = $user->createApiToken($deviceName, $abilities, $expiresAt);

            // Clear rate limiter on successful login
            RateLimiter::clear($key);

            // Update last login time (optional field)
            $user->update(['last_login_at' => now()]);

            return response()->json([
                'success' => true,
                'message' => 'Login successful',
                'data' => [
                    'user' => [
                        'id' => $user->id,
                        'name' => $user->name,
                        'email' => $user->email,
                        'role' => $user->role,
                        'bio' => $user->bio,
                        'avatar_url' => $user->avatar_url,
                        'email_verified_at' => $user->email_verified_at,
                    ],
                    'token' => [
                        'access_token' => $token->plainTextToken,
                        'token_type' => 'Bearer',
                        'abilities' => $abilities,
                        'expires_at' => $expiresAt->toISOString(),
                    ]
                ],
                'meta' => [
                    'device_name' => $deviceName,
                    'remember_me' => $request->boolean('remember_me'),
                    'login_time' => now()->toISOString(),
                ]
            ], 200);

        } catch (\Exception $e) {
            \Log::error('Login failed', [
                'user_id' => $user->id,
                'error' => $e->getMessage(),
                'ip' => $request->ip()
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Login failed. Please try again.',
            ], 500);
        }
    }

    /**
     * Get current authenticated user information
     */
    public function me(Request $request)
    {
        $user = $request->user();
        $tokenInfo = $user->getCurrentTokenInfo();

        return response()->json([
            'success' => true,
            'data' => [
                'user' => [
                    'id' => $user->id,
                    'name' => $user->name,
                    'email' => $user->email,
                    'role' => $user->role,
                    'bio' => $user->bio,
                    'avatar_url' => $user->avatar_url,
                    'is_active' => $user->is_active,
                    'email_verified_at' => $user->email_verified_at,
                    'created_at' => $user->created_at,
                    'updated_at' => $user->updated_at,
                ],
                'token_info' => $tokenInfo,
                'permissions' => [
                    'can_manage_posts' => $user->canManagePosts(),
                    'can_manage_users' => $user->canManageUsers(),
                    'abilities' => $tokenInfo['abilities'] ?? [],
                ]
            ]
        ]);
    }

    /**
     * Logout current session (revoke current token)
     */
    public function logout(Request $request)
    {
        $user = $request->user();
        $currentToken = $user->currentAccessToken();
        
        if ($currentToken) {
            $tokenName = $currentToken->name;
            $currentToken->delete();
            
            \Log::info('User logged out', [
                'user_id' => $user->id,
                'token_name' => $tokenName,
                'logout_time' => now()->toISOString()
            ]);
        }

        return response()->json([
            'success' => true,
            'message' => 'Logged out successfully'
        ]);
    }

    /**
     * Logout from all devices (revoke all tokens)
     */
    public function logoutAll(Request $request)
    {
        $user = $request->user();
        $revokedCount = $user->revokeAllTokens();

        return response()->json([
            'success' => true,
            'message' => "Logged out from all devices successfully",
            'revoked_tokens' => $revokedCount
        ]);
    }

    /**
     * Get user's active sessions/tokens
     */
    public function sessions(Request $request)
    {
        $user = $request->user();
        $activeTokens = $user->active_tokens;

        return response()->json([
            'success' => true,
            'data' => [
                'active_sessions' => $activeTokens,
                'total_count' => $activeTokens->count(),
            ]
        ]);
    }

    /**
     * Revoke specific token/session
     */
    public function revokeSession(Request $request, $tokenId)
    {
        $user = $request->user();
        $token = $user->tokens()->find($tokenId);

        if (!$token) {
            return response()->json([
                'success' => false,
                'message' => 'Session not found'
            ], 404);
        }

        $tokenName = $token->name;
        $token->delete();

        return response()->json([
            'success' => true,
            'message' => "Session '{$tokenName}' revoked successfully"
        ]);
    }

    /*
    |--------------------------------------------------------------------------
    | Helper Methods
    |--------------------------------------------------------------------------
    */

    /**
     * Get default abilities for user role
     */
    private function getDefaultAbilitiesForRole(string $role): array
    {
        return match($role) {
            'admin' => ['*'], // Full access
            'editor' => [
                'posts:create', 'posts:read', 'posts:update', 'posts:delete',
                'categories:read', 'categories:create', 'categories:update',
                'tags:read', 'tags:create', 'tags:update',
                'comments:read', 'comments:moderate'
            ],
            'author' => [
                'posts:create', 'posts:read', 'posts:update',
                'categories:read', 'tags:read',
                'comments:read'
            ],
            'subscriber' => [
                'posts:read', 'categories:read', 'tags:read',
                'comments:create', 'comments:read'
            ],
            default => ['posts:read', 'categories:read']
        };
    }

    /**
     * Get full abilities for user role (after email verification, etc.)
     */
    private function getAbilitiesForRole(string $role): array
    {
        // Same as default for now, tapi bisa diperluas untuk verified users
        return $this->getDefaultAbilitiesForRole($role);
    }
}
```

##### Step 2.3: Create Custom Middleware untuk Role-Based Access

**Isi file `app/Http/Middleware/CheckApiRole.php`:**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckApiRole
{
    /**
     * Handle incoming request dengan role-based authorization
     */
    public function handle(Request $request, Closure $next, ...$roles)
    {
        // Check if user is authenticated
        if (!$request->user()) {
            return response()->json([
                'success' => false,
                'message' => 'Authentication required',
                'error_code' => 'UNAUTHENTICATED'
            ], 401);
        }

        $user = $request->user();

        // Check if user is active
        if (!$user->is_active) {
            return response()->json([
                'success' => false,
                'message' => 'Account is deactivated',
                'error_code' => 'ACCOUNT_DEACTIVATED'
            ], 403);
        }

        // Check if user has required role
        if (!empty($roles) && !in_array($user->role, $roles)) {
            \Log::warning('Unauthorized role access attempt', [
                'user_id' => $user->id,
                'user_role' => $user->role,
                'required_roles' => $roles,
                'endpoint' => $request->path(),
                'method' => $request->method(),
                'ip' => $request->ip()
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Insufficient permissions for this action',
                'error_code' => 'INSUFFICIENT_PERMISSIONS',
                'required_roles' => $roles,
                'current_role' => $user->role
            ], 403);
        }

        return $next($request);
    }
}
```

##### Step 2.4: Create API Rate Limiting Middleware

**Isi file `app/Http/Middleware/ApiRateLimit.php`:**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class ApiRateLimit
{
    /**
     * Handle incoming request dengan dynamic rate limiting
     */
    public function handle(Request $request, Closure $next, $maxAttempts = 60, $decayMinutes = 1)
    {
        // Generate rate limit key berdasarkan authentication status
        if ($request->user()) {
            // Authenticated users - higher limits
            $key = 'api:auth:' . $request->user()->id;
            $maxAttempts = config('sanctum.api_rate_limit.authenticated', '200,1');
        } else {
            // Guest users - lower limits
            $key = 'api:guest:' . $request->ip();
            $maxAttempts = config('sanctum.api_rate_limit.guest', '60,1');
        }

        // Parse rate limit format "requests,minutes"
        if (is_string($maxAttempts) && str_contains($maxAttempts, ',')) {
            [$maxAttempts, $decayMinutes] = explode(',', $maxAttempts);
        }

        $maxAttempts = (int) $maxAttempts;
        $decayMinutes = (int) $decayMinutes;

        // Check rate limit
        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            $seconds = RateLimiter::availableIn($key);
            
            // Log rate limit exceeded
            \Log::warning('API rate limit exceeded', [
                'key' => $key,
                'max_attempts' => $maxAttempts,
                'decay_minutes' => $decayMinutes,
                'retry_after' => $seconds,
                'user_id' => $request->user()?->id,
                'ip' => $request->ip(),
                'endpoint' => $request->path(),
                'user_agent' => $request->userAgent()
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Rate limit exceeded',
                'error_code' => 'RATE_LIMIT_EXCEEDED',
                'retry_after' => $seconds,
                'limit_info' => [
                    'max_attempts' => $maxAttempts,
                    'window_minutes' => $decayMinutes,
                    'available_in_seconds' => $seconds
                ]
            ], 429)->header('Retry-After', $seconds);
        }

        // Hit the rate limiter
        RateLimiter::hit($key, $decayMinutes * 60);

        $response = $next($request);

        // Add rate limit headers ke response
        $remaining = $maxAttempts - RateLimiter::attempts($key);
        $response->headers->add([
            'X-RateLimit-Limit' => $maxAttempts,
            'X-RateLimit-Remaining' => max(0, $remaining),
            'X-RateLimit-Reset' => RateLimiter::availableIn($key) + time(),
        ]);

        return $response;
    }
}
```

##### Step 2.5: Register Middleware

**Update `bootstrap/app.php` untuk register middleware:**
```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',   // <-- tambahkan baris ini
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Register middleware aliases
        $middleware->alias([
            'api.role' => App\Http\Middleware\CheckApiRole::class,
            'api.rate' => App\Http\Middleware\ApiRateLimit::class,
        ]);

        // Ensure Sanctum's stateful middleware and our API limits apply to API routes
        $middleware->group('api', [
            Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            App\Http\Middleware\ApiRateLimit::class,
            'throttle:api',
            Illuminate\Routing\Middleware\SubstituteBindings::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })
    ->create();
```

#### **Bagian 3: API Routes dengan Authentication dan Authorization (20 menit)**

##### Step 3.1: Update API Routes dengan Authentication

**Update `routes/api.php` dengan comprehensive API structure:**
```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\PostApiController;
use App\Http\Controllers\Api\CategoryApiController;
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\UserController;

/*
|--------------------------------------------------------------------------
| API Routes - Praktikum Cloud Computing ITK Week 4
|--------------------------------------------------------------------------
|
| API routes untuk AJAX operations dan frontend integration
| Semua routes menggunakan prefix 'api' dan middleware 'api'
|
*/

// API info endpoint
Route::get('/info', function (Request $request) {
    return response()->json([
        'application' => 'Praktikum Cloud Computing ITK',
        'version' => '1.0.0',
        'laravel_version' => app()->version(),
        'api_version' => 'v1',
        'timestamp' => now()->toISOString(),
        'endpoints' => [
            'posts' => '/api/posts',
            'categories' => '/api/categories',
            'search' => '/api/posts/search',
        ]
    ]);
});

/*
|--------------------------------------------------------------------------
| Posts API Routes
|--------------------------------------------------------------------------
*/

// RESTful API untuk posts
Route::apiResource('posts', PostApiController::class);

// Additional posts API endpoints
Route::prefix('posts')->group(function () {
    // Search posts by keyword
    Route::get('search/{keyword}', [PostApiController::class, 'search'])
          ->name('api.posts.search');
    
    // Get posts by category
    Route::get('category/{category}', [PostApiController::class, 'byCategory'])
          ->name('api.posts.by-category');
    
    // Get posts by tag
    Route::get('tag/{tag}', [PostApiController::class, 'byTag'])
          ->name('api.posts.by-tag');
    
    // Get featured posts
    Route::get('featured', [PostApiController::class, 'featured'])
          ->name('api.posts.featured');
    
    // Get published posts only
    Route::get('published', [PostApiController::class, 'published'])
          ->name('api.posts.published');
});

/*
|--------------------------------------------------------------------------
| Categories API Routes
|--------------------------------------------------------------------------
*/

// RESTful API untuk categories
Route::apiResource('categories', CategoryApiController::class);

// Additional categories API endpoints
Route::prefix('categories')->group(function () {
    // Get category tree (hierarchical structure)
    Route::get('tree', [CategoryApiController::class, 'tree'])
          ->name('api.categories.tree');
    
    // Get active categories only
    Route::get('active', [CategoryApiController::class, 'active'])
          ->name('api.categories.active');
});

/*
|--------------------------------------------------------------------------
| Statistics API Routes
|--------------------------------------------------------------------------
*/

// Dashboard statistics untuk admin interface
Route::get('/stats', function () {
    return response()->json([
        'total_posts' => \App\Models\Post::count(),
        'published_posts' => \App\Models\Post::published()->count(),
        'draft_posts' => \App\Models\Post::where('status', 'draft')->count(),
        'total_categories' => \App\Models\Category::count(),
        'total_tags' => \App\Models\Tag::count(),
        'total_users' => \App\Models\User::count(),
        'recent_posts' => \App\Models\Post::latest()->take(5)->pluck('title'),
        'generated_at' => now()->toISOString(),
    ]);
})->name('api.stats');

// Tambahkan setelah existing API routes

/*
|--------------------------------------------------------------------------
| Advanced Posts API Routes
|--------------------------------------------------------------------------
*/

Route::prefix('posts')->name('api.posts.')->group(function () {
    // Bulk operations via API
    Route::post('bulk-update', [PostApiController::class, 'bulkUpdate'])->name('bulk-update');
    Route::post('bulk-delete', [PostApiController::class, 'bulkDelete'])->name('bulk-delete');
    
    // Advanced filtering dan sorting
    Route::get('filter', [PostApiController::class, 'filter'])->name('filter');
    Route::get('search-advanced', [PostApiController::class, 'advancedSearch'])->name('search-advanced');
    
    // Post statistics dan analytics
    Route::get('stats', [PostApiController::class, 'statistics'])->name('statistics');
    Route::get('analytics', [PostApiController::class, 'analytics'])->name('analytics');
    Route::get('{post}/analytics', [PostApiController::class, 'postAnalytics'])->name('post-analytics');
    
    // Related posts suggestions
    Route::get('{post}/related', [PostApiController::class, 'related'])->name('related');
    
    // Post revisions (if implemented)
    Route::get('{post}/revisions', [PostApiController::class, 'revisions'])->name('revisions');
});

/*
|--------------------------------------------------------------------------
| Admin Analytics API Routes
|--------------------------------------------------------------------------
*/

Route::prefix('admin')->name('api.admin.')->group(function () {
    // Dashboard statistics
    Route::get('dashboard-stats', function () {
        return response()->json([
            'posts' => [
                'total' => \App\Models\Post::count(),
                'published' => \App\Models\Post::published()->count(),
                'draft' => \App\Models\Post::where('status', 'draft')->count(),
                'archived' => \App\Models\Post::where('status', 'archived')->count(),
            ],
            'recent_activity' => [
                'posts_this_week' => \App\Models\Post::where('created_at', '>=', now()->subWeek())->count(),
                'posts_this_month' => \App\Models\Post::where('created_at', '>=', now()->subMonth())->count(),
            ],
            'top_categories' => \App\Models\Category::withCount('posts')
                                                  ->orderBy('posts_count', 'desc')
                                                  ->take(5)
                                                  ->get(['id', 'name', 'posts_count']),
            'popular_tags' => \App\Models\Tag::withCount('posts')
                                             ->orderBy('posts_count', 'desc')
                                             ->take(10)
                                             ->get(['id', 'name', 'posts_count']),
            'generated_at' => now()->toISOString(),
        ]);
    })->name('dashboard-stats');
    
    // Content performance metrics
    Route::get('content-metrics', function () {
        return response()->json([
            'most_viewed_posts' => \App\Models\Post::published()
                                                  ->orderBy('views_count', 'desc')
                                                  ->take(10)
                                                  ->get(['id', 'title', 'views_count', 'published_at']),
            'most_commented_posts' => \App\Models\Post::published()
                                                     ->withCount('comments')
                                                     ->orderBy('comments_count', 'desc')
                                                     ->take(10)
                                                     ->get(['id', 'title', 'comments_count']),
            'recent_comments' => \App\Models\Comment::with(['user:id,name', 'post:id,title'])
                                                   ->latest()
                                                   ->take(5)
                                                   ->get(),
            'generated_at' => now()->toISOString(),
        ]);
    })->name('content-metrics');
});

Route::get('/tags', fn() => \App\Models\Tag::active()->orderBy('name')->get());

/*
|--------------------------------------------------------------------------
| Protected Posts Management (publish/unpublish)
|--------------------------------------------------------------------------
*/
Route::middleware(['auth:sanctum','api.role:editor,admin'])->prefix('posts')->group(function () {
    Route::post('{post}/publish', [PostApiController::class, 'publish'])->name('api.posts.publish');
    Route::post('{post}/unpublish', [PostApiController::class, 'unpublish'])->name('api.posts.unpublish');
    Route::post('{id}/restore', [PostApiController::class, 'restore'])->name('api.posts.restore');
    Route::delete('{id}/force-delete', [PostApiController::class, 'forceDelete'])->name('api.posts.force-delete');
});

/*
|--------------------------------------------------------------------------
| Week 6: Authentication & User Management (Sanctum)
|--------------------------------------------------------------------------
| Menambahkan endpoints baru tanpa menghapus yang sudah ada.
*/

// Authentication routes
Route::prefix('auth')->name('api.auth.')->group(function () {
    Route::post('register', [AuthController::class, 'register'])->name('register');
    Route::post('login', [AuthController::class, 'login'])->name('login');

    Route::middleware('auth:sanctum')->group(function () {
        Route::get('me', [AuthController::class, 'me'])->name('me');
        Route::post('logout', [AuthController::class, 'logout'])->name('logout');
        Route::post('logout-all', [AuthController::class, 'logoutAll'])->name('logout-all');
        Route::get('sessions', [AuthController::class, 'sessions'])->name('sessions');
        Route::delete('sessions/{tokenId}', [AuthController::class, 'revokeSession'])->name('revoke-session');
    });
});

// Protected user management routes
Route::middleware(['auth:sanctum'])->prefix('users')->name('api.users.')->group(function () {
    Route::get('/', [UserController::class, 'index'])->name('index');
    Route::get('{user}', [UserController::class, 'show'])->name('show');

    // Self profile
    Route::put('profile', [UserController::class, 'updateProfile'])->name('update-profile');
    Route::post('profile/avatar', [UserController::class, 'uploadAvatar'])->name('upload-avatar');
    Route::delete('profile/avatar', [UserController::class, 'deleteAvatar'])->name('delete-avatar');

    // Admin-only
    Route::middleware('api.role:admin')->group(function () {
        Route::post('/', [UserController::class, 'store'])->name('store');
        Route::put('{user}', [UserController::class, 'update'])->name('update');
        Route::delete('{user}', [UserController::class, 'destroy'])->name('destroy');
        Route::post('{user}/activate', [UserController::class, 'activate'])->name('activate');
        Route::post('{user}/deactivate', [UserController::class, 'deactivate'])->name('deactivate');
        Route::get('stats/summary', [UserController::class, 'statistics'])->name('stats');
    });
});

// Development-only test routes
if (app()->environment(['local', 'testing'])) {
    Route::prefix('test')->name('api.test.')->group(function () {
        Route::get('auth', function (Request $request) {
            return response()->json([
                'authenticated' => (bool) $request->user(),
                'user' => $request->user()?->only(['id', 'name', 'email', 'role']),
                'token_info' => $request->user()?->getCurrentTokenInfo(),
                'timestamp' => now()->toISOString(),
            ]);
        })->middleware('auth:sanctum')->name('auth');

        Route::get('rate-limit', function () {
            return response()->json([
                'message' => 'Rate limit test successful',
                'timestamp' => now()->toISOString(),
            ]);
        })->middleware('api.rate:5,1')->name('rate-limit');
    });
}

/*
|--------------------------------------------------------------------------
| Public Content Routes (No Authentication)
|--------------------------------------------------------------------------
*/
Route::prefix('public')->name('api.public.')->group(function () {
    Route::get('posts', [PostApiController::class, 'publicIndex'])->name('posts.index');
    Route::get('posts/{post:slug}', [PostApiController::class, 'publicShow'])->name('posts.show');
    Route::get('categories', [CategoryApiController::class, 'publicIndex'])->name('categories.index');
    Route::get('search', [PostApiController::class, 'publicSearch'])->name('search');

    Route::get('stats', function () {
        return response()->json([
            'success' => true,
            'data' => [
                'published_posts' => \App\Models\Post::published()->count(),
                'active_categories' => \App\Models\Category::active()->count(),
                'total_authors' => \App\Models\User::whereIn('role', ['author', 'editor', 'admin'])->count(),
                'recent_posts' => \App\Models\Post::published()->latest('published_at')->take(5)->get(['id','title','slug','published_at']),
            ],
            'generated_at' => now()->toISOString(),
        ]);
    })->name('stats');
});
```

#### **Bagian 4: Enhanced API Controllers dan Security Features (20 menit)**

##### Step 4.1: Create Comprehensive UserController

**Isi file `app/Http/Controllers/Api/UserController.php`:**
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Validator;

class UserController extends Controller
{
    /**
     * Display paginated list of users dengan filtering
     */
    public function index(Request $request)
    {
        $query = User::query();

        // Filtering berdasarkan role
        if ($request->filled('role')) {
            $query->where('role', $request->role);
        }

        // Filtering berdasarkan status
        if ($request->filled('active')) {
            $query->where('is_active', $request->boolean('active'));
        }

        // Search by name atau email
        if ($request->filled('search')) {
            $search = $request->search;
            $query->where(function ($q) use ($search) {
                $q->where('name', 'LIKE', "%{$search}%")
                  ->orWhere('email', 'LIKE', "%{$search}%");
            });
        }

        // Include post dan comment counts
        $query->withCount(['posts', 'comments']);

        // Sorting
        $sortBy = $request->get('sort', 'created_at');
        $sortOrder = $request->get('order', 'desc');
        $query->orderBy($sortBy, $sortOrder);

        // Pagination
        $perPage = min($request->get('per_page', 15), 50);
        $users = $query->paginate($perPage);

        // Transform data untuk response
        $users->getCollection()->transform(function ($user) {
            return [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'role' => $user->role,
                'bio' => $user->bio,
                'avatar_url' => $user->avatar_url,
                'is_active' => $user->is_active,
                'email_verified_at' => $user->email_verified_at,
                'posts_count' => $user->posts_count,
                'comments_count' => $user->comments_count,
                'created_at' => $user->created_at,
                'updated_at' => $user->updated_at,
            ];
        });

        return response()->json([
            'success' => true,
            'data' => $users->items(),
            'pagination' => [
                'current_page' => $users->currentPage(),
                'last_page' => $users->lastPage(),
                'per_page' => $users->perPage(),
                'total' => $users->total(),
                'has_more' => $users->hasMorePages(),
            ],
            'filters' => [
                'role' => $request->role,
                'active' => $request->active,
                'search' => $request->search,
                'sort' => $sortBy,
                'order' => $sortOrder,
            ]
        ]);
    }

    /**
     * Display single user dengan detailed information
     */
    public function show(User $user)
    {
        // Load relationships
        $user->load(['posts' => function ($query) {
            $query->latest()->take(5);
        }, 'comments' => function ($query) {
            $query->latest()->take(5);
        }]);

        $userData = [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'role' => $user->role,
            'bio' => $user->bio,
            'avatar_url' => $user->avatar_url,
            'is_active' => $user->is_active,
            'email_verified_at' => $user->email_verified_at,
            'created_at' => $user->created_at,
            'updated_at' => $user->updated_at,
            'statistics' => [
                'total_posts' => $user->posts()->count(),
                'published_posts' => $user->posts()->where('status', 'published')->count(),
                'total_comments' => $user->comments()->count(),
                'join_date' => $user->created_at->format('M Y'),
            ],
            'recent_posts' => $user->posts->map(function ($post) {
                return [
                    'id' => $post->id,
                    'title' => $post->title,
                    'slug' => $post->slug,
                    'status' => $post->status,
                    'published_at' => $post->published_at,
                ];
            }),
            'recent_comments' => $user->comments->map(function ($comment) {
                return [
                    'id' => $comment->id,
                    'content' => \Str::limit($comment->content, 100),
                    'post_title' => $comment->post->title ?? 'Unknown',
                    'created_at' => $comment->created_at,
                ];
            })
        ];

        return response()->json([
            'success' => true,
            'data' => $userData
        ]);
    }

    /**
     * Create new user (Admin only)
     */
    public function store(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
            'role' => 'required|in:subscriber,author,editor,admin',
            'bio' => 'nullable|string|max:1000',
            'is_active' => 'boolean',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors()
            ], 422);
        }

        try {
            $user = User::create([
                'name' => $request->name,
                'email' => $request->email,
                'password' => Hash::make($request->password),
                'role' => $request->role,
                'bio' => $request->bio,
                'is_active' => $request->boolean('is_active', true),
                'email_verified_at' => now(), // Admin-created users are auto-verified
            ]);

            \Log::info('User created by admin', [
                'created_user_id' => $user->id,
                'created_by' => $request->user()->id,
                'timestamp' => now()
            ]);

            return response()->json([
                'success' => true,
                'message' => 'User created successfully',
                'data' => [
                    'id' => $user->id,
                    'name' => $user->name,
                    'email' => $user->email,
                    'role' => $user->role,
                    'is_active' => $user->is_active,
                    'created_at' => $user->created_at,
                ]
            ], 201);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to create user',
            ], 500);
        }
    }

    /**
     * Update user profile (own profile atau admin)
     */
    public function updateProfile(Request $request)
    {
        $user = $request->user();

        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'bio' => 'nullable|string|max:1000',
            'email' => 'required|string|email|max:255|unique:users,email,' . $user->id,
            'current_password' => 'nullable|string',
            'password' => 'nullable|string|min:8|confirmed',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors()
            ], 422);
        }

        // Verify current password jika ingin change password
        if ($request->filled('password')) {
            if (!$request->filled('current_password') || 
                !Hash::check($request->current_password, $user->password)) {
                return response()->json([
                    'success' => false,
                    'message' => 'Current password is incorrect',
                ], 422);
            }
        }

        try {
            $updateData = [
                'name' => $request->name,
                'bio' => $request->bio,
                'email' => $request->email,
            ];

            if ($request->filled('password')) {
                $updateData['password'] = Hash::make($request->password);
            }

            $user->update($updateData);

            return response()->json([
                'success' => true,
                'message' => 'Profile updated successfully',
                'data' => [
                    'id' => $user->id,
                    'name' => $user->name,
                    'email' => $user->email,
                    'bio' => $user->bio,
                    'avatar_url' => $user->avatar_url,
                    'updated_at' => $user->updated_at,
                ]
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to update profile',
            ], 500);
        }
    }

    /**
     * Upload user avatar
     */
    public function uploadAvatar(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'avatar' => 'required|image|mimes:jpeg,png,jpg,gif|max:2048',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Invalid avatar file',
                'errors' => $validator->errors()
            ], 422);
        }

        try {
            $user = $request->user();

            // Delete old avatar jika ada
            if ($user->avatar) {
                Storage::disk('public')->delete($user->avatar);
            }

            // Upload new avatar
            $file = $request->file('avatar');
            $filename = 'avatar_' . $user->id . '_' . time() . '.' . $file->getClientOriginalExtension();
            $path = $file->storeAs('avatars', $filename, 'public');

            // Update user record
            $user->update(['avatar' => $path]);

            return response()->json([
                'success' => true,
                'message' => 'Avatar uploaded successfully',
                'data' => [
                    'avatar_url' => $user->avatar_url,
                ]
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to upload avatar',
            ], 500);
        }
    }

    /**
     * Delete user avatar
     */
    public function deleteAvatar(Request $request)
    {
        try {
            $user = $request->user();

            if ($user->avatar) {
                Storage::disk('public')->delete($user->avatar);
                $user->update(['avatar' => null]);
            }

            return response()->json([
                'success' => true,
                'message' => 'Avatar deleted successfully',
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to delete avatar',
            ], 500);
        }
    }

    /**
     * Admin: Update any user
     */
    public function update(Request $request, User $user)
    {
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users,email,' . $user->id,
            'role' => 'required|in:subscriber,author,editor,admin',
            'bio' => 'nullable|string|max:1000',
            'is_active' => 'boolean',
            'password' => 'nullable|string|min:8|confirmed',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors()
            ], 422);
        }

        try {
            $updateData = $request->only(['name', 'email', 'role', 'bio', 'is_active']);

            if ($request->filled('password')) {
                $updateData['password'] = Hash::make($request->password);
            }

            $user->update($updateData);

            \Log::info('User updated by admin', [
                'updated_user_id' => $user->id,
                'updated_by' => $request->user()->id,
                'changes' => $user->getChanges(),
                'timestamp' => now()
            ]);

            return response()->json([
                'success' => true,
                'message' => 'User updated successfully',
                'data' => $user->only(['id', 'name', 'email', 'role', 'bio', 'is_active', 'updated_at'])
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to update user',
            ], 500);
        }
    }

    /**
     * Admin: Delete user
     */
    public function destroy(User $user)
    {
        try {
            // Prevent self-deletion
            if ($user->id === auth()->id()) {
                return response()->json([
                    'success' => false,
                    'message' => 'Cannot delete your own account',
                ], 422);
            }

            $userName = $user->name;
            $user->delete();

            return response()->json([
                'success' => true,
                'message' => "User '{$userName}' deleted successfully",
            ]);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to delete user',
            ], 500);
        }
    }

    /**
     * Admin: Activate user
     */
    public function activate(User $user)
    {
        $user->update(['is_active' => true]);

        return response()->json([
            'success' => true,
            'message' => "User '{$user->name}' activated successfully",
        ]);
    }

    /**
     * Admin: Deactivate user
     */
    public function deactivate(User $user)
    {
        // Prevent self-deactivation
        if ($user->id === auth()->id()) {
            return response()->json([
                'success' => false,
                'message' => 'Cannot deactivate your own account',
            ], 422);
        }

        $user->update(['is_active' => false]);

        // Revoke all tokens untuk deactivated user
        $user->revokeAllTokens();

        return response()->json([
            'success' => true,
            'message' => "User '{$user->name}' deactivated successfully",
        ]);
    }

    /**
     * Admin: Get user statistics
     */
    public function statistics()
    {
        $stats = [
            'total_users' => User::count(),
            'active_users' => User::where('is_active', true)->count(),
            'verified_users' => User::whereNotNull('email_verified_at')->count(),
            'by_role' => User::selectRaw('role, COUNT(*) as count')
                            ->groupBy('role')
                            ->pluck('count', 'role'),
            'recent_registrations' => [
                'today' => User::whereDate('created_at', today())->count(),
                'this_week' => User::where('created_at', '>=', now()->subWeek())->count(),
                'this_month' => User::where('created_at', '>=', now()->subMonth())->count(),
            ],
            'active_tokens' => \DB::table('personal_access_tokens')
                                 ->where('expires_at', '>', now())
                                 ->orWhereNull('expires_at')
                                 ->count(),
        ];

        return response()->json([
            'success' => true,
            'data' => $stats,
            'generated_at' => now()->toISOString(),
        ]);
    }
}
```

##### Step 4.2: Run Database Migration dan Test Setup
```bash
# Run migrations jika belum
php artisan migrate

# Test database structure
mysql -u laravel_user -p laravel_app -e "
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'laravel_app' 
AND TABLE_NAME IN ('users', 'personal_access_tokens');
"

# Generate sample data untuk testing
php artisan tinker --execute="
// Create test users with different roles
if (\App\Models\User::count() < 20) {
    \App\Models\User::create([
        'name' => 'Admin User',
        'email' => 'admin@test.com',
        'password' => \Hash::make('password123'),
        'role' => 'admin',
        'is_active' => true,
        'email_verified_at' => now(),
    ]);
    
    \App\Models\User::create([
        'name' => 'Editor User',
        'email' => 'editor@test.com',
        'password' => \Hash::make('password123'),
        'role' => 'editor',
        'is_active' => true,
        'email_verified_at' => now(),
    ]);
    
    \App\Models\User::create([
        'name' => 'Author User',
        'email' => 'author@test.com',
        'password' => \Hash::make('password123'),
        'role' => 'author',
        'is_active' => true,
        'email_verified_at' => now(),
    ]);
    
    echo 'Test users created successfully' . PHP_EOL;
} else {
    echo 'Test users already exist' . PHP_EOL;
}
"
```

##### Step 4.3: Enhanced PostApiController dengan Authorization

**Update method-method penting dalam `app/Http/Controllers/Api/PostApiController.php`:**
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Post;
use App\Models\Category;
use App\Models\Tag;
use Illuminate\Http\Request;

class PostApiController extends Controller
{
    /**
     * Display listing of posts untuk API
     */
    public function index(Request $request)
    {
        $query = Post::with(['user:id,name', 'category:id,name,color', 'tags:id,name,color'])
                    ->withCount('comments');

        // Filter by status (default: published untuk public API)
        $status = $request->get('status', 'published');
        if ($status === 'all') {
            // No filtering - show all posts
        } else {
            $query->where('status', $status);
        }

        // Search functionality
        if ($request->filled('search')) {
            $query->search($request->search);
        }

        // Pagination
        $perPage = min($request->get('per_page', 10), 50); // Max 50 per page
        $posts = $query->latest('published_at')->paginate($perPage);

        return response()->json([
            'status' => 'success',
            'data' => [
                'posts' => $posts->items(),
                'pagination' => [
                    'current_page' => $posts->currentPage(),
                    'last_page' => $posts->lastPage(),
                    'per_page' => $posts->perPage(),
                    'total' => $posts->total(),
                    'has_more' => $posts->hasMorePages(),
                ]
            ],
            'meta' => [
                'timestamp' => now()->toISOString(),
                'total_published' => Post::published()->count(),
            ]
        ]);
    }

    /**
     * Store new post via API
     */
    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'excerpt' => 'nullable|string|max:500',
            'status' => 'required|in:draft,published,archived',
            'category_id' => 'required|exists:categories,id',
            'user_id' => 'required|exists:users,id',
            'tags' => 'nullable|array',
            'tags.*' => 'exists:tags,id',
            'is_featured' => 'boolean',
            'allow_comments' => 'boolean',
        ]);

        // Generate slug
        $validated['slug'] = \Str::slug($validated['title']);
        
        // Set published_at if published
        if ($validated['status'] === 'published') {
            $validated['published_at'] = now();
        }

        $post = Post::create($validated);

        // Attach tags
        if (!empty($validated['tags'])) {
            $post->tags()->attach($validated['tags']);
        }

        // Load relationships untuk response
        $post->load(['user:id,name', 'category:id,name', 'tags:id,name']);

        return response()->json([
            'status' => 'success',
            'message' => 'Post created successfully',
            'data' => $post
        ], 201);
    }

    /**
     * Display single post
     */
    public function show(Post $post)
    {
        $post->load(['user:id,name,email', 'category:id,name,color', 'tags:id,name,color']);
        $post->loadCount('comments');

        return response()->json([
            'status' => 'success',
            'data' => $post
        ]);
    }

    /**
     * Update post via API
     */
    public function update(Request $request, Post $post)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'excerpt' => 'nullable|string|max:500',
            'status' => 'required|in:draft,published,archived',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'nullable|array',
            'tags.*' => 'exists:tags,id',
            'is_featured' => 'boolean',
            'allow_comments' => 'boolean',
        ]);

        // Update slug if title changed
        if ($post->title !== $validated['title']) {
            $validated['slug'] = \Str::slug($validated['title']);
        }

        // Set published_at if status changed to published
        if ($validated['status'] === 'published' && $post->status !== 'published') {
            $validated['published_at'] = now();
        }

        $post->update($validated);

        // Sync tags
        if (isset($validated['tags'])) {
            $post->tags()->sync($validated['tags']);
        }

        $post->load(['user:id,name', 'category:id,name', 'tags:id,name']);

        return response()->json([
            'status' => 'success',
            'message' => 'Post updated successfully',
            'data' => $post
        ]);
    }

    /**
     * Delete post via API
     */
    public function destroy(Post $post)
    {
        $title = $post->title;
        $post->delete();

        return response()->json([
            'status' => 'success',
            'message' => "Post '{$title}' deleted successfully"
        ]);
    }

    /**
     * Search posts by keyword
     */
    public function search($keyword)
    {
        $posts = Post::published()
                    ->search($keyword)
                    ->with(['user:id,name', 'category:id,name,color'])
                    ->withCount('comments')
                    ->latest('published_at')
                    ->take(20)
                    ->get();

        return response()->json([
            'status' => 'success',
            'data' => $posts,
            'meta' => [
                'keyword' => $keyword,
                'count' => $posts->count(),
                'timestamp' => now()->toISOString(),
            ]
        ]);
    }

    /**
     * Get posts by category
     */
    public function byCategory(Category $category)
    {
        $posts = Post::published()
                    ->where('category_id', $category->id)
                    ->with(['user:id,name', 'category:id,name,color'])
                    ->withCount('comments')
                    ->latest('published_at')
                    ->paginate(10);

        return response()->json([
            'status' => 'success',
            'data' => [
                'category' => $category,
                'posts' => $posts->items(),
                'pagination' => [
                    'current_page' => $posts->currentPage(),
                    'last_page' => $posts->lastPage(),
                    'total' => $posts->total(),
                ]
            ]
        ]);
    }

    /**
     * Get featured posts
     */
    public function featured()
    {
        $posts = Post::published()
                    ->featured()
                    ->with(['user:id,name', 'category:id,name,color'])
                    ->withCount('comments')
                    ->latest('published_at')
                    ->take(5)
                    ->get();

        return response()->json([
            'status' => 'success',
            'data' => $posts,
            'meta' => [
                'count' => $posts->count(),
                'timestamp' => now()->toISOString(),
            ]
        ]);
    }

    /**
     * Get posts by tag
     */
    public function byTag(Tag $tag)
    {
        $posts = Post::published()
            ->whereHas('tags', fn($q) => $q->where('tags.id', $tag->id))
            ->with(['user:id,name', 'category:id,name,color'])
            ->withCount('comments')
            ->latest('published_at')
            ->paginate(10);

        return response()->json([
            'status' => 'success',
            'data' => [
                'tag' => $tag,
                'posts' => $posts->items(),
                'pagination' => [
                    'current_page' => $posts->currentPage(),
                    'last_page' => $posts->lastPage(),
                    'total' => $posts->total(),
                ],
            ],
        ]);
    }

    /**
     * Published posts list
     */
    public function published(Request $request)
    {
        $perPage = min((int) $request->get('per_page', 10), 50);
        $posts = Post::published()
            ->with(['user:id,name', 'category:id,name,color'])
            ->withCount('comments')
            ->latest('published_at')
            ->paginate($perPage);

        return response()->json([
            'status' => 'success',
            'data' => $posts->items(),
            'pagination' => [
                'current_page' => $posts->currentPage(),
                'last_page' => $posts->lastPage(),
                'per_page' => $posts->perPage(),
                'total' => $posts->total(),
            ],
        ]);
    }

    /**
     * Advanced filter
     */
    public function filter(Request $request)
    {
        $q = Post::query()->with(['user:id,name', 'category:id,name,color', 'tags:id,name'])
            ->withCount('comments');

        if ($request->filled('status') && in_array($request->status, ['draft','published','archived'], true)) {
            $q->where('status', $request->status);
        }
        if ($request->filled('category_id')) {
            $q->where('category_id', $request->integer('category_id'));
        }
        if ($request->filled('tag_id')) {
            $q->whereHas('tags', fn($x) => $x->where('tags.id', (int) $request->tag_id));
        }
        if ($request->filled('featured')) {
            $q->where('is_featured', filter_var($request->featured, FILTER_VALIDATE_BOOLEAN));
        }
        if ($request->filled('search')) {
            $q->search($request->search);
        }

        $sort = $request->get('sort', 'published_at');
        $order = $request->get('order', 'desc');
        $q->orderBy($sort, $order);

        $perPage = min((int) $request->get('per_page', 10), 100);
        $posts = $q->paginate($perPage);

        return response()->json([
            'status' => 'success',
            'data' => $posts->items(),
            'pagination' => [
                'current_page' => $posts->currentPage(),
                'last_page' => $posts->lastPage(),
                'per_page' => $posts->perPage(),
                'total' => $posts->total(),
            ],
        ]);
    }

    /**
     * Advanced search
     */
    public function advancedSearch(Request $request)
    {
        $q = Post::published()->with(['user:id,name', 'category:id,name,color'])
            ->withCount('comments');

        $q->search($request->get('q'));
        if ($request->filled('category_id')) {
            $q->where('category_id', $request->integer('category_id'));
        }
        if ($request->filled('tag_id')) {
            $q->whereHas('tags', fn($x) => $x->where('tags.id', (int) $request->tag_id));
        }

        $posts = $q->latest('published_at')->paginate(min((int) $request->get('per_page', 10), 50));
        return response()->json([
            'status' => 'success',
            'data' => $posts->items(),
            'pagination' => [
                'current_page' => $posts->currentPage(),
                'last_page' => $posts->lastPage(),
                'per_page' => $posts->perPage(),
                'total' => $posts->total(),
            ],
        ]);
    }

    /**
     * Statistics summary
     */
    public function statistics()
    {
        return response()->json([
            'status' => 'success',
            'data' => [
                'total' => Post::count(),
                'published' => Post::published()->count(),
                'draft' => Post::where('status', 'draft')->count(),
                'archived' => Post::where('status', 'archived')->count(),
                'this_month' => Post::whereMonth('created_at', now()->month)->count(),
            ],
            'generated_at' => now()->toISOString(),
        ]);
    }

    /**
     * Global analytics
     */
    public function analytics()
    {
        $topViewed = Post::published()->orderByDesc('views_count')->take(10)->get(['id','title','views_count']);
        $mostCommented = Post::published()->withCount('comments')->orderByDesc('comments_count')->take(10)->get(['id','title']);

        return response()->json([
            'status' => 'success',
            'data' => [
                'top_viewed' => $topViewed,
                'most_commented' => $mostCommented,
                'recent_published' => Post::published()->latest('published_at')->take(10)->get(['id','title','published_at']),
            ],
        ]);
    }

    /**
     * Per-post analytics
     */
    public function postAnalytics(Post $post)
    {
        return response()->json([
            'status' => 'success',
            'data' => [
                'id' => $post->id,
                'title' => $post->title,
                'views' => (int) ($post->views_count ?? 0),
                'comments' => $post->comments()->count(),
                'published_at' => $post->published_at,
                'is_published' => $post->isPublished(),
            ],
        ]);
    }

    /**
     * Related posts
     */
    public function related(Post $post)
    {
        $related = Post::published()
            ->where('id', '!=', $post->id)
            ->where(function($q) use ($post) {
                $q->where('category_id', $post->category_id)
                  ->orWhereHas('tags', function($t) use ($post) {
                      $t->whereIn('tags.id', $post->tags()->pluck('tags.id'));
                  });
            })
            ->latest('published_at')
            ->take(5)
            ->get(['id','title','slug','published_at','category_id']);

        return response()->json([
            'status' => 'success',
            'data' => $related,
        ]);
    }

    /**
     * Revisions placeholder (not implemented)
     */
    public function revisions(Post $post)
    {
        return response()->json([
            'status' => 'success',
            'message' => 'Revisions feature not implemented',
            'data' => [],
        ]);
    }

    /**
     * Bulk update certain fields
     */
    public function bulkUpdate(Request $request)
    {
        $validated = $request->validate([
            'ids' => 'required|array|min:1',
            'ids.*' => 'integer|exists:posts,id',
            'status' => 'nullable|in:draft,published,archived',
            'category_id' => 'nullable|exists:categories,id',
            'is_featured' => 'nullable|boolean',
            'allow_comments' => 'nullable|boolean',
        ]);

        $data = collect($validated)->only(['status','category_id','is_featured','allow_comments'])
            ->filter(fn($v) => !is_null($v))->all();

        if (empty($data)) {
            return response()->json(['status' => 'success', 'updated' => 0, 'message' => 'No fields to update']);
        }

        $updated = Post::whereIn('id', $validated['ids'])->update($data);
        return response()->json(['status' => 'success', 'updated' => $updated]);
    }

    /**
     * Bulk delete (soft delete)
     */
    public function bulkDelete(Request $request)
    {
        $validated = $request->validate([
            'ids' => 'required|array|min:1',
            'ids.*' => 'integer|exists:posts,id',
        ]);

        $deleted = Post::whereIn('id', $validated['ids'])->delete();
        return response()->json(['status' => 'success', 'deleted' => $deleted]);
    }

    /**
     * Publish a post
     */
    public function publish(Request $request, Post $post)
    {
        $user = $request->user();
        if ($user && !$user->hasApiAbility('posts:update')) {
            return response()->json(['success' => false, 'message' => 'Insufficient permissions to publish'], 403);
        }

        if ($post->status !== 'published') {
            $post->status = 'published';
            $post->published_at = now();
            $post->save();
        }

        return response()->json(['status' => 'success', 'message' => 'Post published', 'data' => $post]);
    }

    /**
     * Unpublish a post (back to draft)
     */
    public function unpublish(Request $request, Post $post)
    {
        $user = $request->user();
        if ($user && !$user->hasApiAbility('posts:update')) {
            return response()->json(['success' => false, 'message' => 'Insufficient permissions to unpublish'], 403);
        }

        $post->status = 'draft';
        $post->save();
        return response()->json(['status' => 'success', 'message' => 'Post unpublished', 'data' => $post]);
    }

    /**
     * Restore soft-deleted post
     */
    public function restore($id)
    {
        $post = Post::withTrashed()->findOrFail($id);
        $post->restore();
        return response()->json(['status' => 'success', 'message' => 'Post restored']);
    }

    /**
     * Force delete post
     */
    public function forceDelete($id)
    {
        $post = Post::withTrashed()->findOrFail($id);
        $post->forceDelete();
        return response()->json(['status' => 'success', 'message' => 'Post permanently deleted']);
    }

    /*
    |--------------------------------------------------------------------------
    | Public endpoints (no auth)
    |--------------------------------------------------------------------------
    */
    public function publicIndex(Request $request)
    {
        $perPage = min((int) $request->get('per_page', 10), 50);
        $posts = Post::published()
            ->with(['user:id,name', 'category:id,name,slug,color'])
            ->latest('published_at')
            ->paginate($perPage, ['id','title','slug','excerpt','published_at','user_id','category_id']);

        return response()->json([
            'success' => true,
            'data' => $posts->items(),
            'pagination' => [
                'current_page' => $posts->currentPage(),
                'last_page' => $posts->lastPage(),
                'per_page' => $posts->perPage(),
                'total' => $posts->total(),
            ],
        ]);
    }

    public function publicShow(Post $post)
    {
        if (!$post->isPublished()) {
            return response()->json(['success' => false, 'message' => 'Post not found'], 404);
        }
        $post->load(['user:id,name', 'category:id,name,slug', 'tags:id,name,slug']);
        $post->incrementViews();
        return response()->json(['success' => true, 'data' => $post]);
    }

    public function publicSearch(Request $request)
    {
        $q = trim((string) $request->get('q', ''));
        $posts = Post::published()->search($q)
            ->with(['user:id,name', 'category:id,name,slug'])
            ->latest('published_at')
            ->paginate(min((int) $request->get('per_page', 10), 50));

        return response()->json([
            'success' => true,
            'data' => $posts->items(),
            'pagination' => [
                'current_page' => $posts->currentPage(),
                'last_page' => $posts->lastPage(),
                'per_page' => $posts->perPage(),
                'total' => $posts->total(),
            ],
            'meta' => [
                'q' => $q,
            ],
        ]);
    }
}
```

#### **Bagian 5: API Testing & Documentation (15 menit)**

##### Step 5.1: Buat Script Testing (PowerShell)

Buat file `tests/api/test-api.ps1`:
```powershell
Param(
    [string]$BaseUrl = "http://localhost:8000/api"
)

Write-Host "=== API Testing - Week 6 (PowerShell) ===" -ForegroundColor Cyan
Write-Host "Base URL: $BaseUrl"

function Invoke-ApiRequest {
    Param(
        [Parameter(Mandatory=$true)][ValidateSet('GET','POST','PUT','DELETE')][string]$Method,
        [Parameter(Mandatory=$true)][string]$Path,
        [hashtable]$Body,
        [string]$Token
    )
    $headers = @{ 'Accept' = 'application/json' }
    if ($Token) { $headers['Authorization'] = "Bearer $Token" }
    $uri = "$BaseUrl/" + $Path.TrimStart('/')
    try {
        if ($null -ne $Body) {
            $json = $Body | ConvertTo-Json -Depth 8
            $resp = Invoke-WebRequest -Method $Method -Uri $uri -Headers $headers -ContentType 'application/json' -Body $json -ErrorAction Stop
        } else {
            $resp = Invoke-WebRequest -Method $Method -Uri $uri -Headers $headers -ErrorAction Stop
        }
    } catch {
        $resp = $_.Exception.Response
    }
    return $resp
}

function Get-Status([object]$Response) {
    try { return [int]$Response.StatusCode } catch { return -1 }
}

function Parse-Json([object]$Response) {
    if ($null -eq $Response) { return $null }
    $content = $null
    try { $content = $Response.Content } catch { $content = $null }
    if ([string]::IsNullOrWhiteSpace($content)) { return $null }
    try { return ($content | ConvertFrom-Json) } catch { return $null }
}

# 1. Public info
Write-Host "1) API Info (public)" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'info'
Write-Host ("  Status: {0}" -f (Get-Status $r))
$info = Parse-Json $r
if ($info) { Write-Host ("  Laravel: {0}, API: {1}" -f $info.laravel_version, $info.api_version) }

# 2. Register
Write-Host "2) Register new user" -ForegroundColor Yellow
$rand = Get-Random
$email = "testapi+$rand@example.com"
$r = Invoke-ApiRequest -Method POST -Path 'auth/register' -Body @{ name='Test API'; email=$email; password='Password123!'; password_confirmation='Password123!' }
Write-Host ("  Status: {0} (email: {1})" -f (Get-Status $r), $email)
$reg = Parse-Json $r

# 3. Login
Write-Host "3) Login" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method POST -Path 'auth/login' -Body @{ email=$email; password='Password123!'; device_name='ps-test' }
Write-Host ("  Status: {0}" -f (Get-Status $r))
$login = Parse-Json $r
$token = $null
if ($login -and $login.data -and $login.data.token -and $login.data.token.access_token) {
    $token = $login.data.token.access_token
}
if (-not $token) { Write-Host "  Failed to get token" -ForegroundColor Red; exit 1 }
Write-Host "  Token acquired"

# 4. Authenticated me
Write-Host "4) Authenticated /auth/me" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'auth/me' -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))
$me = Parse-Json $r
Write-Host ("  User: {0} [{1}] (role: {2})" -f $me.data.name, $me.data.email, $me.data.role)

# 5. Users index (authenticated)
Write-Host "5) GET /users (auth)" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'users' -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

# 6. Rate limit test (dev route)
Write-Host "6) Rate limit test (dev)" -ForegroundColor Yellow
for ($i = 1; $i -le 6; $i++) {
    $r = Invoke-ApiRequest -Method GET -Path 'test/rate-limit'
    $code = Get-Status $r
    Write-Host ("  Attempt {0}: {1}" -f $i, $code)
}

# 7. Sessions
Write-Host "7) Sessions list + revoke one" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'auth/sessions' -Token $token
$parsedSessions = Parse-Json $r
$sessions = $null
if ($parsedSessions -and $parsedSessions.data -and $parsedSessions.data.active_sessions) {
    $sessions = $parsedSessions.data.active_sessions
}
$sessionsCount = 0
if ($sessions) { try { $sessionsCount = [int]$sessions.Count } catch { $sessionsCount = 0 } }
Write-Host ("  Sessions: {0}" -f $sessionsCount)
if ($sessions -and $sessions.Count -gt 1) {
    $toRevoke = $sessions[0].id
    $r = Invoke-ApiRequest -Method DELETE -Path ("auth/sessions/{0}" -f $toRevoke) -Token $token
    Write-Host ("  Revoke status: {0}" -f (Get-Status $r))
}

# 8. Public endpoints quick check
Write-Host "8) Public endpoints" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'public/posts'
Write-Host ("  /public/posts -> {0}" -f (Get-Status $r))
$r = Invoke-ApiRequest -Method GET -Path 'public/categories'
Write-Host ("  /public/categories -> {0}" -f (Get-Status $r))
$r = Invoke-ApiRequest -Method GET -Path 'public/search?q=laravel'
Write-Host ("  /public/search -> {0}" -f (Get-Status $r))

# 9. Logout
Write-Host "9) Logout" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method POST -Path 'auth/logout' -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

Write-Host "=== Done ===" -ForegroundColor Cyan
```

Buat file `tests/api/test-admin-flow.ps1`:
```powershell
Param(
    [string]$BaseUrl = "http://localhost:8000/api"
)

Write-Host "=== API Admin/Editor Flow Test (PowerShell) ===" -ForegroundColor Cyan
Write-Host "Base URL: $BaseUrl"

function Invoke-ApiRequest {
    Param(
        [Parameter(Mandatory=$true)][ValidateSet('GET','POST','PUT','DELETE')][string]$Method,
        [Parameter(Mandatory=$true)][string]$Path,
        [hashtable]$Body,
        [string]$Token
    )
    $headers = @{ 'Accept' = 'application/json' }
    if ($Token) { $headers['Authorization'] = "Bearer $Token" }
    $uri = "$BaseUrl/" + $Path.TrimStart('/')
    try {
        if ($null -ne $Body) {
            $json = $Body | ConvertTo-Json -Depth 8
            $resp = Invoke-WebRequest -Method $Method -Uri $uri -Headers $headers -ContentType 'application/json' -Body $json -ErrorAction Stop
        } else {
            $resp = Invoke-WebRequest -Method $Method -Uri $uri -Headers $headers -ErrorAction Stop
        }
    } catch {
        $resp = $_.Exception.Response
    }
    return $resp
}

function Get-Status([object]$Response) {
    try { return [int]$Response.StatusCode } catch { return -1 }
}

function Parse-Json([object]$Response) {
    if ($null -eq $Response) { return $null }
    $content = $null
    try { $content = $Response.Content } catch { $content = $null }
    if ([string]::IsNullOrWhiteSpace($content)) { return $null }
    try { return ($content | ConvertFrom-Json) } catch { return $null }
}

# 1) Register EDITOR user
Write-Host "1) Register editor user" -ForegroundColor Yellow
$rand = Get-Random
$email = "editor+$rand@example.com"
$regBody = @{ name='Editor User'; email=$email; password='Password123!'; password_confirmation='Password123!'; role='editor' }
$r = Invoke-ApiRequest -Method POST -Path 'auth/register' -Body $regBody
Write-Host ("  Status: {0} (email: {1})" -f (Get-Status $r), $email)

# 2) Login as editor
Write-Host "2) Login as editor" -ForegroundColor Yellow
$loginBody = @{ email=$email; password='Password123!'; device_name='ps-admin-flow' }
$r = Invoke-ApiRequest -Method POST -Path 'auth/login' -Body $loginBody
Write-Host ("  Status: {0}" -f (Get-Status $r))
$login = Parse-Json $r
$token = $null
if ($login -and $login.data -and $login.data.token -and $login.data.token.access_token) { $token = $login.data.token.access_token }
if (-not $token) { Write-Host "  Failed to get token" -ForegroundColor Red; exit 1 }
Write-Host "  Token acquired"

# 3) Get editor user profile (to read user_id)
Write-Host "3) Get editor profile" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method GET -Path 'auth/me' -Token $token
$me = Parse-Json $r
$userId = $me.data.id
Write-Host ("  UserId: {0}" -f $userId)

# 4) Create category (uses existing categories resource)
Write-Host "4) Create category" -ForegroundColor Yellow
$catName = "Category $rand"
$catBody = @{ name=$catName; description='Admin flow test'; color='#3B82F6'; is_active=$true; sort_order=0 }
$r = Invoke-ApiRequest -Method POST -Path 'categories' -Body $catBody -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))
$cat = Parse-Json $r
$categoryId = $null
if ($cat -and $cat.data -and $cat.data.id) { $categoryId = $cat.data.id }
elseif ($cat -and $cat.id) { $categoryId = $cat.id }
Write-Host ("  CategoryId: {0}" -f $categoryId)

# 5) Create draft post
Write-Host "5) Create draft post" -ForegroundColor Yellow
$title = "Post $rand"
$postBody = @{ title=$title; content='Draft content'; excerpt='Excerpt'; status='draft'; category_id=$categoryId; user_id=$userId; is_featured=$false; allow_comments=$true }
$r = Invoke-ApiRequest -Method POST -Path 'posts' -Body $postBody -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))
$post = Parse-Json $r
$postId = $null
if ($post -and $post.data -and $post.data.id) { $postId = $post.data.id }
elseif ($post -and $post.id) { $postId = $post.id }
Write-Host ("  PostId: {0}" -f $postId)

# 6) Publish
Write-Host "6) Publish post" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method POST -Path ("posts/{0}/publish" -f $postId) -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

# 7) Unpublish
Write-Host "7) Unpublish post" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method POST -Path ("posts/{0}/unpublish" -f $postId) -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

# 8) Delete (soft)
Write-Host "8) Delete post (soft)" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method DELETE -Path ("posts/{0}" -f $postId) -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

# 9) Restore
Write-Host "9) Restore post" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method POST -Path ("posts/{0}/restore" -f $postId) -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

# 10) Force delete
Write-Host "10) Force delete post" -ForegroundColor Yellow
$r = Invoke-ApiRequest -Method DELETE -Path ("posts/{0}/force-delete" -f $postId) -Token $token
Write-Host ("  Status: {0}" -f (Get-Status $r))

Write-Host "=== Admin/Editor flow completed ===" -ForegroundColor Cyan
```

##### Step 5.2: Gunakan Script PowerShell untuk Testing
```powershell
# Jalankan server lebih dulu (terminal terpisah)
php artisan serve --host=0.0.0.0 --port=8000

# Jalankan testing dasar (auth, sessions, public)
powershell -ExecutionPolicy Bypass -File .\tests\api\test-api.ps1

# Jalankan testing alur editor/admin (publish/unpublish/restore/force-delete)
powershell -ExecutionPolicy Bypass -File .\tests\api\test-admin-flow.ps1
```

##### Step 5.3: Create API Documentation

**Isi file `public/api-docs.html`:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API Documentation - Praktikum Cloud Computing ITK</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        .endpoint { @apply border-l-4 border-blue-500 pl-4 mb-4; }
        .method-get { @apply bg-green-100 text-green-800 px-2 py-1 rounded text-sm font-mono; }
        .method-post { @apply bg-blue-100 text-blue-800 px-2 py-1 rounded text-sm font-mono; }
        .method-put { @apply bg-yellow-100 text-yellow-800 px-2 py-1 rounded text-sm font-mono; }
        .method-delete { @apply bg-red-100 text-red-800 px-2 py-1 rounded text-sm font-mono; }
    </style>
</head>
<body class="bg-gray-50">
    <div class="max-w-6xl mx-auto p-6">
        <header class="mb-8">
            <h1 class="text-4xl font-bold text-gray-900 mb-2">API Documentation</h1>
            <p class="text-xl text-gray-600">Praktikum Cloud Computing ITK - Week 6</p>
            <p class="text-gray-500">Laravel Sanctum Authentication & Authorization API</p>
        </header>

        <!-- API Overview -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Overview</h2>
            <div class="grid md:grid-cols-2 gap-6">
                <div>
                    <h3 class="font-semibold mb-2">Base Information</h3>
                    <ul class="space-y-1 text-sm">
                        <li><strong>Base URL:</strong> <code class="bg-gray-100 px-2 py-1 rounded">http://localhost:8000/api</code></li>
                        <li><strong>Version:</strong> v1</li>
                        <li><strong>Authentication:</strong> Bearer Token (Laravel Sanctum)</li>
                        <li><strong>Response Format:</strong> JSON</li>
                    </ul>
                </div>
                <div>
                    <h3 class="font-semibold mb-2">Rate Limits</h3>
                    <ul class="space-y-1 text-sm">
                        <li><strong>Authenticated:</strong> 200 requests/minute</li>
                        <li><strong>Guest:</strong> 60 requests/minute</li>
                        <li><strong>Headers:</strong> X-RateLimit-Limit, X-RateLimit-Remaining</li>
                    </ul>
                </div>
            </div>
        </section>

        <!-- Authentication -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Authentication</h2>
            
            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/auth/register</code>
                </div>
                <p class="text-gray-600 mb-2">Register new user account</p>
                <details class="mb-4">
                    <summary class="cursor-pointer text-blue-600 hover:underline">Request Body</summary>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm overflow-x-auto"><code>{
  "name": "User Name",
  "email": "user@example.com",
  "password": "password123",
  "password_confirmation": "password123",
  "role": "author" // Optional: subscriber, author, editor
}</code></pre>
                </details>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/auth/login</code>
                </div>
                <p class="text-gray-600 mb-2">Authenticate user and get access token</p>
                <details class="mb-4">
                    <summary class="cursor-pointer text-blue-600 hover:underline">Request Body</summary>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm overflow-x-auto"><code>{
  "email": "user@example.com",
  "password": "password123",
  "device_name": "My Device", // Optional
  "remember_me": true // Optional, extends token lifetime
}</code></pre>
                </details>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-get">GET</span>
                    <code class="ml-2 font-mono">/auth/me</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Get current authenticated user information</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/auth/logout</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Logout and revoke current token</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/auth/logout-all</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Logout from all devices (revoke all tokens)</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-get">GET</span>
                    <code class="ml-2 font-mono">/auth/sessions</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">List active sessions/tokens</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-delete">DELETE</span>
                    <code class="ml-2 font-mono">/auth/sessions/{tokenId}</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Revoke specific session/token</p>
            </div>

            <div class="bg-blue-50 p-4 rounded-lg mt-4">
                <h4 class="font-semibold mb-2">Authentication Header</h4>
                <code class="bg-white px-3 py-2 rounded block">Authorization: Bearer YOUR_TOKEN_HERE</code>
            </div>
        </section>

        <!-- Posts Advanced & Management -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Posts Advanced & Management</h2>

            <div class="grid md:grid-cols-2 gap-6">
                <div>
                    <h3 class="font-semibold mb-2">Advanced Queries</h3>
                    <ul class="space-y-1 text-sm text-gray-700">
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/filter</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/search-advanced</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/stats</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/analytics</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/{post}/analytics</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/{post}/related</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">GET /posts/{post}/revisions</code> (placeholder)</li>
                    </ul>
                </div>
                <div>
                    <h3 class="font-semibold mb-2">Management (Editor/Admin)</h3>
                    <ul class="space-y-1 text-sm text-gray-700">
                        <li><code class="bg-gray-100 px-2 py-1 rounded">POST /posts/bulk-update</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">POST /posts/bulk-delete</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">POST /posts/{post}/publish</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">POST /posts/{post}/unpublish</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">POST /posts/{id}/restore</code></li>
                        <li><code class="bg-gray-100 px-2 py-1 rounded">DELETE /posts/{id}/force-delete</code></li>
                    </ul>
                </div>
            </div>
        </section>

        <!-- Public API -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Public API</h2>
            <ul class="space-y-1 text-sm text-gray-700">
                <li><code class="bg-gray-100 px-2 py-1 rounded">GET /public/posts</code></li>
                <li><code class="bg-gray-100 px-2 py-1 rounded">GET /public/posts/{post:slug}</code></li>
                <li><code class="bg-gray-100 px-2 py-1 rounded">GET /public/categories</code></li>
                <li><code class="bg-gray-100 px-2 py-1 rounded">GET /public/search</code></li>
                <li><code class="bg-gray-100 px-2 py-1 rounded">GET /public/stats</code></li>
            </ul>
        </section>

        <!-- Posts API -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Posts API</h2>
            
            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-get">GET</span>
                    <code class="ml-2 font-mono">/posts</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600 mb-2">Get paginated list of posts</p>
                <details>
                    <summary class="cursor-pointer text-blue-600 hover:underline">Query Parameters</summary>
                    <ul class="mt-2 text-sm space-y-1">
                        <li><code>status</code> - Filter by status (draft, published, archived)</li>
                        <li><code>search</code> - Search in title and content</li>
                        <li><code>category</code> - Filter by category ID</li>
                        <li><code>per_page</code> - Items per page (max 50)</li>
                        <li><code>page</code> - Page number</li>
                    </ul>
                </details>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/posts</code>
                    <span class="ml-2 text-xs bg-red-100 text-red-800 px-2 py-1 rounded">Author+</span>
                </div>
                <p class="text-gray-600">Create new post (requires author role or higher)</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-get">GET</span>
                    <code class="ml-2 font-mono">/posts/{id}</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Get single post details</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-put">PUT</span>
                    <code class="ml-2 font-mono">/posts/{id}</code>
                    <span class="ml-2 text-xs bg-red-100 text-red-800 px-2 py-1 rounded">Owner/Editor+</span>
                </div>
                <p class="text-gray-600">Update post (owner or editor/admin)</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-delete">DELETE</span>
                    <code class="ml-2 font-mono">/posts/{id}</code>
                    <span class="ml-2 text-xs bg-red-100 text-red-800 px-2 py-1 rounded">Owner/Editor+</span>
                </div>
                <p class="text-gray-600">Delete post (owner or editor/admin)</p>
            </div>
        </section>

        <!-- Users API -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Users API</h2>
            
            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-get">GET</span>
                    <code class="ml-2 font-mono">/users</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Get list of users</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-put">PUT</span>
                    <code class="ml-2 font-mono">/users/profile</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Update own profile</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-post">POST</span>
                    <code class="ml-2 font-mono">/users/profile/avatar</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Upload profile avatar</p>
            </div>

            <div class="endpoint">
                <div class="flex items-center mb-2">
                    <span class="method-delete">DELETE</span>
                    <code class="ml-2 font-mono">/users/profile/avatar</code>
                    <span class="ml-2 text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Auth Required</span>
                </div>
                <p class="text-gray-600">Delete profile avatar</p>
            </div>

            <div class="bg-yellow-50 p-4 rounded-lg mt-6">
                <h4 class="font-semibold mb-2">Admin Only</h4>
                <ul class="list-disc ml-6 text-sm text-gray-700 space-y-1">
                    <li><code class="bg-white px-2 py-1 rounded">POST /users</code> — Create user</li>
                    <li><code class="bg-white px-2 py-1 rounded">PUT /users/{user}</code> — Update user</li>
                    <li><code class="bg-white px-2 py-1 rounded">DELETE /users/{user}</code> — Delete user</li>
                    <li><code class="bg-white px-2 py-1 rounded">POST /users/{user}/activate</code> — Activate user</li>
                    <li><code class="bg-white px-2 py-1 rounded">POST /users/{user}/deactivate</code> — Deactivate user</li>
                    <li><code class="bg-white px-2 py-1 rounded">GET /users/stats/summary</code> — User statistics</li>
                </ul>
            </div>
        </section>

        <!-- Error Responses -->
        <section class="bg-white rounded-lg shadow-sm p-6 mb-8">
            <h2 class="text-2xl font-bold mb-4">Error Responses</h2>
            
            <div class="space-y-4">
                <div class="border-l-4 border-red-500 pl-4">
                    <h4 class="font-semibold">401 Unauthorized</h4>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm"><code>{
  "success": false,
  "message": "Authentication required",
  "error_code": "UNAUTHENTICATED"
}</code></pre>
                </div>

                <div class="border-l-4 border-orange-500 pl-4">
                    <h4 class="font-semibold">403 Forbidden</h4>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm"><code>{
  "success": false,
  "message": "Insufficient permissions",
  "error_code": "INSUFFICIENT_PERMISSIONS",
  "required_roles": ["admin"]
}</code></pre>
                </div>

                <div class="border-l-4 border-yellow-500 pl-4">
                    <h4 class="font-semibold">429 Rate Limit Exceeded</h4>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm"><code>{
  "success": false,
  "message": "Rate limit exceeded",
  "error_code": "RATE_LIMIT_EXCEEDED",
  "retry_after": 45
}</code></pre>
                </div>

                <div class="border-l-4 border-purple-500 pl-4">
                    <h4 class="font-semibold">422 Validation Error</h4>
                    <pre class="bg-gray-100 p-3 rounded mt-2 text-sm"><code>{
  "success": false,
  "message": "Validation failed",
  "errors": {
    "email": ["The email field is required."],
    "password": ["The password must be at least 8 characters."]
  }
}</code></pre>
                </div>
            </div>
        </section>

        <!-- Testing -->
        <section class="bg-white rounded-lg shadow-sm p-6">
            <h2 class="text-2xl font-bold mb-4">Testing the API</h2>
            
            <div class="bg-gray-50 p-4 rounded-lg mb-4">
                <h4 class="font-semibold mb-2">Quick Test with cURL</h4>
                <pre class="text-sm overflow-x-auto"><code># 1. Register user
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"password123","password_confirmation":"password123"}'

# 2. Login and get token
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# 3. Use token for authenticated request
curl -X GET http://localhost:8000/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"</code></pre>
            </div>

            <div class="bg-blue-50 p-4 rounded-lg">
                <h4 class="font-semibold mb-2">Automated Testing</h4>
                <p class="text-sm mb-2">Gunakan script PowerShell yang tersedia:</p>
                <code class="bg-white px-3 py-2 rounded block">powershell -ExecutionPolicy Bypass -File .\tests\api\test-api.ps1</code>
                <code class="bg-white px-3 py-2 rounded block mt-2">powershell -ExecutionPolicy Bypass -File .\tests\api\test-admin-flow.ps1</code>
            </div>
        </section>
    </div>
</body>
</html>
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Sanctum Installation & Configuration Verification
```powershell
Write-Host "=== Sanctum Installation & Configuration Testing ===" -ForegroundColor Cyan

# Test 1.1: Verify Sanctum package installation
$sanctumInstalled = $false
try {
    & composer show laravel/sanctum *> $null
    if ($LASTEXITCODE -eq 0) { $sanctumInstalled = $true }
} catch { $sanctumInstalled = $false }
if ($sanctumInstalled) { Write-Host "✓ Sanctum package installed" -ForegroundColor Green } else { Write-Host "✗ Sanctum package missing" -ForegroundColor Red }

# Test 1.2: Check Sanctum configuration
if (Test-Path "config/sanctum.php") {
    Write-Host "✓ Sanctum config file exists" -ForegroundColor Green
    try {
        php artisan config:show sanctum.expiration *> $null
        if ($LASTEXITCODE -eq 0) { Write-Host "✓ Token expiration configured" -ForegroundColor Green } else { Write-Host "⚠ Could not read sanctum.expiration" -ForegroundColor Yellow }
    } catch { Write-Host "⚠ Could not read sanctum.expiration" -ForegroundColor Yellow }
} else {
    Write-Host "✗ Sanctum config file missing" -ForegroundColor Red
}

# Test 1.3: Verify personal_access_tokens table
php artisan tinker --execute "echo(\Schema::hasTable('personal_access_tokens') ? '✓ personal_access_tokens table exists' : '✗ personal_access_tokens table missing'), PHP_EOL;"

php artisan tinker --execute "`$req=['id','tokenable_type','tokenable_id','name','token','abilities','last_used_at','expires_at']; `$cols=\Schema::getColumnListing('personal_access_tokens'); `$miss=array_diff(`$req,`$cols); echo empty(`$miss) ? '✓ All required columns present' : '✗ Missing columns: '.implode(', ',`$miss), PHP_EOL;"


# Test 1.4: Check middleware registration
$routes = php artisan route:list 2>&1
if ($routes | Select-String -Quiet "sanctum") { Write-Host "✓ Sanctum middleware registered" -ForegroundColor Green } else { Write-Host "⚠ Sanctum middleware not detected in routes" -ForegroundColor Yellow }
```

#### Test 2: Authentication Flow Testing
```powershell
Write-Host "=== Authentication Flow Testing ===" -ForegroundColor Cyan
$BaseUrl = "http://localhost:8000/api"

# Test 2.1: User model Sanctum integration (via Tinker) — super safe via temp file + STDIN
$tmp = Join-Path $env:TEMP ("tinker-auth-{0}.php" -f (Get-Random))
@'
$user = \App\Models\User::first();
if ($user) {
    $methods = get_class_methods($user);
    $required = array('createToken','tokens','currentAccessToken');
    $missing = array_diff($required, $methods);
    if (empty($missing)) { echo '[OK] User model has all Sanctum methods', PHP_EOL; }
    else { echo '[ERR] Missing method(s): ' . implode(', ', $missing), PHP_EOL; }

    try {
        $token = $user->createToken('test-token', array('posts:read'));
        echo '[OK] Token creation working', PHP_EOL;
        if ($token->accessToken->can('posts:read')) { echo '[OK] Token abilities working', PHP_EOL; }
        $token->accessToken->delete();
        echo '[OK] Token cleanup successful', PHP_EOL;
    } catch (\Exception $e) {
        echo '[ERR] Token creation failed: ' . $e->getMessage(), PHP_EOL;
    }
} else {
    echo '[ERR] No users found for testing', PHP_EOL;
}
'@ | Set-Content -Encoding UTF8 $tmp

# Kirim via STDIN (hindari --execute yang suka mengunyah kutip)
Get-Content -Raw $tmp | php artisan tinker --no-interaction

Remove-Item $tmp -ErrorAction SilentlyContinue

# Test 2.2: API info endpoint format
$r = Invoke-Api -Method GET -Url "$BaseUrl/info"
if ($r -and $r.StatusCode -eq 200) {
    try { $json = $r.Content | ConvertFrom-Json } catch { $json = $null }
    if ($json -and $json.api_version) {
        Write-Host "[OK] API info endpoint working" -ForegroundColor Green
    } else {
        Write-Host "[ERR] API info response invalid" -ForegroundColor Red
    }
} else {
    Write-Host ("[ERR] API info endpoint unreachable (HTTP {0})" -f ($r.StatusCode)) -ForegroundColor Red
}

# Test 2.3: Registration + Login + authenticated request
Write-Host "Testing registration + login..." -ForegroundColor Yellow
$rand = Get-Random
$email = "verify+$rand@test.com"

$register = Invoke-Api -Method POST -Url "$BaseUrl/auth/register" -Body @{
    name='Test Verification User'; email=$email; password='password123'; password_confirmation='password123'
}

if ($register -and ($register.StatusCode -eq 201 -or $register.StatusCode -eq 200)) {
    Write-Host "[OK] Registration endpoint working" -ForegroundColor Green
} else {
    Write-Host ("[WARN] Registration status: {0}" -f ($register.StatusCode)) -ForegroundColor Yellow
}

$token = $null
try {
    if ($register.Content) {
        $regJson = $register.Content | ConvertFrom-Json
        if ($regJson -and $regJson.data -and $regJson.data.token -and $regJson.data.token.access_token) {
            $token = $regJson.data.token.access_token
        }
    }
} catch {}

if (-not $token) {
    $login = Invoke-Api -Method POST -Url "$BaseUrl/auth/login" -Body @{
        email=$email; password='password123'; device_name='verify-test'
    }
    if ($login -and $login.Content) {
        try {
            $loginJson = $login.Content | ConvertFrom-Json
            $token = $loginJson.data.token.access_token
        } catch {}
    }
}

if ($token) {
    $Global:VERIFY_TOKEN = $token
    $auth = Invoke-Api -Method GET -Url "$BaseUrl/auth/me" -Headers @{ Authorization = "Bearer $token" }
    if ($auth -and $auth.StatusCode -eq 200) {
        Write-Host "[OK] Bearer token authentication working" -ForegroundColor Green
        # Write-Host $auth.Content  # uncomment kalau mau lihat payload
    } else {
        Write-Host ("[ERR] Bearer token authentication failed (HTTP {0})" -f $auth.StatusCode) -ForegroundColor Red
    }
} else {
    Write-Host "[ERR] Failed to acquire token" -ForegroundColor Red
}
```

<!-- #### Test 3: Role-Based Authorization Testing
```powershell
Write-Host "=== Role-Based Authorization Testing ===" -ForegroundColor Cyan
$BaseUrl = "http://localhost:8000/api"

# Test 3.1: Middleware functionality (via Tinker)
$tinkerMw = @'
if (class_exists("\App\Http\Middleware\CheckApiRole")) {
    echo "✓ CheckApiRole middleware exists\n";
    $middleware = new \App\Http\Middleware\CheckApiRole();
    if (method_exists($middleware, "handle")) { echo "✓ Middleware has handle method\n"; }
} else {
    echo "✗ CheckApiRole middleware missing\n";
}

if (class_exists("\App\Http\Middleware\ApiRateLimit")) {
    echo "✓ ApiRateLimit middleware exists\n";
} else {
    echo "✗ ApiRateLimit middleware missing\n";
}
'@
php artisan tinker --execute "$tinkerMw"

# Helper to get status code even on failures
function Get-StatusCode {
    param([string]$Url, [hashtable]$Headers)
    try { $resp = Invoke-WebRequest -Uri $Url -Headers $Headers -ErrorAction Stop } catch { $resp = $_.Exception.Response }
    try { return [int]$resp.StatusCode } catch { return -1 }
}

# Test 3.2: Route protection
Write-Host "Testing protected routes..." -ForegroundColor Yellow

# Without authentication (should be 401)
$code = Get-StatusCode -Url "$BaseUrl/admin/dashboard-stats"
if ($code -eq 401) { Write-Host "✓ Admin routes protected from unauthenticated access" -ForegroundColor Green } else { Write-Host ("✗ Expected 401, got {0}" -f $code) -ForegroundColor Red }

# With non-admin token (should be 403 if token exists)
if ($Global:VERIFY_TOKEN) {
    $code = Get-StatusCode -Url "$BaseUrl/admin/content-metrics" -Headers @{ Authorization = "Bearer $($Global:VERIFY_TOKEN)" }
    if ($code -eq 403) { Write-Host "✓ Admin routes protected from non-admin users" -ForegroundColor Green } else { Write-Host ("✗ Expected 403, got {0}" -f $code) -ForegroundColor Red }
}
``` -->

#### Test 3: API Rate Limiting Testing
```powershell
Write-Host "=== API Rate Limiting Testing ===" -ForegroundColor Cyan
$BaseUrl = "http://localhost:8000/api"

# Test 4.1: Rate limit headers
try {
    $head = Invoke-WebRequest -Uri "$BaseUrl/info" -Method Head -ErrorAction Stop
} catch {
    $head = $_.Exception.Response
}
if (-not $head -or $head.StatusCode -eq 405) { $head = Invoke-WebRequest -Uri "$BaseUrl/info" -Method Get -ErrorAction SilentlyContinue }
if ($head -and $head.Headers["X-RateLimit-Limit"]) {
    Write-Host "✓ Rate limit headers present" -ForegroundColor Green
    if ($head.Headers["X-RateLimit-Limit"]) { Write-Host ("  X-RateLimit-Limit: {0}" -f $head.Headers["X-RateLimit-Limit"]) }
    if ($head.Headers["X-RateLimit-Remaining"]) { Write-Host ("  X-RateLimit-Remaining: {0}" -f $head.Headers["X-RateLimit-Remaining"]) }
} else {
    Write-Host "✗ Rate limit headers missing" -ForegroundColor Red
}

# Test 4.2: Rate limit enforcement (test endpoint)
function Get-Code {
    param([string]$Url)
    try { $r = Invoke-WebRequest -Uri $Url -ErrorAction Stop } catch { $r = $_.Exception.Response }
    try { return [int]$r.StatusCode } catch { return -1 }
}

try { $ok = (Get-Code "$BaseUrl/test/rate-limit"); $endpointAvailable = ($ok -gt 0) } catch { $endpointAvailable = $false }
if ($endpointAvailable) {
    Write-Host "Testing rate limit enforcement..." -ForegroundColor Yellow
    $triggered = $false
    for ($i = 1; $i -le 8; $i++) {
        $code = Get-Code "$BaseUrl/test/rate-limit"
        if ($code -eq 429) { Write-Host ("✓ Rate limiting triggered after {0} requests" -f $i) -ForegroundColor Green; $triggered = $true; break }
        Start-Sleep -Milliseconds 100
    }
    if (-not $triggered) { Write-Host "⚠ Rate limiting not triggered (limit may be higher)" -ForegroundColor Yellow }
} else {
    Write-Host "⚠ Rate limit test endpoint not available" -ForegroundColor Yellow
}
```

#### Test 4: API Security & Performance Testing
```powershell
Write-Host "=== API Security & Performance Testing ===" -ForegroundColor Cyan
$BaseUrl = "http://localhost:8000/api"

# Helper to get status code on errors
function Invoke-Code {
    param([string]$Url, [string]$Method = 'GET', [hashtable]$Body, [hashtable]$Headers, [string]$ContentType)
    $ct = if ([string]::IsNullOrWhiteSpace($ContentType)) { 'application/json' } else { $ContentType }
    try {
        if ($Body) {
            $json = $Body | ConvertTo-Json -Depth 8
            $r = Invoke-WebRequest -Uri $Url -Method $Method -Headers $Headers -ContentType $ct -Body $json -ErrorAction Stop
        } else {
            $r = Invoke-WebRequest -Uri $Url -Method $Method -Headers $Headers -ContentType $ct -ErrorAction Stop
        }
    } catch {
        $r = $_.Exception.Response
    }
    try { return [int]$r.StatusCode } catch { return -1 }
}

# Test 5.1: CSRF protection for API routes (should NOT require CSRF, i.e., NOT 419)
$csrfCode = Invoke-Code -Url "$BaseUrl/auth/login" -Method POST -Body @{ email='test'; password='test' }
if ($csrfCode -ne 419) { Write-Host "✓ API routes properly exempt from CSRF protection" -ForegroundColor Green } else { Write-Host "✗ API routes incorrectly requiring CSRF tokens" -ForegroundColor Red }

# Test 5.2: Content-Type validation (send invalid content-type)
$invalidCode = Invoke-Code -Url "$BaseUrl/auth/login" -Method POST -Headers @{ 'Content-Type' = 'text/plain' } -ContentType 'text/plain'
Write-Host ("Content-Type validation response: {0}" -f $invalidCode)

# Test 5.3: Large payload handling
Write-Host "Testing large payload handling..." -ForegroundColor Yellow
$largeName = ('A' * 1000)
$largeBody = @{ name=$largeName; email='large@test.com'; password='password123'; password_confirmation='password123' }
$largeCode = Invoke-Code -Url "$BaseUrl/auth/register" -Method POST -Body $largeBody
Write-Host ("Large payload response: {0}" -f $largeCode)

# Test 5.4: Response time measurement
Write-Host "Measuring API response times..." -ForegroundColor Yellow
foreach ($endpoint in @("/api/info", "/api/public/stats")) {
    $url = "http://localhost:8000$endpoint"
    try { $t = (Measure-Command { Invoke-WebRequest -Uri $url -ErrorAction SilentlyContinue | Out-Null }).TotalSeconds; Write-Host ("  {0}: {1:N3}s" -f $endpoint, $t) } catch { Write-Host ("  {0}: unreachable" -f $endpoint) }
}
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Sanctum Installation Issues
**Gejala:** `Class 'Laravel\Sanctum\...' not found` errors
**Solusi:**
```bash
# Verify Composer installation
composer show laravel/sanctum

# If not installed, install manually
composer require laravel/sanctum:^4.0

# Clear application cache
php artisan config:clear
php artisan cache:clear
php artisan route:clear

# Republish Sanctum configuration
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider" --force

# Dump autoload
composer dump-autoload
```

#### Problem 2: Token Authentication Not Working
**Gejala:** `Unauthenticated` error meskipun token valid
**Solusi:**
```bash
# Check Sanctum middleware in api routes
php artisan route:list | grep sanctum

# Verify token format
php artisan tinker --execute="
\$user = \App\Models\User::first();
if (\$user) {
    \$token = \$user->createToken('debug-token');
    echo 'Generated token: ' . \$token->plainTextToken . PHP_EOL;
    echo 'Token parts: ' . count(explode('|', \$token->plainTextToken)) . PHP_EOL;
    \$token->accessToken->delete();
}
"

# Test token validation manually
curl -v -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8000/api/auth/me

# Check personal_access_tokens table
mysql -u laravel_user -p laravel_app -e "
SELECT id, tokenable_id, name, abilities, last_used_at, expires_at, created_at 
FROM personal_access_tokens 
ORDER BY created_at DESC 
LIMIT 5;"

# Clear Sanctum cache
php artisan config:cache
```

#### Problem 3: Rate Limiting Not Working
**Gejala:** Rate limiting tidak ter-trigger atau tidak berfungsi
**Solusi:**
```bash
# Check middleware registration
grep -r "ApiRateLimit" bootstrap/app.php

# Verify cache driver for rate limiting
php artisan tinker --execute="
echo 'Cache driver: ' . config('cache.default') . PHP_EOL;
echo 'Cache working: ' . (\Cache::put('test', 'value', 60) ? 'Yes' : 'No') . PHP_EOL;
echo 'Cache retrieve: ' . \Cache::get('test', 'Not found') . PHP_EOL;
\Cache::forget('test');
"

# Test rate limiter directly
php artisan tinker --execute="
use Illuminate\Support\Facades\RateLimiter;

\$key = 'test-rate-limit';
echo 'Rate limiter test:' . PHP_EOL;

for (\$i = 1; \$i <= 5; \$i++) {
    if (RateLimiter::tooManyAttempts(\$key, 3)) {
        echo 'Rate limit triggered at attempt ' . \$i . PHP_EOL;
        break;
    }
    RateLimiter::hit(\$key, 60);
    echo 'Attempt ' . \$i . ' - ' . RateLimiter::attempts(\$key) . ' attempts' . PHP_EOL;
}

RateLimiter::clear(\$key);
"

# Check Redis/Cache connection if using Redis
redis-cli ping 2>/dev/null || echo "Redis not available, using file cache"
```

#### Problem 4: Role-Based Authorization Failing
**Gejala:** Users dengan role yang benar tetap dapat akses denied
**Solusi:**
```bash
# Check user roles in database
mysql -u laravel_user -p laravel_app -e "SELECT id, name, email, role, is_active FROM users;"

# Verify middleware order in bootstrap/app.php
grep -A 10 -B 5 "api.role" bootstrap/app.php

# Test role checking logic
php artisan tinker --execute="
\$user = \App\Models\User::where('role', 'admin')->first();
if (\$user) {
    echo 'User role: ' . \$user->role . PHP_EOL;
    echo 'Can manage posts: ' . (\$user->canManagePosts() ? 'Yes' : 'No') . PHP_EOL;
    echo 'Can manage users: ' . (\$user->canManageUsers() ? 'Yes' : 'No') . PHP_EOL;
    echo 'Has role admin: ' . (\$user->hasRole('admin') ? 'Yes' : 'No') . PHP_EOL;
    echo 'Is active: ' . (\$user->is_active ? 'Yes' : 'No') . PHP_EOL;
} else {
    echo 'No admin user found' . PHP_EOL;
}
"

# Check routes with role middleware
php artisan route:list | grep "api.role"

# Test middleware directly
php artisan tinker --execute="
\$request = new \Illuminate\Http\Request();
\$user = \App\Models\User::where('role', 'admin')->first();
\$request->setUserResolver(function() use (\$user) { return \$user; });

\$middleware = new \App\Http\Middleware\CheckApiRole();
try {
    \$response = \$middleware->handle(\$request, function(\$req) {
        return response()->json(['message' => 'Access granted']);
    }, 'admin');
    echo 'Middleware test: Access granted' . PHP_EOL;
} catch (\Exception \$e) {
    echo 'Middleware test failed: ' . \$e->getMessage() . PHP_EOL;
}
"
```

#### Problem 5: API Performance Issues
**Gejala:** API responses lambat atau timeout
**Solusi:**
```bash
# Check database connection performance
php artisan tinker --execute="
\$start = microtime(true);
\$users = \App\Models\User::count();
\$end = microtime(true);
echo 'Database query time: ' . round((\$end - \$start) * 1000, 2) . 'ms' . PHP_EOL;
echo 'Users count: ' . \$users . PHP_EOL;
"

# Optimize configurations
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Check for N+1 queries in API responses
# Enable query logging
php artisan tinker --execute="
\DB::enableQueryLog();
\$posts = \App\Models\Post::with(['user', 'category', 'tags'])->take(5)->get();
\$queries = \DB::getQueryLog();
echo 'Number of queries: ' . count(\$queries) . PHP_EOL;
foreach (\$queries as \$query) {
    echo \$query['query'] . PHP_EOL;
}
"

# Monitor memory usage
php artisan tinker --execute="
echo 'Memory usage: ' . round(memory_get_usage(true) / 1024 / 1024, 2) . 'MB' . PHP_EOL;
echo 'Peak memory: ' . round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB' . PHP_EOL;
"

# Check Laravel debug mode
php artisan tinker --execute="
echo 'App debug: ' . (config('app.debug') ? 'enabled' : 'disabled') . PHP_EOL;
echo 'App env: ' . config('app.env') . PHP_EOL;
"
```

#### Problem 6: Token Expiration Issues
**Gejala:** Tokens expire unexpectedly atau tidak expire
**Solusi:**
```bash
# Check Sanctum expiration configuration
php artisan tinker --execute="
echo 'Sanctum expiration: ' . config('sanctum.expiration') . ' minutes' . PHP_EOL;
echo 'Current time: ' . now()->toISOString() . PHP_EOL;
"

# Check existing tokens
mysql -u laravel_user -p laravel_app -e "
SELECT name, abilities, expires_at, created_at, 
       CASE 
         WHEN expires_at IS NULL THEN 'Never expires'
         WHEN expires_at > NOW() THEN 'Valid'
         ELSE 'Expired'
       END as status
FROM personal_access_tokens 
ORDER BY created_at DESC;"

# Test token creation with expiration
php artisan tinker --execute "
\$user = \App\Models\User::first();
if (\$user) {
    // Create token with 1 hour expiration
    \$token = \$user->createApiToken('test-expiry', ['*'], now()->addHour());
    echo 'Token created with expiration: ' . \$token->accessToken->expires_at . PHP_EOL;
    
    // Check if expires_at is set correctly
    \$tokenRecord = \$user->tokens()->latest()->first();
    echo 'Database expires_at: ' . \$tokenRecord->expires_at . PHP_EOL;
    
    // Cleanup
    \$tokenRecord->delete();
}
"

# Clean up expired tokens
php artisan tinker --execute="
\$expired = \DB::table('personal_access_tokens')
              ->where('expires_at', '<', now())
              ->count();
echo 'Expired tokens: ' . \$expired . PHP_EOL;

// \DB::table('personal_access_tokens')->where('expires_at', '<', now())->delete();
// echo 'Expired tokens cleaned up' . PHP_EOL;
"
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

<!-- #### ✅ Sanctum Installation & Configuration
- [ ] Screenshot output `composer show laravel/sanctum` (jalankan di root project; foto paket & versi)
- [ ] Lampirkan file `config/sanctum.php` jika ada kustomisasi (cukup file, tidak perlu screenshot)
- [ ] Screenshot `php artisan migrate:status` menampilkan migrasi `personal_access_tokens` = Yes
- [ ] Salin variabel `SANCTUM_*` dan `API_*` ke `submission/week6/env-sanitized.txt` (jangan unggah `.env` penuh)
- [ ] Lampirkan `app/Providers/AppServiceProvider.php` (definisi rate limiter `api`); opsional screenshot header `X-RateLimit-*` via `curl.exe -i http://localhost:8000/api/posts`

#### ✅ Authentication System Implementation
- [ ] Lampirkan `app/Http/Controllers/Api/AuthController.php` (register, login, logout, logout-all)
- [ ] Lampirkan `app/Models/User.php` dengan trait `HasApiTokens`
- [ ] Screenshot login menghasilkan token dan validasi `GET /api/auth/me` (Perintah: login via `curl`, lalu `curl.exe -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8000/api/auth/me`)
- [ ] Bukti uji endpoint auth (`register`, `login`, `me`, `logout`, `logout-all`) via Postman/cURL

#### ✅ Authorization & Role-Based Access Control
- [ ] Lampirkan `app/Http/Middleware/CheckApiRole.php` (role checking)
- [ ] Lampirkan `app/Http/Middleware/ApiRateLimit.php` (jika ada) atau konfigurasi rate limit dinamis
- [ ] Screenshot akses admin/editor-only dengan user biasa (403) dan dengan role yang benar (200)
  — contoh: `curl.exe -i -X POST http://localhost:8000/api/posts/{id}/publish` (protected oleh `auth:sanctum, api.role:editor,admin`)
- [ ] Lampirkan `routes/api.php` (route groups + middleware)

#### ✅ Enhanced API Controllers
- [ ] Lampirkan `app/Http/Controllers/Api/UserController.php` (CRUD + profile management)
- [ ] Lampirkan `app/Http/Controllers/Api/PostApiController.php` (ownership + authorization)
- [ ] Screenshot respons JSON rapi contoh: `curl.exe -s http://localhost:8000/api/posts`
- [ ] Screenshot upload avatar berhasil Postman: `POST /api/users/profile/avatar` (form-data `avatar`)

#### ✅ API Security Features
- [ ] Screenshot header `X-RateLimit-Limit` dan `X-RateLimit-Remaining` (Perintah: `curl.exe -i -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8000/api/posts`)
- [ ] Bukti abilities pada token (lampirkan potongan kode pembuatan token dengan abilities, mis. `createToken($name, $abilities)`)
- [ ] Lampirkan potongan `app/Http/Middleware/VerifyCsrfToken.php` (`$except` untuk `api/*`) atau bukti rute API tidak memerlukan CSRF
- [ ] Lampirkan potongan kode/log audit keamanan (mis. pembuatan/logout token) + screenshot `storage/logs/laravel.log`

#### ✅ Testing & Documentation
- [ ] Screenshot hasil running testing pada `tests\api\`
- [ ] Lampirkan `api-docs.html` atau `submission/week6/api-docs.md` (ringkas bila pakai Markdown)
- [ ] Screenshot alur auth: `register`, `login`, `me`, `logout`, `logout-all` (Postman/cURL)

#### ✅ Advanced Features Implementation
- [ ] Multi-device token management (sessions endpoint) screenshot daftar sesi aktif (contoh: `GET /api/auth/sessions` bila tersedia)
- [ ] Token expiration handling bukti `SANCTUM_TOKEN_EXPIRATION` dan field `expires_at` pada `personal_access_tokens`
- [ ] Bulk operations bukti hanya role yang berwenang (screenshot respons + potongan kode otorisasi)
- [ ] Admin analytics endpoints — screenshot admin (200) vs user (403)
  — contoh: `curl.exe -i http://localhost:8000/api/admin/dashboard-stats` atau `curl.exe -i http://localhost:8000/api/admin/content-metrics` (pastikan dilindungi `auth:sanctum` + `api.role:admin` jika diterapkan) -->

#### ✅ Sanctum Installation & Configuration
- [ ] Screenshot output `composer show laravel/sanctum` (paket & versi).  
- [ ] Screenshot `php artisan migrate:status` menampilkan `personal_access_tokens = Yes`.  
- [ ] Salin variabel `SANCTUM_*` dan `API_*` ke `submission/week6/env-sanitized.txt` (jangan unggah `.env` penuh).  
- [ ] Lampirkan `app/Providers/AppServiceProvider.php` (definisi rate limiter `api`); opsional screenshot header `X-RateLimit-*` via `curl`.  

#### ✅ Authentication System
- [ ] Lampirkan `app/Http/Controllers/Api/AuthController.php` (register, login, logout, me).  
- [ ] Lampirkan `app/Models/User.php` dengan trait `HasApiTokens`.  
- [ ] Screenshot login menghasilkan token dan validasi `GET /api/auth/me` dengan Bearer token.  
- [ ] Bukti uji endpoint auth (`register`, `login`, `me`, `logout`) via Postman/cURL.  

#### ✅ Authorization & Role-Based Access Control
- [ ] Lampirkan `app/Http/Middleware/CheckApiRole.php`.  
- [ ] Lampirkan `routes/api.php` (route groups + middleware).  
- [ ] Screenshot akses endpoint admin/editor-only dengan user biasa (403) dan dengan role yang benar (200).  

#### ✅ CRUD API Sederhana
- [ ] Lampirkan `app/Http/Controllers/Api/PostApiController.php` (CRUD + ownership check).  
- [ ] Screenshot respons JSON rapi contoh: `curl.exe -s http://localhost:8000/api/posts`.  

#### ✅ API Security Features (Dasar)
- [ ] Screenshot header `X-RateLimit-Limit` dan `X-RateLimit-Remaining`, contoh : `curl.exe -i http://localhost:8000/api/test/rate-limit`  
- [ ] Bukti abilities pada token (potongan kode `createToken($name, $abilities)`).  

#### ✅ Dokumentasi Ringkas
- [ ] Lampirkan `submission/week6/api-docs.md` (ringkas, minimal daftar endpoint auth + 1 CRUD terproteksi, contoh request/response).  

### Create comprehensive README
cat > submission/week6/README.md << 'EOF'
```
# Week 6: API Development & Authentication

## Completed Implementation
- [x] Laravel Sanctum installation dan configuration
- [x] Comprehensive authentication system (register, login, logout)
- [x] Role-based authorization dengan custom middleware
- [x] API rate limiting dengan dynamic configuration
- [x] Token-based authentication dengan abilities system
- [x] Multi-device session management
- [x] File upload functionality (avatar management)
- [x] Advanced API security features
- [x] Comprehensive testing framework
- [x] Complete API documentation

## Technical Architecture

### Authentication Flow
1. User registration dengan role assignment
2. Login generates JWT-like Sanctum token
3. Token includes abilities based on user role
4. Middleware validates token dan checks permissions
5. Rate limiting applied based on authentication status

### Authorization Levels
- **Public**: API info, public content
- **Authenticated**: Basic user operations
- **Author**: Content creation dan own content management
- **Editor**: Content moderation dan multi-user management
- **Admin**: Full system access dan analytics

### Security Features
- CSRF protection exempt untuk API routes
- Rate limiting: 200/min authenticated, 60/min guest
- Token expiration dengan configurable timeouts
- Ability-based permissions untuk granular control
- Audit logging untuk security events
- Input validation dan sanitization

## API Endpoints Summary
### Authentication
- POST /api/auth/register - User registration
- POST /api/auth/login - User login
- GET /api/auth/me - Current user info
- POST /api/auth/logout - Logout current session
- POST /api/auth/logout-all - Logout all devices

### Users Management
- GET /api/users - List users (authenticated)
- PUT /api/users/profile - Update own profile
- POST /api/users/profile/avatar - Upload avatar
- PUT /api/users/{id} - Update user (admin only)

### Posts Management
- GET /api/posts - List posts
- POST /api/posts - Create post (author+)
- PUT /api/posts/{id} - Update post (owner/editor+)
- DELETE /api/posts/{id} - Delete post (owner/editor+)
- POST /api/posts/{id}/publish - Publish post (editor+)

### Admin Analytics
- GET /api/admin/dashboard - Dashboard statistics
- GET /api/admin/users/stats - User statistics
- GET /api/posts/analytics - Content analytics

## Performance Optimizations
- Eager loading untuk relationships
- Database query optimization
- Response caching untuk static data
- Efficient pagination implementation
- Memory usage optimization

## Testing Results
- ✓ All authentication flows working
- ✓ Role-based authorization enforced
- ✓ Rate limiting functional
- ✓ Token abilities system working
- ✓ API security measures implemented
- ✓ Performance within acceptable limits
- ✓ Comprehensive error handling
- ✓ Audit logging functional

## Production Readiness
- Environment configuration separated
- Security best practices implemented
- Error handling dan logging comprehensive
- Documentation complete
- Testing framework established

## Next Steps
- Week 7: User Authentication & Authorization (Frontend)
- Advanced permission systems
- OAuth integration preparation
- API versioning implementation
EOF

# Run final testing suite
echo "Running final API testing suite..."
powershell -ExecutionPolicy Bypass -File .\\tests\\api\\test-api.ps1 > submission/week6/tests/final-test-api-results.txt 2>&1
powershell -ExecutionPolicy Bypass -File .\\tests\\api\\test-admin-flow.ps1 > submission/week6/tests/final-admin-flow-results.txt 2>&1

# Generate API endpoint documentation
php artisan route:list --path=api --json > submission/week6/api-routes.json

# Export sample API responses
mkdir -p submission/week6/api-samples
curl -s http://localhost:8000/api/info > submission/week6/api-samples/info-response.json
curl -s http://localhost:8000/api/public/stats > submission/week6/api-samples/public-stats.json
```

**Format Submission:**
1. Buat folder submission/week6/
2. Masukkan semua screenshot dengan nama yang jelas
3. Buat file laporan dalam format Markdown
4. Commit dan push ke repository
5. Sertakan link commit terakhir

