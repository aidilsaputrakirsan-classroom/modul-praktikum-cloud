# WEEK 6: API Development & Authentication
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 6
- **Durasi**: 100 menit  
- **Topik**: Advanced API Development dengan Laravel Sanctum Authentication
- **Target**: Mahasiswa Semester 6
- **Platform**: Google Cloud Shell
- **Repository**: github.com/aidilsaputrakirsan/praktikum-cc

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
cd ~/praktikum-cc/week2/laravel-app

# Verifikasi existing API endpoints dan database
php artisan route:list | grep api
php artisan tinker --execute="
echo 'Users count: ' . \App\Models\User::count() . PHP_EOL;
echo 'Posts count: ' . \App\Models\Post::count() . PHP_EOL;
echo 'Existing API routes: ' . \Route::getRoutes()->count() . PHP_EOL;
"

# Check Laravel version compatibility dengan Sanctum
php artisan --version
composer show laravel/sanctum 2>/dev/null || echo "Sanctum not installed yet"
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Laravel Sanctum Installation & Configuration (20 menit)**

##### Step 1.1: Install Laravel Sanctum Package
```bash
# Install Laravel Sanctum via Composer
composer require laravel/sanctum

# Publish Sanctum configuration dan migration files
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

# Verify published files
ls -la config/sanctum.php
ls -la database/migrations/ | grep sanctum
```

##### Step 1.2: Configure Sanctum Settings
```bash
# Edit Sanctum configuration file
nano config/sanctum.php
```

**Update config/sanctum.php dengan custom settings:**
```php
<?php

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
        'verify_csrf_token' => App\Http\Middleware\VerifyCsrfToken::class,
        'encrypt_cookies' => App\Http\Middleware\EncryptCookies::class,
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
```bash
# Add Sanctum configuration ke .env file
nano .env
```

**Tambahkan ke .env file:**
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

##### Step 1.4: Run Sanctum Migrations
```bash
# Run migration untuk personal access tokens table
php artisan migrate

# Verify migration success
php artisan migrate:status | grep sanctum
mysql -u laravel_user -p laravel_app -e "DESCRIBE personal_access_tokens;"
```

##### Step 1.5: Configure User Model untuk Sanctum
```bash
# Update User model untuk enable Sanctum traits
nano app/Models/User.php
```

**Update User model dengan Sanctum integration:**
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
ls -la app/Http/Controllers/Api/
ls -la app/Http/Middleware/
```

##### Step 2.2: Implement AuthController dengan Comprehensive Features
```bash
# Edit AuthController untuk authentication endpoints
nano app/Http/Controllers/Api/AuthController.php
```

**Isi file app/Http/Controllers/Api/AuthController.php:**
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
```bash
# Edit CheckApiRole middleware
nano app/Http/Middleware/CheckApiRole.php
```

**Isi file app/Http/Middleware/CheckApiRole.php:**
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
```bash
# Edit ApiRateLimit middleware
nano app/Http/Middleware/ApiRateLimit.php
```

**Isi file app/Http/Middleware/ApiRateLimit.php:**
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
```bash
# Register middleware dalam Kernel.php
nano app/Http/Kernel.php
```

**Update app/Http/Kernel.php untuk register middleware:**
```php
// Dalam $middlewareAliases array, tambahkan:
'api.role' => \App\Http\Middleware\CheckApiRole::class,
'api.rate' => \App\Http\Middleware\ApiRateLimit::class,

// Dalam $middlewareGroups['api'], update menjadi:
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    \App\Http\Middleware\ApiRateLimit::class,
    'throttle:api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

#### **Bagian 3: API Routes dengan Authentication dan Authorization (20 menit)**

##### Step 3.1: Update API Routes dengan Authentication
```bash
# Update API routes untuk include authentication
nano routes/api.php
```

**Update routes/api.php dengan comprehensive API structure:**
```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\UserController;
use App\Http\Controllers\Api\PostApiController;
use App\Http\Controllers\Api\CategoryApiController;

/*
|--------------------------------------------------------------------------
| API Routes - Praktikum Cloud Computing ITK Week 6
|--------------------------------------------------------------------------
|
| Comprehensive API dengan authentication, authorization, dan versioning
| Menggunakan Laravel Sanctum untuk token-based authentication
|
*/

// API Information endpoint (public)
Route::get('/info', function (Request $request) {
    return response()->json([
        'application' => 'Praktikum Cloud Computing ITK',
        'api_version' => 'v1',
        'laravel_version' => app()->version(),
        'sanctum_version' => '3.x',
        'endpoints' => [
            'auth' => '/api/auth/*',
            'users' => '/api/users',
            'posts' => '/api/posts',
            'categories' => '/api/categories',
        ],
        'authentication' => [
            'type' => 'Bearer Token (Sanctum)',
            'header' => 'Authorization: Bearer {token}',
            'login' => 'POST /api/auth/login',
            'register' => 'POST /api/auth/register',
        ],
        'rate_limits' => [
            'authenticated' => '200 requests per minute',
            'guest' => '60 requests per minute',
        ],
        'documentation' => '/api/docs',
        'timestamp' => now()->toISOString(),
    ]);
});

/*
|--------------------------------------------------------------------------
| Authentication Routes (Public)
|--------------------------------------------------------------------------
*/

Route::prefix('auth')->name('api.auth.')->group(function () {
    // Public authentication endpoints
    Route::post('register', [AuthController::class, 'register'])->name('register');
    Route::post('login', [AuthController::class, 'login'])->name('login');
    
    // Protected authentication endpoints
    Route::middleware('auth:sanctum')->group(function () {
        Route::get('me', [AuthController::class, 'me'])->name('me');
        Route::post('logout', [AuthController::class, 'logout'])->name('logout');
        Route::post('logout-all', [AuthController::class, 'logoutAll'])->name('logout-all');
        Route::get('sessions', [AuthController::class, 'sessions'])->name('sessions');
        Route::delete('sessions/{tokenId}', [AuthController::class, 'revokeSession'])->name('revoke-session');
    });
});

/*
|--------------------------------------------------------------------------
| Protected API Routes (Requires Authentication)
|--------------------------------------------------------------------------
*/

Route::middleware(['auth:sanctum'])->group(function () {
    
    /*
    |--------------------------------------------------------------------------
    | User Management Routes
    |--------------------------------------------------------------------------
    */
    
    Route::prefix('users')->name('api.users.')->group(function () {
        // Public user endpoints (all authenticated users)
        Route::get('/', [UserController::class, 'index'])->name('index');
        Route::get('{user}', [UserController::class, 'show'])->name('show');
        
        // User profile management (own profile only)
        Route::put('profile', [UserController::class, 'updateProfile'])->name('update-profile');
        Route::post('profile/avatar', [UserController::class, 'uploadAvatar'])->name('upload-avatar');
        Route::delete('profile/avatar', [UserController::class, 'deleteAvatar'])->name('delete-avatar');
        
        // Admin-only user management
        Route::middleware('api.role:admin')->group(function () {
            Route::post('/', [UserController::class, 'store'])->name('store');
            Route::put('{user}', [UserController::class, 'update'])->name('update');
            Route::delete('{user}', [UserController::class, 'destroy'])->name('destroy');
            Route::post('{user}/activate', [UserController::class, 'activate'])->name('activate');
            Route::post('{user}/deactivate', [UserController::class, 'deactivate'])->name('deactivate');
        });
    });

    /*
    |--------------------------------------------------------------------------
    | Posts API Routes dengan Role-Based Access
    |--------------------------------------------------------------------------
    */
    
    Route::prefix('posts')->name('api.posts.')->group(function () {
        // Public read access (all authenticated users)
        Route::get('/', [PostApiController::class, 'index'])->name('index');
        Route::get('{post}', [PostApiController::class, 'show'])->name('show');
        Route::get('search/{keyword}', [PostApiController::class, 'search'])->name('search');
        Route::get('category/{category}', [PostApiController::class, 'byCategory'])->name('by-category');
        Route::get('tag/{tag}', [PostApiController::class, 'byTag'])->name('by-tag');
        Route::get('featured', [PostApiController::class, 'featured'])->name('featured');
        
        // Content creation (author, editor, admin)
        Route::middleware('api.role:author,editor,admin')->group(function () {
            Route::post('/', [PostApiController::class, 'store'])->name('store');
            Route::put('{post}', [PostApiController::class, 'update'])->name('update');
            Route::delete('{post}', [PostApiController::class, 'destroy'])->name('destroy');
        });
        
        // Editorial functions (editor, admin)
        Route::middleware('api.role:editor,admin')->group(function () {
            Route::post('bulk-update', [PostApiController::class, 'bulkUpdate'])->name('bulk-update');
            Route::post('bulk-delete', [PostApiController::class, 'bulkDelete'])->name('bulk-delete');
            Route::post('{post}/publish', [PostApiController::class, 'publish'])->name('publish');
            Route::post('{post}/unpublish', [PostApiController::class, 'unpublish'])->name('unpublish');
        });
        
        // Analytics dan advanced features (admin only)
        Route::middleware('api.role:admin')->group(function () {
            Route::get('analytics', [PostApiController::class, 'analytics'])->name('analytics');
            Route::get('{post}/analytics', [PostApiController::class, 'postAnalytics'])->name('post-analytics');
            Route::post('{post}/restore', [PostApiController::class, 'restore'])->name('restore');
            Route::delete('{post}/force-delete', [PostApiController::class, 'forceDelete'])->name('force-delete');
        });
    });

    /*
    |--------------------------------------------------------------------------
    | Categories API Routes
    |--------------------------------------------------------------------------
    */
    
    Route::prefix('categories')->name('api.categories.')->group(function () {
        // Public read access
        Route::get('/', [CategoryApiController::class, 'index'])->name('index');
        Route::get('{category}', [CategoryApiController::class, 'show'])->name('show');
        Route::get('tree', [CategoryApiController::class, 'tree'])->name('tree');
        Route::get('active', [CategoryApiController::class, 'active'])->name('active');
        
        // Category management (editor, admin)
        Route::middleware('api.role:editor,admin')->group(function () {
            Route::post('/', [CategoryApiController::class, 'store'])->name('store');
            Route::put('{category}', [CategoryApiController::class, 'update'])->name('update');
            Route::delete('{category}', [CategoryApiController::class, 'destroy'])->name('destroy');
        });
    });

    /*
    |--------------------------------------------------------------------------
    | Admin Analytics dan Management Routes
    |--------------------------------------------------------------------------
    */
    
    Route::prefix('admin')->name('api.admin.')->middleware('api.role:admin')->group(function () {
        // Dashboard statistics
        Route::get('dashboard', function () {
            return response()->json([
                'success' => true,
                'data' => [
                    'users' => [
                        'total' => \App\Models\User::count(),
                        'active' => \App\Models\User::where('is_active', true)->count(),
                        'by_role' => \App\Models\User::selectRaw('role, COUNT(*) as count')
                                                   ->groupBy('role')
                                                   ->pluck('count', 'role'),
                        'recent_registrations' => \App\Models\User::where('created_at', '>=', now()->subDays(7))
                                                                 ->count(),
                    ],
                    'posts' => [
                        'total' => \App\Models\Post::count(),
                        'published' => \App\Models\Post::published()->count(),
                        'draft' => \App\Models\Post::where('status', 'draft')->count(),
                        'archived' => \App\Models\Post::where('status', 'archived')->count(),
                        'this_month' => \App\Models\Post::whereMonth('created_at', now()->month)->count(),
                    ],
                    'categories' => [
                        'total' => \App\Models\Category::count(),
                        'active' => \App\Models\Category::where('is_active', true)->count(),
                        'with_posts' => \App\Models\Category::has('posts')->count(),
                    ],
                    'activity' => [
                        'posts_this_week' => \App\Models\Post::where('created_at', '>=', now()->subWeek())->count(),
                        'users_this_week' => \App\Models\User::where('created_at', '>=', now()->subWeek())->count(),
                        'active_tokens' => \DB::table('personal_access_tokens')
                                             ->where('expires_at', '>', now())
                                             ->orWhereNull('expires_at')
                                             ->count(),
                    ]
                ],
                'generated_at' => now()->toISOString(),
            ]);
        })->name('dashboard');
        
        // User management statistics
        Route::get('users/stats', [UserController::class, 'statistics'])->name('users.stats');
        
        // Content moderation queue
        Route::get('moderation/posts', [PostApiController::class, 'moderationQueue'])->name('moderation.posts');
        Route::get('moderation/comments', function () {
            // Implementation untuk comment moderation
            return response()->json(['message' => 'Comment moderation endpoint']);
        })->name('moderation.comments');
        
        // System health check
        Route::get('health', function () {
            return response()->json([
                'success' => true,
                'system' => [
                    'database' => \DB::connection()->getPdo() ? 'connected' : 'disconnected',
                    'cache' => \Cache::store()->getStore() ? 'working' : 'error',
                    'queue' => 'working', // Implement actual queue check
                    'storage' => \Storage::disk('public')->exists('') ? 'accessible' : 'error',
                ],
                'performance' => [
                    'memory_usage' => memory_get_usage(true),
                    'peak_memory' => memory_get_peak_usage(true),
                    'execution_time' => microtime(true) - LARAVEL_START,
                ],
                'timestamp' => now()->toISOString(),
            ]);
        })->name('health');
    });
});

/*
|--------------------------------------------------------------------------
| Public Content Routes (No Authentication Required)
|--------------------------------------------------------------------------
*/

Route::prefix('public')->name('api.public.')->group(function () {
    // Public blog content
    Route::get('posts', [PostApiController::class, 'publicIndex'])->name('posts.index');
    Route::get('posts/{post:slug}', [PostApiController::class, 'publicShow'])->name('posts.show');
    Route::get('categories', [CategoryApiController::class, 'publicIndex'])->name('categories.index');
    Route::get('search', [PostApiController::class, 'publicSearch'])->name('search');
    
    // Public statistics
    Route::get('stats', function () {
        return response()->json([
            'success' => true,
            'data' => [
                'published_posts' => \App\Models\Post::published()->count(),
                'active_categories' => \App\Models\Category::active()->count(),
                'total_authors' => \App\Models\User::whereIn('role', ['author', 'editor', 'admin'])->count(),
                'recent_posts' => \App\Models\Post::published()
                                                 ->latest('published_at')
                                                 ->take(5)
                                                 ->get(['id', 'title', 'slug', 'published_at']),
            ],
            'generated_at' => now()->toISOString(),
        ]);
    })->name('stats');
});

/*
|--------------------------------------------------------------------------
| API Testing Routes (Development Only)
|--------------------------------------------------------------------------
*/

if (app()->environment(['local', 'testing'])) {
    Route::prefix('test')->name('api.test.')->group(function () {
        // Test authentication
        Route::get('auth', function (Request $request) {
            return response()->json([
                'authenticated' => (bool) $request->user(),
                'user' => $request->user()?->only(['id', 'name', 'email', 'role']),
                'token_info' => $request->user()?->getCurrentTokenInfo(),
                'timestamp' => now()->toISOString(),
            ]);
        })->middleware('auth:sanctum')->name('auth');
        
        // Test rate limiting
        Route::get('rate-limit', function () {
            return response()->json([
                'message' => 'Rate limit test successful',
                'timestamp' => now()->toISOString(),
            ]);
        })->middleware('api.rate:5,1')->name('rate-limit'); // 5 requests per minute for testing
        
        // Test roles
        Route::get('admin-only', function () {
            return response()->json(['message' => 'Admin access granted']);
        })->middleware(['auth:sanctum', 'api.role:admin'])->name('admin-only');
        
        Route::get('editor-author', function () {
            return response()->json(['message' => 'Editor/Author access granted']);
        })->middleware(['auth:sanctum', 'api.role:editor,author'])->name('editor-author');
    });
}
```

#### **Bagian 4: Enhanced API Controllers dan Security Features (20 menit)**

##### Step 4.1: Create Comprehensive UserController
```bash
# Implement UserController dengan advanced features
nano app/Http/Controllers/Api/UserController.php
```

**Isi file app/Http/Controllers/Api/UserController.php:**
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
if (\App\Models\User::count() < 5) {
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
```bash
# Update PostApiController untuk include token abilities checking
nano app/Http/Controllers/Api/PostApiController.php
```

**Update method-method penting dalam PostApiController:**
```php
/**
 * Enhanced store method dengan ability checking
 */
public function store(Request $request)
{
    // Check token abilities
    if (!$request->user()->hasApiAbility('posts:create')) {
        return response()->json([
            'success' => false,
            'message' => 'Insufficient permissions to create posts',
            'required_ability' => 'posts:create'
        ], 403);
    }

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

    try {
        // Set author sebagai current user
        $validated['user_id'] = $request->user()->id;
        
        // Generate slug
        $validated['slug'] = \Str::slug($validated['title']);
        
        // Set published_at jika status published
        if ($validated['status'] === 'published') {
            $validated['published_at'] = now();
        }

        $post = Post::create($validated);

        // Attach tags jika ada
        if (!empty($validated['tags'])) {
            $post->tags()->attach($validated['tags']);
        }

        // Load relationships untuk response
        $post->load(['user:id,name', 'category:id,name', 'tags:id,name']);

        return response()->json([
            'success' => true,
            'message' => 'Post created successfully',
            'data' => $post
        ], 201);

    } catch (\Exception $e) {
        \Log::error('API Post creation failed', [
            'user_id' => $request->user()->id,
            'error' => $e->getMessage(),
            'data' => $validated
        ]);

        return response()->json([
            'success' => false,
            'message' => 'Failed to create post'
        ], 500);
    }
}

/**
 * Enhanced update method dengan ownership checking
 */
public function update(Request $request, Post $post)
{
    $user = $request->user();
    
    // Check ownership atau admin/editor permissions
    if ($post->user_id !== $user->id && 
        !in_array($user->role, ['admin', 'editor'])) {
        return response()->json([
            'success' => false,
            'message' => 'You can only edit your own posts',
            'error_code' => 'UNAUTHORIZED_POST_ACCESS'
        ], 403);
    }

    // Check token abilities
    if (!$user->hasApiAbility('posts:update')) {
        return response()->json([
            'success' => false,
            'message' => 'Insufficient permissions to update posts',
            'required_ability' => 'posts:update'
        ], 403);
    }

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

    try {
        // Update slug jika title berubah
        if ($post->title !== $validated['title']) {
            $validated['slug'] = \Str::slug($validated['title']);
            
            // Ensure uniqueness
            $originalSlug = $validated['slug'];
            $counter = 1;
            while (Post::where('slug', $validated['slug'])->where('id', '!=', $post->id)->exists()) {
                $validated['slug'] = $originalSlug . '-' . $counter;
                $counter++;
            }
        }

        // Handle publishing logic
        if ($validated['status'] === 'published' && $post->status !== 'published') {
            $validated['published_at'] = now();
        }

        $post->update($validated);

        // Sync tags
        if (array_key_exists('tags', $validated)) {
            $post->tags()->sync($validated['tags'] ?? []);
        }

        $post->load(['user:id,name', 'category:id,name', 'tags:id,name']);

        return response()->json([
            'success' => true,
            'message' => 'Post updated successfully',
            'data' => $post
        ]);

    } catch (\Exception $e) {
        return response()->json([
            'success' => false,
            'message' => 'Failed to update post'
        ], 500);
    }
}

/**
 * Enhanced destroy method dengan authorization
 */
public function destroy(Request $request, Post $post)
{
    $user = $request->user();
    
    // Check ownership atau admin/editor permissions
    if ($post->user_id !== $user->id && 
        !in_array($user->role, ['admin', 'editor'])) {
        return response()->json([
            'success' => false,
            'message' => 'You can only delete your own posts'
        ], 403);
    }

    // Check token abilities
    if (!$user->hasApiAbility('posts:delete')) {
        return response()->json([
            'success' => false,
            'message' => 'Insufficient permissions to delete posts',
            'required_ability' => 'posts:delete'
        ], 403);
    }

    try {
        $title = $post->title;
        
        // Soft delete dengan cleanup
        $post->tags()->detach();
        $post->delete();

        return response()->json([
            'success' => true,
            'message' => "Post '{$title}' deleted successfully"
        ]);

    } catch (\Exception $e) {
        return response()->json([
            'success' => false,
            'message' => 'Failed to delete post'
        ], 500);
    }
}

/**
 * Publish post (Editor/Admin only)
 */
public function publish(Request $request, Post $post)
{
    if (!in_array($request->user()->role, ['admin', 'editor'])) {
        return response()->json([
            'success' => false,
            'message' => 'Only editors and admins can publish posts'
        ], 403);
    }

    $post->update([
        'status' => 'published',
        'published_at' => now()
    ]);

    return response()->json([
        'success' => true,
        'message' => 'Post published successfully',
        'data' => $post
    ]);
}

/**
 * Unpublish post (Editor/Admin only)
 */
public function unpublish(Request $request, Post $post)
{
    if (!in_array($request->user()->role, ['admin', 'editor'])) {
        return response()->json([
            'success' => false,
            'message' => 'Only editors and admins can unpublish posts'
        ], 403);
    }

    $post->update(['status' => 'draft']);

    return response()->json([
        'success' => true,
        'message' => 'Post unpublished successfully',
        'data' => $post
    ]);
}

/**
 * Get analytics data (Admin only)
 */
public function analytics(Request $request)
{
    $analytics = [
        'total_posts' => Post::count(),
        'published_posts' => Post::published()->count(),
        'draft_posts' => Post::where('status', 'draft')->count(),
        'archived_posts' => Post::where('status', 'archived')->count(),
        'posts_this_month' => Post::whereMonth('created_at', now()->month)->count(),
        'most_viewed_posts' => Post::orderBy('views_count', 'desc')->take(10)->get(['id', 'title', 'views_count']),
        'posts_by_category' => \DB::table('posts')
                                 ->join('categories', 'posts.category_id', '=', 'categories.id')
                                 ->select('categories.name', \DB::raw('COUNT(*) as count'))
                                 ->groupBy('categories.name')
                                 ->orderBy('count', 'desc')
                                 ->get(),
        'posts_by_author' => \DB::table('posts')
                               ->join('users', 'posts.user_id', '=', 'users.id')
                               ->select('users.name', \DB::raw('COUNT(*) as count'))
                               ->groupBy('users.name')
                               ->orderBy('count', 'desc')
                               ->get(),
    ];

    return response()->json([
        'success' => true,
        'data' => $analytics,
        'generated_at' => now()->toISOString()
    ]);
}

/**
 * Get moderation queue (Editor/Admin only)
 */
public function moderationQueue(Request $request)
{
    $posts = Post::where('status', 'draft')
                 ->with(['user:id,name', 'category:id,name'])
                 ->orderBy('created_at', 'asc')
                 ->paginate(20);

    return response()->json([
        'success' => true,
        'data' => $posts->items(),
        'pagination' => [
            'current_page' => $posts->currentPage(),
            'last_page' => $posts->lastPage(),
            'total' => $posts->total(),
        ]
    ]);
}
```

#### **Bagian 5: API Testing & Documentation (15 menit)**

##### Step 5.1: Create API Testing Script
```bash
# Buat script untuk testing API endpoints
mkdir -p tests/api
nano tests/api/test-api.sh
```

**Isi file tests/api/test-api.sh:**
```bash
#!/bin/bash

# API Testing Script untuk Praktikum Cloud Computing ITK Week 6
# Usage: ./test-api.sh

BASE_URL="http://localhost:8000/api"
TOKEN=""
USER_ID=""

echo "=== API Testing - Week 6 Authentication & Authorization ==="
echo "Base URL: $BASE_URL"
echo ""

# Colors untuk output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function untuk print test results
print_result() {
    if [ $1 -eq 0 ]; then
        echo -e "${GREEN}✓ $2${NC}"
    else
        echo -e "${RED}✗ $2${NC}"
    fi
}

# Function untuk extract token dari response
extract_token() {
    echo "$1" | jq -r '.data.token.access_token' 2>/dev/null
}

# Function untuk extract user ID dari response
extract_user_id() {
    echo "$1" | jq -r '.data.user.id' 2>/dev/null
}

echo "1. Testing API Info Endpoint (Public)"
INFO_RESPONSE=$(curl -s "$BASE_URL/info")
INFO_STATUS=$?
print_result $INFO_STATUS "API Info endpoint"

if [ $INFO_STATUS -eq 0 ]; then
    echo "   Application: $(echo $INFO_RESPONSE | jq -r '.application')"
    echo "   API Version: $(echo $INFO_RESPONSE | jq -r '.api_version')"
fi
echo ""

echo "2. Testing User Registration"
REGISTER_DATA='{
    "name": "Test API User",
    "email": "testapi@example.com",
    "password": "password123",
    "password_confirmation": "password123",
    "role": "author"
}'

REGISTER_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/register" \
    -H "Content-Type: application/json" \
    -d "$REGISTER_DATA")
REGISTER_STATUS=$?
print_result $REGISTER_STATUS "User registration"

if [ $REGISTER_STATUS -eq 0 ]; then
    TOKEN=$(extract_token "$REGISTER_RESPONSE")
    USER_ID=$(extract_user_id "$REGISTER_RESPONSE")
    echo "   Token: ${TOKEN:0:20}..."
    echo "   User ID: $USER_ID"
fi
echo ""

echo "3. Testing User Login"
LOGIN_DATA='{
    "email": "admin@test.com",
    "password": "password123",
    "device_name": "API Test Device"
}'

LOGIN_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/login" \
    -H "Content-Type: application/json" \
    -d "$LOGIN_DATA")
LOGIN_STATUS=$?
print_result $LOGIN_STATUS "User login"

if [ $LOGIN_STATUS -eq 0 ]; then
    ADMIN_TOKEN=$(extract_token "$LOGIN_RESPONSE")
    echo "   Admin Token: ${ADMIN_TOKEN:0:20}..."
fi
echo ""

echo "4. Testing Authenticated Endpoint - Get User Profile"
if [ ! -z "$TOKEN" ]; then
    ME_RESPONSE=$(curl -s -X GET "$BASE_URL/auth/me" \
        -H "Authorization: Bearer $TOKEN")
    ME_STATUS=$?
    print_result $ME_STATUS "Get user profile"
    
    if [ $ME_STATUS -eq 0 ]; then
        echo "   User: $(echo $ME_RESPONSE | jq -r '.data.user.name')"
        echo "   Role: $(echo $ME_RESPONSE | jq -r '.data.user.role')"
    fi
else
    echo -e "${YELLOW}⚠ Skipping - No token available${NC}"
fi
echo ""

echo "5. Testing Posts API - Create Post"
if [ ! -z "$TOKEN" ]; then
    POST_DATA='{
        "title": "API Test Post",
        "content": "This is a test post created via API for Week 6 testing",
        "status": "draft",
        "category_id": 1
    }'
    
    CREATE_POST_RESPONSE=$(curl -s -X POST "$BASE_URL/posts" \
        -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d "$POST_DATA")
    CREATE_POST_STATUS=$?
    print_result $CREATE_POST_STATUS "Create post via API"
    
    if [ $CREATE_POST_STATUS -eq 0 ]; then
        POST_ID=$(echo $CREATE_POST_RESPONSE | jq -r '.data.id')
        echo "   Post ID: $POST_ID"
        echo "   Title: $(echo $CREATE_POST_RESPONSE | jq -r '.data.title')"
    fi
else
    echo -e "${YELLOW}⚠ Skipping - No token available${NC}"
fi
echo ""

echo "6. Testing Rate Limiting"
echo "   Making 5 rapid requests to test rate limiting..."
for i in {1..5}; do
    RATE_RESPONSE=$(curl -s -w "%{http_code}" "$BASE_URL/test/rate-limit" -o /dev/null)
    if [ "$RATE_RESPONSE" = "429" ]; then
        echo -e "   ${GREEN}✓ Rate limiting triggered on request $i${NC}"
        break
    elif [ $i -eq 5 ]; then
        echo -e "   ${YELLOW}⚠ Rate limiting not triggered (may be configured differently)${NC}"
    fi
done
echo ""

echo "7. Testing Role-Based Access Control"
if [ ! -z "$TOKEN" ]; then
    # Test admin-only endpoint dengan non-admin token
    ADMIN_RESPONSE=$(curl -s -w "%{http_code}" "$BASE_URL/test/admin-only" \
        -H "Authorization: Bearer $TOKEN" -o /dev/null)
    
    if [ "$ADMIN_RESPONSE" = "403" ]; then
        print_result 0 "Role-based access control (non-admin blocked)"
    else
        print_result 1 "Role-based access control (should block non-admin)"
    fi
    
    # Test dengan admin token jika ada
    if [ ! -z "$ADMIN_TOKEN" ]; then
        ADMIN_ACCESS_RESPONSE=$(curl -s -w "%{http_code}" "$BASE_URL/test/admin-only" \
            -H "Authorization: Bearer $ADMIN_TOKEN" -o /dev/null)
        
        if [ "$ADMIN_ACCESS_RESPONSE" = "200" ]; then
            print_result 0 "Admin access granted"
        else
            print_result 1 "Admin access (should allow admin)"
        fi
    fi
else
    echo -e "${YELLOW}⚠ Skipping - No token available${NC}"
fi
echo ""

echo "8. Testing Token Sessions Management"
if [ ! -z "$TOKEN" ]; then
    SESSIONS_RESPONSE=$(curl -s -X GET "$BASE_URL/auth/sessions" \
        -H "Authorization: Bearer $TOKEN")
    SESSIONS_STATUS=$?
    print_result $SESSIONS_STATUS "Get active sessions"
    
    if [ $SESSIONS_STATUS -eq 0 ]; then
        SESSIONS_COUNT=$(echo $SESSIONS_RESPONSE | jq -r '.data.total_count')
        echo "   Active sessions: $SESSIONS_COUNT"
    fi
fi
echo ""

echo "9. Testing Logout"
if [ ! -z "$TOKEN" ]; then
    LOGOUT_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/logout" \
        -H "Authorization: Bearer $TOKEN")
    LOGOUT_STATUS=$?
    print_result $LOGOUT_STATUS "User logout"
    
    # Test token setelah logout
    if [ $LOGOUT_STATUS -eq 0 ]; then
        AFTER_LOGOUT_RESPONSE=$(curl -s -w "%{http_code}" "$BASE_URL/auth/me" \
            -H "Authorization: Bearer $TOKEN" -o /dev/null)
        
        if [ "$AFTER_LOGOUT_RESPONSE" = "401" ]; then
            print_result 0 "Token invalidated after logout"
        else
            print_result 1 "Token should be invalidated after logout"
        fi
    fi
fi
echo ""

echo "=== API Testing Complete ==="
echo ""
echo "Notes:"
echo "- Ensure Laravel dev server is running on localhost:8000"
echo "- Check logs in storage/logs/laravel.log for detailed API activity"
echo "- Review database tables: users, personal_access_tokens, posts"
```

```bash
# Make script executable
chmod +x tests/api/test-api.sh
```

##### Step 5.2: Create API Documentation
```bash
# Buat dokumentasi API lengkap
nano public/api-docs.html
```

**Isi file public/api-docs.html:**
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

            <div class="bg-blue-50 p-4 rounded-lg mt-4">
                <h4 class="font-semibold mb-2">Authentication Header</h4>
                <code class="bg-white px-3 py-2 rounded block">Authorization: Bearer YOUR_TOKEN_HERE</code>
            </div>
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
                <p class="text-sm mb-2">Use the provided test script:</p>
                <code class="bg-white px-3 py-2 rounded block">./tests/api/test-api.sh</code>
            </div>
        </section>
    </div>
</body>
</html>
```

##### Step 5.3: Run Comprehensive Testing
```bash
# Start Laravel development server
php artisan serve --host=0.0.0.0 --port=8000 &

# Run API test script
./tests/api/test-api.sh

# Test dengan manual cURL commands
echo "=== Manual API Testing ==="

# Test 1: API Info
curl -s http://localhost:8000/api/info | jq .

# Test 2: Register User
curl -s -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Manual Test User",
    "email": "manual@test.com",
    "password": "password123",
    "password_confirmation": "password123",
    "role": "author"
  }' | jq .

# Test 3: Login and extract token
TOKEN=$(curl -s -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@test.com",
    "password": "password123"
  }' | jq -r '.data.token.access_token')

echo "Extracted token: ${TOKEN:0:20}..."

# Test 4: Authenticated request
curl -s -X GET http://localhost:8000/api/auth/me \
  -H "Authorization: Bearer $TOKEN" | jq .

# Test 5: Create post
curl -s -X POST http://localhost:8000/api/posts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Test API Post",
    "content": "This post was created via API testing",
    "status": "draft",
    "category_id": 1
  }' | jq .

### 🧪 TESTING & VERIFIKASI

#### Test 1: Sanctum Installation & Configuration Verification
```bash
echo "=== Sanctum Installation & Configuration Testing ==="

# Test 1.1: Verify Sanctum package installation
composer show laravel/sanctum && echo "✓ Sanctum package installed"

# Test 1.2: Check Sanctum configuration
if [ -f "config/sanctum.php" ]; then
    echo "✓ Sanctum config file exists"
    php artisan config:show sanctum.expiration && echo "✓ Token expiration configured"
else
    echo "✗ Sanctum config file missing"
fi

# Test 1.3: Verify personal_access_tokens table
php artisan tinker --execute="
if (\Schema::hasTable('personal_access_tokens')) {
    echo '✓ personal_access_tokens table exists' . PHP_EOL;
    \$columns = \Schema::getColumnListing('personal_access_tokens');
    \$required = ['id', 'tokenable_type', 'tokenable_id', 'name', 'token', 'abilities', 'last_used_at', 'expires_at'];
    \$missing = array_diff(\$required, \$columns);
    if (empty(\$missing)) {
        echo '✓ All required columns present' . PHP_EOL;
    } else {
        echo '✗ Missing columns: ' . implode(', ', \$missing) . PHP_EOL;
    }
} else {
    echo '✗ personal_access_tokens table missing' . PHP_EOL;
}
"

# Test 1.4: Check middleware registration
php artisan route:list | grep -q "sanctum" && echo "✓ Sanctum middleware registered"
```

#### Test 2: Authentication Flow Testing
```bash
echo "=== Authentication Flow Testing ==="

# Test 2.1: User model Sanctum integration
php artisan tinker --execute="
\$user = \App\Models\User::first();
if (\$user) {
    // Test HasApiTokens trait methods
    \$methods = get_class_methods(\$user);
    \$required = ['createToken', 'tokens', 'currentAccessToken'];
    \$hasAll = true;
    foreach (\$required as \$method) {
        if (!in_array(\$method, \$methods)) {
            echo '✗ Missing method: ' . \$method . PHP_EOL;
            \$hasAll = false;
        }
    }
    if (\$hasAll) {
        echo '✓ User model has all Sanctum methods' . PHP_EOL;
    }
    
    // Test token creation
    try {
        \$token = \$user->createToken('test-token', ['posts:read']);
        echo '✓ Token creation working' . PHP_EOL;
        
        // Test token abilities
        if (\$token->accessToken->can('posts:read')) {
            echo '✓ Token abilities working' . PHP_EOL;
        }
        
        // Cleanup test token
        \$token->accessToken->delete();
        echo '✓ Token cleanup successful' . PHP_EOL;
    } catch (\Exception \$e) {
        echo '✗ Token creation failed: ' . \$e->getMessage() . PHP_EOL;
    }
} else {
    echo '✗ No users found for testing' . PHP_EOL;
}
"

# Test 2.2: API endpoints response format
curl -s http://localhost:8000/api/info | jq -e '.api_version' >/dev/null && echo "✓ API info endpoint working"

# Test 2.3: Authentication endpoints
echo "Testing registration endpoint..."
REGISTER_RESPONSE=$(curl -s -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Verification User",
    "email": "verify@test.com",
    "password": "password123",
    "password_confirmation": "password123"
  }')

if echo "$REGISTER_RESPONSE" | jq -e '.success' >/dev/null 2>&1; then
    echo "✓ Registration endpoint working"
    VERIFY_TOKEN=$(echo "$REGISTER_RESPONSE" | jq -r '.data.token.access_token')
    
    # Test authenticated endpoint
    AUTH_TEST=$(curl -s -X GET http://localhost:8000/api/auth/me \
      -H "Authorization: Bearer $VERIFY_TOKEN")
    
    if echo "$AUTH_TEST" | jq -e '.success' >/dev/null 2>&1; then
        echo "✓ Bearer token authentication working"
    else
        echo "✗ Bearer token authentication failed"
    fi
else
    echo "✗ Registration endpoint failed"
fi
```

#### Test 3: Role-Based Authorization Testing
```bash
echo "=== Role-Based Authorization Testing ==="

# Test 3.1: Middleware functionality
php artisan tinker --execute="
// Test CheckApiRole middleware exists and has correct methods
if (class_exists('\App\Http\Middleware\CheckApiRole')) {
    echo '✓ CheckApiRole middleware exists' . PHP_EOL;
    
    \$middleware = new \App\Http\Middleware\CheckApiRole();
    if (method_exists(\$middleware, 'handle')) {
        echo '✓ Middleware has handle method' . PHP_EOL;
    }
} else {
    echo '✗ CheckApiRole middleware missing' . PHP_EOL;
}

// Test ApiRateLimit middleware
if (class_exists('\App\Http\Middleware\ApiRateLimit')) {
    echo '✓ ApiRateLimit middleware exists' . PHP_EOL;
} else {
    echo '✗ ApiRateLimit middleware missing' . PHP_EOL;
}
"

# Test 3.2: Route protection
echo "Testing protected routes..."

# Test admin-only endpoint without authentication
UNAUTH_RESPONSE=$(curl -s -w "%{http_code}" http://localhost:8000/api/admin/dashboard -o /dev/null)
if [ "$UNAUTH_RESPONSE" = "401" ]; then
    echo "✓ Admin routes protected from unauthenticated access"
else
    echo "✗ Admin routes should return 401 for unauthenticated users"
fi

# Test with non-admin user (if token available)
if [ ! -z "$VERIFY_TOKEN" ]; then
    NONAUTH_RESPONSE=$(curl -s -w "%{http_code}" http://localhost:8000/api/admin/dashboard \
      -H "Authorization: Bearer $VERIFY_TOKEN" -o /dev/null)
    if [ "$NONAUTH_RESPONSE" = "403" ]; then
        echo "✓ Admin routes protected from non-admin users"
    else
        echo "✗ Admin routes should return 403 for non-admin users"
    fi
fi
```

#### Test 4: API Rate Limiting Testing
```bash
echo "=== API Rate Limiting Testing ==="

# Test 4.1: Rate limit headers
RATE_RESPONSE=$(curl -s -I http://localhost:8000/api/info)
if echo "$RATE_RESPONSE" | grep -q "X-RateLimit-Limit"; then
    echo "✓ Rate limit headers present"
    echo "$RATE_RESPONSE" | grep "X-RateLimit"
else
    echo "✗ Rate limit headers missing"
fi

# Test 4.2: Rate limit enforcement (test endpoint)
if curl -s http://localhost:8000/api/test/rate-limit >/dev/null 2>&1; then
    echo "Testing rate limit enforcement..."
    for i in {1..8}; do
        RESPONSE_CODE=$(curl -s -w "%{http_code}" http://localhost:8000/api/test/rate-limit -o /dev/null)
        if [ "$RESPONSE_CODE" = "429" ]; then
            echo "✓ Rate limiting triggered after $i requests"
            break
        elif [ $i -eq 8 ]; then
            echo "⚠ Rate limiting not triggered (might be configured for higher limits)"
        fi
        sleep 0.1
    done
else
    echo "⚠ Rate limit test endpoint not available"
fi
```

#### Test 5: API Security & Performance Testing
```bash
echo "=== API Security & Performance Testing ==="

# Test 5.1: CSRF protection for API routes
CSRF_RESPONSE=$(curl -s -w "%{http_code}" -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test","password":"test"}' -o /dev/null)

if [ "$CSRF_RESPONSE" != "419" ]; then
    echo "✓ API routes properly exempt from CSRF protection"
else
    echo "✗ API routes incorrectly requiring CSRF tokens"
fi

# Test 5.2: Content-Type validation
INVALID_CONTENT=$(curl -s -w "%{http_code}" -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: text/plain" \
  -d "invalid data" -o /dev/null)

echo "Content-Type validation response: $INVALID_CONTENT"

# Test 5.3: Large payload handling
echo "Testing large payload handling..."
LARGE_PAYLOAD=$(printf '{"name":"%*s","email":"large@test.com","password":"password123","password_confirmation":"password123"}' 1000 'A')
LARGE_RESPONSE=$(curl -s -w "%{http_code}" -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d "$LARGE_PAYLOAD" -o /dev/null)

echo "Large payload response: $LARGE_RESPONSE"

# Test 5.4: Response time measurement
echo "Measuring API response times..."
for endpoint in "/api/info" "/api/public/stats"; do
    if curl -s "$endpoint" >/dev/null 2>&1; then
        TIME=$(curl -s -w "%{time_total}" http://localhost:8000$endpoint -o /dev/null)
        echo "  $endpoint: ${TIME}s"
    fi
done
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Sanctum Installation Issues
**Gejala:** `Class 'Laravel\Sanctum\...' not found` errors
**Solusi:**
```bash
# Verify Composer installation
composer show laravel/sanctum

# If not installed, install manually
composer require laravel/sanctum

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
grep -r "ApiRateLimit" app/Http/Kernel.php

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

# Verify middleware order in Kernel.php
grep -A 10 -B 5 "api.role" app/Http/Kernel.php

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

#### ✅ Sanctum Installation & Configuration
- [ ] Screenshot `composer show laravel/sanctum` menampilkan package installed
- [ ] Config file `config/sanctum.php` dengan custom settings
- [ ] Screenshot migration `personal_access_tokens` table berhasil
- [ ] Environment variables configured dalam `.env` file

#### ✅ Authentication System Implementation
- [ ] AuthController dengan comprehensive login/register/logout methods
- [ ] User model dengan HasApiTokens trait dan custom methods
- [ ] Screenshot token generation dan validation working
- [ ] API endpoints untuk authentication tested dan functional

#### ✅ Authorization & Role-Based Access Control
- [ ] CheckApiRole middleware dengan proper role checking
- [ ] ApiRateLimit middleware dengan dynamic rate limiting
- [ ] Screenshot role-based access working (admin vs user permissions)
- [ ] Route groups dengan proper middleware applied

#### ✅ Enhanced API Controllers
- [ ] UserController dengan CRUD operations dan profile management
- [ ] PostApiController dengan ownership checking dan authorization
- [ ] Screenshot API responses dengan proper JSON format
- [ ] File upload functionality working untuk avatars

#### ✅ API Security Features
- [ ] Rate limiting working dengan proper headers dan enforcement
- [ ] Token abilities system implemented dan tested
- [ ] Screenshot CSRF protection exempt untuk API routes
- [ ] Security logging implemented untuk audit trails

#### ✅ Testing & Documentation
- [ ] API testing script (`test-api.sh`) running successfully
- [ ] API documentation (`api-docs.html`) comprehensive dan accessible
- [ ] Screenshot all authentication flows tested via cURL
- [ ] Performance testing results dan optimization implemented

#### ✅ Advanced Features Implementation
- [ ] Multi-device token management (sessions endpoint)
- [ ] Token expiration handling dengan configurable timeouts
- [ ] Bulk operations dengan proper authorization
- [ ] Analytics endpoints dengan admin-only access

**Format Submission:**
```bash
cd ~/praktikum-cc
mkdir -p submission/week6/{controllers,middleware,config,tests,docs,screenshots}

# Copy implementation files
cp app/Http/Controllers/Api/AuthController.php submission/week6/controllers/
cp app/Http/Controllers/Api/UserController.php submission/week6/controllers/
cp app/Http/Controllers/Api/PostApiController.php submission/week6/controllers/
cp app/Http/Middleware/CheckApiRole.php submission/week6/middleware/
cp app/Http/Middleware/ApiRateLimit.php submission/week6/middleware/

# Copy configuration files
cp config/sanctum.php submission/week6/config/
cp routes/api.php submission/week6/routes-api.php
cp .env submission/week6/env-example.txt

# Copy testing and documentation
cp tests/api/test-api.sh submission/week6/tests/
cp public/api-docs.html submission/week6/docs/

# Export database schema
php artisan schema:dump --prune
cp database/schema/mysql-schema.sql submission/week6/database-schema.sql

# Create comprehensive README
cat > submission/week6/README.md << 'EOF'
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
./tests/api/test-api.sh > submission/week6/tests/final-test-results.txt 2>&1

# Generate API endpoint documentation
php artisan route:list --path=api --json > submission/week6/api-routes.json

# Export sample API responses
mkdir -p submission/week6/api-samples
curl -s http://localhost:8000/api/info > submission/week6/api-samples/info-response.json
curl -s http://localhost:8000/api/public/stats > submission/week6/api-samples/public-stats.json

**Format Submission:**
1. Buat folder submission/week6/
2. Masukkan semua screenshot dengan nama yang jelas
3. Buat file laporan dalam format Markdown
4. Commit dan push ke repository
5. Sertakan link commit terakhir