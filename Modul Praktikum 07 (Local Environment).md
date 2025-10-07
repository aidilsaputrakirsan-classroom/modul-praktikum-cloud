# WEEK 7: User Authentication & Authorization
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 7
- **Durasi**: 100 menit  
- **Topik**: Web-Based Authentication & Advanced Authorization System
- **Target**: Mahasiswa Semester 6
- **Platform**: Laptop/PC (Windows, macOS, atau Linux)

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Menerapkan session-based authentication (login, register, logout) dengan AuthController kustom
2. Mengkonfigurasi routes auth untuk guest dan proteksi dengan middleware `auth`
3. Mengembangkan tampilan login/register menggunakan layout guest berbasis Tailwind
4. Menambahkan helper role sederhana pada model `User` untuk kebutuhan UI/ops
6. Memvalidasi akses ke protected resource routes (posts, categories, tags, comments)

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1-6 telah completed dengan API authentication functional
- Laravel Sanctum sudah implemented dan working
- Database dengan user roles dan permissions structure
- Tailwind CSS sudah integrated dengan custom components


### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Custom Session Authentication (20 menit)**

##### Step 1.1: Implementasi AuthController (Login, Register, Logout)
```bash
# Generate AuthController dan siapkan views
php artisan make:controller Auth/AuthController
php artisan make:controller Auth/SessionBridgeController

# Verifikasi struktur Controllers
ls app\Http\Controllers\Auth 
```

**Update file app/Http/Controllers/Auth/AuthController.php**
```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;

class AuthController extends Controller
{
    public function showLoginForm()
    {
        return view('auth.login');
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required','email'],
            'password' => ['required','string'],
        ]);

        $remember = (bool) $request->boolean('remember');

        if (Auth::attempt($credentials, $remember)) {
            $request->session()->regenerate();
            return redirect()->intended(route('home'));
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }

    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        return redirect()->route('home');
    }

    public function showRegisterForm()
    {
        return view('auth.register');
    }

    public function register(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name' => ['required','string','max:255'],
            'email' => ['required','email','max:255','unique:users,email'],
            'password' => ['required','string','min:8','confirmed'],
        ]);

        if ($validator->fails()) {
            return back()->withErrors($validator)->withInput();
        }

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
            'role' => 'subscriber',
            'is_active' => true,
        ]);

        // Auto login after register
        Auth::login($user);

        return redirect()->route('home')
                         ->with('message', 'Registration successful.');
    }
}
```

**Update file app/Http/Controllers/Auth/SessionBridgeController.php**
```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Laravel\Sanctum\PersonalAccessToken;

class SessionBridgeController extends Controller
{
    /**
     * Attach a web session for the current Sanctum-authenticated API user.
     * Requires Authorization: Bearer <token> header (auth:sanctum).
     */
    public function attach(Request $request)
    {
        // If already authenticated via web guard, nothing to do
        if (Auth::check()) {
            $user = Auth::user();
            return response()->json([
                'success' => true,
                'message' => 'Session already active',
                'data' => [
                    'id' => $user->id,
                    'name' => $user->name,
                    'email' => $user->email,
                    'role' => $user->role,
                ],
            ]);
        }

        // Validate Bearer token manually (no auth:sanctum on this route)
        $bearer = $request->bearerToken();
        if (!$bearer) {
            return response()->json([
                'success' => false,
                'message' => 'Bearer token missing',
            ], 401);
        }

        $accessToken = PersonalAccessToken::findToken($bearer);
        if (!$accessToken || !$accessToken->tokenable) {
            return response()->json([
                'success' => false,
                'message' => 'Invalid or revoked token',
            ], 401);
        }

        // Optional: check token expiry if present
        if ($accessToken->expires_at && now()->greaterThan($accessToken->expires_at)) {
            return response()->json([
                'success' => false,
                'message' => 'Token expired',
            ], 401);
        }

        $user = $accessToken->tokenable;

        // Attach web session
        Auth::login($user, true);
        $request->session()->regenerate();
        $request->session()->put('api_token', $bearer);

        return response()->json([
            'success' => true,
            'message' => 'Session attached',
            'data' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'role' => $user->role,
            ],
        ]);
    }
}
```

##### Step 1.2: Konfigurasi Middleware dan Models
```bash
# Generate AuthController dan siapkan views
php artisan make:middleware CheckRole

# Verifikasi struktur Controllers
ls app\Http\Middleware 
```

**Update file app/Http/Middleware/CheckRole.php**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckRole
{
    /**
     * Handle an incoming request for web routes (session-based).
     */
    public function handle(Request $request, Closure $next, ...$roles)
    {
        $user = $request->user();

        if (!$user) {
            return redirect()->route('login');
        }

        if ($user->is_active === false) {
            return redirect()->route('home')->with('error', 'Your account is deactivated.');
        }

        // Normalize role parameters to support both comma and pipe delimiters
        $normalized = [];
        foreach ($roles as $r) {
            foreach (preg_split('/[\,\|]/', (string) $r) as $piece) {
                $piece = strtolower(trim($piece));
                if ($piece !== '') {
                    $normalized[] = $piece;
                }
            }
        }
        $roles = array_values(array_unique($normalized));

        if (!empty($roles) && !in_array(strtolower((string) $user->role), $roles, true)) {
            return redirect()->route('blog.index')
                             ->with('error', 'You do not have permission to access that page.');
        }

        return $next($request);
    }
}
```

**Update file app/Http/Middleware/CheckApiRole.php**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckApiRole
{
    /**
     * Handle incoming request dengan role-based authorization.
     */
    public function handle(Request $request, Closure $next, ...$roles)
    {
        if (!$request->user()) {
            return response()->json([
                'success' => false,
                'message' => 'Authentication required',
                'error_code' => 'UNAUTHENTICATED',
            ], 401);
        }

        $user = $request->user();

        if (!$user->is_active) {
            return response()->json([
                'success' => false,
                'message' => 'Account is deactivated',
                'error_code' => 'ACCOUNT_DEACTIVATED',
            ], 403);
        }

        // Normalize role parameters to support both comma and pipe delimiters
        $normalized = [];
        foreach ($roles as $r) {
            foreach (preg_split('/[\,\|]/', (string) $r) as $piece) {
                $piece = strtolower(trim($piece));
                if ($piece !== '') {
                    $normalized[] = $piece;
                }
            }
        }
        $roles = array_values(array_unique($normalized));

        if (!empty($roles) && !in_array(strtolower((string) $user->role), $roles, true)) {
            \Log::warning('Unauthorized role access attempt', [
                'user_id' => $user->id,
                'user_role' => $user->role,
                'required_roles' => $roles,
                'endpoint' => $request->path(),
                'method' => $request->method(),
                'ip' => $request->ip(),
            ]);

            return response()->json([
                'success' => false,
                'message' => 'Insufficient permissions for this action',
                'error_code' => 'INSUFFICIENT_PERMISSIONS',
                'required_roles' => $roles,
                'current_role' => $user->role,
            ], 403);
        }

        return $next($request);
    }
}
```

**Update file app/Http/Models/User.php**
```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
// Removed email verification requirement


class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    /**
     * Atribut yang dapat diisi secara mass assignment
     */
    protected $fillable = [
        'name',        // Nama lengkap user
        'email',       // Email untuk authentication
        'password',    // Password ter-hash
        'role',        // Role user (admin, editor, author, subscriber)
        'bio',         // Bio/deskripsi user
        'avatar',      // Path ke avatar image
        'is_active',   // Status aktif user
        'email_verified_at',
    ];

    /**
     * Atribut yang disembunyikan dari serialization
     */
    protected $hidden = [
        'password',         // Jangan expose password
        'remember_token',   // Jangan expose remember token
    ];

    /**
     * Casting atribut ke tipe data yang sesuai
     */
    protected $casts = [
        'email_verified_at' => 'datetime',  // Cast ke Carbon instance
        'password' => 'hashed',             // Auto-hash password saat assignment
        'is_active' => 'boolean',           // Cast ke boolean
    ];

    /**
     * Relationship: User has many Posts
     * Satu user dapat memiliki banyak posts
     */
    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    /**
     * Relationship: User has many Comments
     * Satu user dapat memiliki banyak comments
     */
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * Relationship: User has many Media uploads
     * Satu user dapat upload banyak media files
     */
    public function mediaUploads()
    {
        return $this->hasMany(Media::class, 'uploaded_by');
    }

    /**
     * Scope: Filter users by role
     * Usage: User::role('admin')->get()
     */
    public function scopeRole($query, $role)
    {
        return $query->where('role', $role);
    }

    /**
     * Scope: Filter active users only
     * Usage: User::active()->get()
     */
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    /**
     * Accessor: Get full avatar URL
     * Mengembalikan full URL untuk avatar image
     */
    public function getAvatarUrlAttribute()
    {
        if ($this->avatar) {
            return asset('storage/' . $this->avatar);
        }
        
        // Default avatar menggunakan Gravatar
        return 'https://www.gravatar.com/avatar/' . md5(strtolower($this->email)) . '?d=mp&s=150';
    }

    /**
     * Check if user has specific role
     * Usage: $user->hasRole('admin')
     */
    public function hasRole($role)
    {
        return $this->role === $role;
    }

    /**
     * Check if user can perform action based on role
     * Simple role-based authorization
     */
    public function canManagePosts()
    {
        return in_array($this->role, ['admin', 'editor', 'author']);
    }

    public function canManageUsers()
    {
        return $this->role === 'admin';
    }

    /**
     * Sanctum: Create token dengan custom abilities dan expiration
     */
    public function createApiToken(string $name, array $abilities = ['*'], \DateTimeInterface $expiresAt = null)
    {
        $token = $this->createToken($name, $abilities, $expiresAt);

        \Log::info('API token created', [
            'user_id' => $this->id,
            'token_name' => $name,
            'abilities' => $abilities,
            'expires_at' => $expiresAt?->format(DATE_ATOM),
            'created_at' => now()->toISOString(),
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
    public function revokeAllTokens(): int
    {
        $tokenCount = $this->tokens()->count();
        $this->tokens()->delete();

        \Log::info('All API tokens revoked', [
            'user_id' => $this->id,
            'revoked_count' => $tokenCount,
            'revoked_at' => now()->toISOString(),
        ]);

        return $tokenCount;
    }

    /**
     * Check if current token has specific API ability
     */
    public function hasApiAbility(string $ability): bool
    {
        $token = $this->currentAccessToken();
        if (!$token) {
            return false;
        }
        return in_array('*', $token->abilities ?? [], true) || in_array($ability, $token->abilities ?? [], true);
    }

    /**
     * Get current token info for diagnostics
     */
    public function getCurrentTokenInfo(): ?array
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

}
```

##### Step 1.3: Update `bootsrap\app.php`
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
            'role' => App\Http\Middleware\CheckRole::class,
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

##### Step 1.4: Konfigurasi Routes Authentication & Proteksi

**Update file routes/web.php**
```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\TagController;
use App\Http\Controllers\CommentController;
use App\Http\Controllers\Auth\AuthController;
use App\Http\Controllers\Auth\SessionBridgeController;

/*
|--------------------------------------------------------------------------
| Web Routes - Praktikum Cloud Computing ITK Week 4
|--------------------------------------------------------------------------
|
| Routes untuk CRUD operations dengan Laravel Resource Controllers
| Menggunakan RESTful conventions untuk consistent API design
|
*/

// Homepage route (dari week 2)
Route::get('/', function () {
    return view('welcome');
})->name('home');


/*
|--------------------------------------------------------------------------
| Resource Routes untuk CRUD Operations
|--------------------------------------------------------------------------
*/

// Authentication routes
Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'showLoginForm'])->name('login');
    Route::post('/login', [AuthController::class, 'login']);

    Route::get('/register', [AuthController::class, 'showRegisterForm'])->name('register');
    Route::post('/register', [AuthController::class, 'register']);
});

Route::post('/logout', [AuthController::class, 'logout'])->middleware('auth')->name('logout');

// Bridge: attach web session from API token (API-first auth flow)
// Attach web session using Bearer token from API login
// CSRF protected by web group; controller validates Bearer token
Route::post('/auth/attach-session', [SessionBridgeController::class, 'attach'])
    ->name('auth.attach-session');

// Protected content management routes (login + verified)
Route::middleware(['auth', 'verified'])->group(function () {
    // Posts management routes (admin/editor only)
    Route::resource('posts', PostController::class)->middleware('role:admin,editor')->names([
        'index' => 'posts.index',       // GET /posts - List all posts
        'create' => 'posts.create',     // GET /posts/create - Show create form
        'show' => 'posts.show',         // GET /posts/{post} - Show single post
        'edit' => 'posts.edit',         // GET /posts/{post}/edit - Show edit form
        'update' => 'posts.update',     // PUT/PATCH /posts/{post} - Update post
        'destroy' => 'posts.destroy',   // DELETE /posts/{post} - Delete post
    ]);

    // Categories management routes
    Route::resource('categories', CategoryController::class)->middleware('role:admin,editor')->except([
        'show' // Categories tidak memerlukan show page individual
    ]);

    // Tags management routes  
    Route::resource('tags', TagController::class)->middleware('role:admin,editor')->except([
        'show' // Tags tidak memerlukan show page individual
    ]);

    // Comments routes (nested dalam posts)
    Route::resource('posts.comments', CommentController::class)->middleware('role:admin,editor')->except([
        'index', 'show' // Comments di-handle dalam post show page
    ])->shallow(); // Shallow nesting untuk edit/update/delete
});

/*
|--------------------------------------------------------------------------
| Additional Routes untuk Enhanced Functionality
|--------------------------------------------------------------------------
*/

// Route untuk public post viewing (tanpa authentication)
Route::get('/blog', [PostController::class, 'publicIndex'])->name('blog.index');
Route::get('/blog/{post:slug}', [PostController::class, 'publicShow'])->name('blog.show');

// Route untuk category filtering
Route::get('/category/{category:slug}', [PostController::class, 'byCategory'])->name('posts.by-category');

// Route untuk tag filtering
Route::get('/tag/{tag:slug}', [PostController::class, 'byTag'])->name('posts.by-tag');

// Search functionality
Route::get('/search', [PostController::class, 'search'])->name('posts.search');

/*
|--------------------------------------------------------------------------
| Testing Routes untuk Development
|--------------------------------------------------------------------------
*/

// Route testing untuk development purposes
Route::get('/test-data', function () {
    return response()->json([
        'posts_count' => \App\Models\Post::count(),
        'categories_count' => \App\Models\Category::count(),
        'tags_count' => \App\Models\Tag::count(),
        'users_count' => \App\Models\User::count(),
        'latest_post' => \App\Models\Post::latest()->first()?->title ?? 'No posts yet',
        'timestamp' => now()->toISOString(),
    ]);
})->name('test-data');
```

#### **Bagian 2: Layouts Configuration (10 menit)**

##### Step 2.1: Create file resources/views/layouts/guest.blade.php
```php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }} - @yield('title', 'Authentication')</title>

    <!-- Fonts -->
    <link rel="preconnect" href="https://fonts.bunny.net">
    <link href="https://fonts.bunny.net/css?family=inter:400,500,600&display=swap" rel="stylesheet" />

    <!-- Scripts -->
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="font-sans text-gray-900 antialiased bg-gradient-to-br from-blue-50 via-white to-indigo-50 min-h-screen">
    <!-- Background Pattern -->
    <div class="absolute inset-0 bg-grid-pattern opacity-5"></div>
    
    <!-- Navigation Header -->
    <nav class="relative bg-white/80 backdrop-blur-sm border-b border-gray-200">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex justify-between h-16">
                <div class="flex items-center">
                    <a href="{{ route('home') }}" class="flex items-center space-x-2">
                        <div class="w-8 h-8 bg-gradient-to-r from-blue-600 to-indigo-600 rounded-lg flex items-center justify-center">
                            <span class="text-white font-bold text-sm">CC</span>
                        </div>
                        <h1 class="text-xl font-bold text-gradient">
                            Praktikum Cloud Computing ITK
                        </h1>
                    </a>
                </div>
                
                <div class="flex items-center space-x-4">
                    <a href="{{ route('blog.index') }}" 
                       class="text-gray-500 hover:text-gray-700 px-3 py-2 text-sm font-medium transition">
                        Blog
                    </a>
                    @if (Route::has('login'))
                        @auth
                            <a href="{{ route('dashboard') }}" 
                               class="btn-primary text-sm">Dashboard</a>
                        @else
                            <a href="{{ route('login') }}" 
                               class="text-gray-500 hover:text-gray-700 px-3 py-2 text-sm font-medium transition">
                               Log in
                            </a>
                            @if (Route::has('register'))
                                <a href="{{ route('register') }}" 
                                   class="btn-primary text-sm">Register</a>
                            @endif
                        @endauth
                    @endif
                </div>
            </div>
        </div>
    </nav>

    <!-- Main Content -->
    <div class="relative flex flex-col justify-center items-center min-h-[calc(100vh-4rem)] py-12">
        <div class="w-full sm:max-w-md px-6">
            <!-- Auth Card -->
            <div class="bg-white/80 backdrop-blur-sm shadow-xl rounded-2xl border border-white/20 p-8">
                <!-- Auth Header -->
                <div class="text-center mb-8">
                    <div class="w-16 h-16 bg-gradient-to-r from-blue-600 to-indigo-600 rounded-2xl flex items-center justify-center mx-auto mb-4">
                        <svg class="w-8 h-8 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 15v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2zm10-10V7a4 4 0 00-8 0v4h8z"/>
                        </svg>
                    </div>
                    @hasSection('auth-header')
                        @yield('auth-header')
                    @else
                        <h2 class="text-2xl font-bold text-gray-900">Welcome Back</h2>
                        <p class="text-gray-600 mt-2">Sign in to your account</p>
                    @endif
                </div>

                <!-- Auth Content -->
                {{ $slot ?? '' }}
                @yield('content')
            </div>

            <!-- Auth Footer -->
            <div class="text-center mt-6">
                <p class="text-sm text-gray-500">
                    © {{ date('Y') }} Praktikum Cloud Computing ITK. 
                    <a href="{{ route('home') }}" class="text-blue-600 hover:underline">Back to Home</a>
                </p>
            </div>
        </div>
    </div>

    <!-- Custom Styles -->
    <style>
        .bg-grid-pattern {
            background-image: radial-gradient(circle, #e5e7eb 1px, transparent 1px);
            background-size: 24px 24px;
        }
        
        .text-gradient {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            background-clip: text;
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
    </style>
</body>
</html>
```

##### Step 2.2: Update file resources/views/layouts/dashboard.blade.php
```php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>@yield('title', 'Dashboard') - {{ config('app.name') }}</title>

    <!-- Fonts -->
    <link rel="preconnect" href="https://fonts.bunny.net">
    <link href="https://fonts.bunny.net/css?family=inter:400,500,600&display=swap" rel="stylesheet" />

    <!-- Scripts dan Styles -->
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="font-sans antialiased bg-gray-50">
    <div class="min-h-screen flex">
        <!-- Sidebar -->
        <div class="w-64 bg-white shadow-sm border-r border-gray-200 fixed h-full">
            <div class="p-6 border-b border-gray-200">
                <h1 class="text-xl font-bold text-gradient">
                    Dashboard CC ITK
                </h1>
            </div>
            
            <nav class="mt-6">
                <div class="px-6 py-2">
                    <h3 class="text-xs font-semibold text-gray-500 uppercase tracking-wider">
                        Content Management
                    </h3>
                </div>
                
                <a href="{{ route('posts.index') }}" 
                   class="flex items-center px-6 py-3 text-gray-700 hover:bg-gray-50 hover:text-primary-600 transition {{ request()->routeIs('posts.*') ? 'bg-primary-50 text-primary-600 border-r-2 border-primary-600' : '' }}">
                    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
                    </svg>
                    Posts
                </a>
                
                <a href="{{ route('categories.index') }}" 
                   class="flex items-center px-6 py-3 text-gray-700 hover:bg-gray-50 hover:text-primary-600 transition {{ request()->routeIs('categories.*') ? 'bg-primary-50 text-primary-600 border-r-2 border-primary-600' : '' }}">
                    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 7h.01M7 3h5c.512 0 1.024.195 1.414.586l7 7a2 2 0 010 2.828l-7 7a2 2 0 01-2.828 0l-7-7A1.99 1.99 0 013 12V7a4 4 0 014-4z"/>
                    </svg>
                    Categories
                </a>
                
                <a href="{{ route('tags.index') }}" 
                   class="flex items-center px-6 py-3 text-gray-700 hover:bg-gray-50 hover:text-primary-600 transition {{ request()->routeIs('tags.*') ? 'bg-primary-50 text-primary-600 border-r-2 border-primary-600' : '' }}">
                    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 7h.01M7 3h5c.512 0 1.024.195 1.414.586l7 7a2 2 0 010 2.828l-7 7a2 2 0 01-2.828 0l-7-7A1.99 1.99 0 013 12V7a4 4 0 014-4z"/>
                    </svg>
                    Tags
                </a>
            </nav>
        </div>
        
        <!-- Main Content -->
        <div class="flex-1 ml-64">
            <!-- Top Navigation -->
            <header class="bg-white shadow-sm border-b border-gray-200">
                <div class="px-6 py-4">
                    <div class="flex justify-between items-center">
                        <div>
                            <h1 class="text-2xl font-bold text-gray-900">@yield('page-title')</h1>
                            @hasSection('page-description')
                                <p class="text-gray-600 mt-1">@yield('page-description')</p>
                            @endif
                        </div>
                        
                        <div class="flex items-center space-x-4">
                            <a href="{{ route('home') }}" class="btn-secondary">
                                View Site
                            </a>
                            <div class="h-8 w-8 bg-gray-300 rounded-full"></div>
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf
                                <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
                            </form>
                        </div>
                    </div>
                </div>
            </header>
            
            <!-- Page Content -->
            <main class="p-6">
                <!-- Alert Messages -->
                @if(session('success'))
                    <div class="mb-6 bg-green-50 border border-green-200 text-green-700 px-4 py-3 rounded-lg flex items-center" role="alert">
                        <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd"/>
                        </svg>
                        {{ session('success') }}
                    </div>
                @endif
                
                @if(session('error'))
                    <div class="mb-6 bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded-lg flex items-center" role="alert">
                        <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"/>
                        </svg>
                        {{ session('error') }}
                    </div>
                @endif

                @yield('content')
            </main>
        </div>
    </div>
</body>
</html>
```

##### Step 2.3: Update file resources/views/layouts/app.blade.php
```php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }} - @yield('title', 'Home')</title>

    <!-- Fonts -->
    <link rel="preconnect" href="https://fonts.bunny.net">
    <link href="https://fonts.bunny.net/css?family=inter:400,500,600&display=swap" rel="stylesheet" />

    <!-- Scripts dan Styles -->
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="font-sans antialiased bg-gray-50">
    <!-- Navigation Bar -->
    <nav class="bg-white shadow-sm border-b border-gray-200">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex justify-between h-16">
                <div class="flex items-center">
                    <!-- Logo -->
                    <div class="flex-shrink-0">
                        <h1 class="text-2xl font-bold text-gradient">
                            {{ config('app.name') }}
                        </h1>
                    </div>
                    
                    <!-- Navigation Links -->
                    <div class="hidden md:ml-10 md:flex md:space-x-8">
                        <a href="{{ route('home') }}" class="text-gray-900 hover:text-primary-600 px-3 py-2 text-sm font-medium transition">
                            Home
                        </a>
                        <a href="#" class="text-gray-500 hover:text-primary-600 px-3 py-2 text-sm font-medium transition">
                            About
                        </a>
                        <a href="#" class="text-gray-500 hover:text-primary-600 px-3 py-2 text-sm font-medium transition">
                            Contact
                        </a>
                    </div>
                </div>
                
                <!-- Right Side -->
                <div class="flex items-center space-x-4">
                    @auth
                        @php($role = auth()->user()->role ?? null)
                        @if($role === 'subscriber')
                            <span class="hidden sm:inline text-sm text-gray-600">Hi, {{ auth()->user()->name }}</span>
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf
                                <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
                            </form>
                        @else
                            <a href="{{ route('posts.index') }}" class="btn-secondary hidden sm:inline">Dashboard</a>
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf
                                <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
                            </form>
                        @endif
                    @else
                        <a href="{{ route('login') }}" class="text-gray-500 hover:text-primary-600 px-3 py-2 text-sm font-medium transition">Log in</a>
                        <a href="{{ route('register') }}" class="btn-primary">Get Started</a>
                    @endauth
                </div>
            </div>
        </div>
    </nav>

    <!-- Page Header -->
    @hasSection('header')
        <header class="bg-white shadow-sm">
            <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
                @yield('header')
            </div>
        </header>
    @endif

    <!-- Main Content -->
    <main class="py-8">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <!-- Alert Messages -->
            @if(session('success'))
                <div class="mb-6 bg-green-50 border border-green-200 text-green-700 px-4 py-3 rounded-lg" role="alert">
                    {{ session('success') }}
                </div>
            @endif
            
            @if(session('error'))
                <div class="mb-6 bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded-lg" role="alert">
                    {{ session('error') }}
                </div>
            @endif

            <!-- Page Content -->
            @yield('content')
        </div>
    </main>

    <!-- Footer -->
    <footer class="bg-white border-t border-gray-200 mt-12">
        <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
            <div class="text-center text-gray-500 text-sm">
                © {{ date('Y') }} Praktikum Cloud Computing - Institut Teknologi Kalimantan
            </div>
        </div>
    </footer>
</body>
</html>
```

#### **Bagian 3: Authentication Pages (10 menit)**

##### Step 3.1: Create file resources/views/auth/login.blade.php 
```php
@extends('layouts.guest')

@section('title', 'Login')

@section('auth-header')
    <h2 class="text-2xl font-bold text-gray-900">Sign In</h2>
    <p class="text-gray-600 mt-2">Welcome back! Please enter your details.</p>
@endsection

@section('content')
<!-- Session Status -->
<x-auth-session-status class="mb-4" :status="session('status')" />

<form id="loginForm" method="POST" action="{{ route('login') }}" class="space-y-6">
    @csrf

    <!-- Email Address -->
    <div>
        <x-input-label for="email" :value="__('Email')" class="text-sm font-medium text-gray-700" />
        <x-text-input id="email" 
                     class="mt-1 block w-full rounded-lg border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 transition" 
                     type="email" 
                     name="email" 
                     :value="old('email')" 
                     required 
                     autofocus 
                     autocomplete="username"
                     placeholder="Enter your email address" />
        <x-input-error :messages="$errors->get('email')" class="mt-2" />
    </div>

    <!-- Password -->
    <div>
        <div class="flex items-center justify-between">
            <x-input-label for="password" :value="__('Password')" class="text-sm font-medium text-gray-700" />
            @if (Route::has('password.request'))
                <a class="text-sm text-blue-600 hover:text-blue-500 hover:underline transition" 
                   href="{{ route('password.request') }}">
                    Forgot password?
                </a>
            @endif
        </div>
        <x-text-input id="password" 
                     class="mt-1 block w-full rounded-lg border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 transition"
                     type="password"
                     name="password"
                     required 
                     autocomplete="current-password"
                     placeholder="Enter your password" />
        <x-input-error :messages="$errors->get('password')" class="mt-2" />
    </div>

    <!-- Remember Me -->
    <div class="flex items-center">
        <input id="remember_me" 
               type="checkbox" 
               class="rounded border-gray-300 text-blue-600 shadow-sm focus:ring-blue-500" 
               name="remember">
        <label for="remember_me" class="ml-2 block text-sm text-gray-700">
            Remember me for 30 days
        </label>
    </div>

    <!-- Submit Button -->
    <div>
        <x-primary-button class="w-full justify-center py-3 text-base font-medium bg-gradient-to-r from-blue-600 to-indigo-600 hover:from-blue-700 hover:to-indigo-700 transition duration-200">
            {{ __('Sign In') }}
        </x-primary-button>
    </div>

    <!-- API Error Placeholder -->
    <div data-error class="hidden text-red-600 text-sm"></div>

    <!-- Social Login Placeholder -->
    <div class="relative">
        <div class="absolute inset-0 flex items-center">
            <div class="w-full border-t border-gray-300"></div>
        </div>
        <div class="relative flex justify-center text-sm">
            <span class="px-2 bg-white text-gray-500">Or continue with</span>
        </div>
    </div>

    <!-- Social Login Buttons (Placeholder) -->
    <div class="grid grid-cols-2 gap-3">
        <button type="button" 
                class="w-full inline-flex justify-center py-2 px-4 border border-gray-300 rounded-lg shadow-sm bg-white text-sm font-medium text-gray-500 hover:bg-gray-50 transition">
            <svg class="w-5 h-5" viewBox="0 0 24 24">
                <path fill="currentColor" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
                <path fill="currentColor" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
                <path fill="currentColor" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
                <path fill="currentColor" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
            </svg>
            <span class="ml-2">Google</span>
        </button>

        <button type="button" 
                class="w-full inline-flex justify-center py-2 px-4 border border-gray-300 rounded-lg shadow-sm bg-white text-sm font-medium text-gray-500 hover:bg-gray-50 transition">
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 24 24">
                <path d="M24 12.073c0-6.627-5.373-12-12-12s-12 5.373-12 12c0 5.99 4.388 10.954 10.125 11.854v-8.385H7.078v-3.47h3.047V9.43c0-3.007 1.792-4.669 4.533-4.669 1.312 0 2.686.235 2.686.235v2.953H15.83c-1.491 0-1.956.925-1.956 1.874v2.25h3.328l-.532 3.47h-2.796v8.385C19.612 23.027 24 18.062 24 12.073z"/>
            </svg>
            <span class="ml-2">Facebook</span>
        </button>
    </div>

    <!-- Register Link -->
    <div class="text-center">
        <p class="text-sm text-gray-600">
            Don't have an account? 
            <a href="{{ route('register') }}" class="font-medium text-blue-600 hover:text-blue-500 hover:underline transition">
                Sign up for free
            </a>
        </p>
    </div>
</form>
@endsection
```

##### Step 3.2: Create file resources/views/auth/register.blade.php
```php
@extends('layouts.guest')

@section('title', 'Register')

@section('auth-header')
    <h2 class="text-2xl font-bold text-gray-900">Create your account</h2>
    <p class="text-gray-600 mt-2">Join and start managing your posts.</p>
@endsection

@section('content')
<form id="registerForm" method="POST" action="{{ route('register') }}" class="space-y-6">
    @csrf

    <div>
        <x-input-label for="name" :value="__('Name')" class="text-sm font-medium text-gray-700" />
        <x-text-input id="name" name="name" type="text" class="mt-1 block w-full" 
                      :value="old('name')" required autofocus placeholder="Your full name" />
        <x-input-error :messages="$errors->get('name')" class="mt-2" />
    </div>

    <div>
        <x-input-label for="email" :value="__('Email')" class="text-sm font-medium text-gray-700" />
        <x-text-input id="email" name="email" type="email" class="mt-1 block w-full" 
                      :value="old('email')" required placeholder="you@example.com" />
        <x-input-error :messages="$errors->get('email')" class="mt-2" />
    </div>

    <div>
        <x-input-label for="password" :value="__('Password')" class="text-sm font-medium text-gray-700" />
        <x-text-input id="password" name="password" type="password" class="mt-1 block w-full" 
                      required autocomplete="new-password" placeholder="At least 8 characters" />
        <x-input-error :messages="$errors->get('password')" class="mt-2" />
    </div>

    <div>
        <x-input-label for="password_confirmation" :value="__('Confirm Password')" class="text-sm font-medium text-gray-700" />
        <x-text-input id="password_confirmation" name="password_confirmation" type="password" class="mt-1 block w-full" required />
    </div>

    <!-- Role selection (Admin via API is not allowed) -->
    <div>
        <x-input-label for="role" :value="__('Role (optional)')" class="text-sm font-medium text-gray-700" />
        <select id="role" name="role" class="mt-1 block w-full border border-gray-300 rounded-lg px-3 py-2">
            <option value="">Default: Subscriber</option>
            <option value="subscriber">Subscriber</option>
            <option value="author">Author</option>
            <option value="editor">Editor</option>
        </select>
        <p class="mt-1 text-xs text-gray-500">Admin tidak dapat register via API.</p>
    </div>

    <div>
        <x-primary-button class="w-full justify-center py-3 text-base">Create account</x-primary-button>
    </div>

    <!-- API Error Placeholder -->
    <div data-error class="hidden text-red-600 text-sm"></div>

    <div class="text-center">
        <p class="text-sm text-gray-600">
            Already have an account?
            <a href="{{ route('login') }}" class="font-medium text-blue-600 hover:underline">Sign in</a>
        </p>
    </div>
</form>
@endsection
```

##### **Bagian 4: Components Form Validation & JS (15 menit)**

**Create folder resources/views/components**
```php
### Tambahkan ke dalam components/auth-session-status.blade.php
@props(['status'])

@if ($status)
    <div {{ $attributes->merge(['class' => 'mb-4 font-medium text-sm text-green-600']) }}>
        {{ $status }}
    </div>
@endif


### Tambahkan ke dalam components/input-error.blade.php
@props(['messages'])
@if ($messages)
    <ul {{ $attributes->merge(['class' => 'text-sm text-red-600 space-y-1']) }}>
        @foreach ((array) $messages as $message)
            <li>{{ $message }}</li>
        @endforeach
    </ul>
@endif


### Tambahkan ke dalam components/input-label.blade.php
@props(['value'])
<label {{ $attributes->merge(['class' => 'block font-medium text-sm text-gray-700']) }}>
    {{ $value ?? $slot }}
</label>


### Tambahkan ke dalam components/primary-button.blade.php
<button {{ $attributes->merge(['type' => 'submit', 'class' => 'inline-flex items-center px-4 py-2 bg-blue-600 border border-transparent rounded-md font-semibold text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 transition']) }}>
    {{ $slot }}
</button>


### Tambahkan ke dalam components/text-input.blade.php
@props(['disabled' => false])
<input {{ $disabled ? 'disabled' : '' }} {!! $attributes->merge(['class' => 'rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 w-full']) !!}>
```

**Create file resources/js/auth.js**
```js
import axios from 'axios';

async function attachSession() {
  try {
    await axios.post('/auth/attach-session');
    return true;
  } catch (e) {
    const status = e?.response?.status;
    const msg = e?.response?.data?.message || e?.message || 'Attach session failed';
    console.error('Attach session failed', status, msg);
    return false;
  }
}

function setAuthToken(token, remember = false) {
  try {
    if (remember) {
      localStorage.setItem('api_access_token', token);
    } else {
      sessionStorage.setItem('api_access_token', token);
    }
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  } catch (e) {
    console.warn('Failed storing token', e);
  }
}

function clearAuthToken() {
  try {
    localStorage.removeItem('api_access_token');
    sessionStorage.removeItem('api_access_token');
    delete axios.defaults.headers.common['Authorization'];
  } catch {}
}

async function handleApiLogin(form) {
  const data = new FormData(form);
  const payload = {
    email: data.get('email'),
    password: data.get('password'),
    device_name: 'web-client',
    remember_me: data.get('remember') === 'on',
  };
  const submitBtn = form.querySelector('button[type="submit"], .btn-primary');
  submitBtn && (submitBtn.disabled = true);
  try {
    const res = await axios.post('/api/auth/login', payload);
    const token = res?.data?.data?.token?.access_token;
    const role = res?.data?.data?.user?.role;
    if (!token) throw new Error('Missing token');
    setAuthToken(token, !!payload.remember_me);
    const attached = await attachSession();
    if (!attached) {
      const errorBox = form.querySelector('[data-error]');
      if (errorBox) {
        errorBox.textContent = 'Login berhasil, tetapi gagal membuat sesi. Coba ulangi atau refresh halaman.';
        errorBox.classList.remove('hidden');
      } else {
        alert('Login berhasil, tetapi gagal membuat sesi. Coba ulangi atau refresh halaman.');
      }
      return; // stop redirect until session is attached
    }
    if (role === 'subscriber') {
      window.location.href = '/blog';
    } else {
      window.location.href = '/posts';
    }
  } catch (e) {
    const message = e?.response?.data?.message || 'Login failed';
    const errorBox = form.querySelector('[data-error]');
    if (errorBox) {
      errorBox.textContent = message;
      errorBox.classList.remove('hidden');
    } else {
      alert(message);
    }
  } finally {
    submitBtn && (submitBtn.disabled = false);
  }
}

async function handleApiRegister(form) {
  const data = new FormData(form);
  const payload = {
    name: data.get('name'),
    email: data.get('email'),
    password: data.get('password'),
    password_confirmation: data.get('password_confirmation'),
    role: (data.get('role') || undefined),
  };
  const submitBtn = form.querySelector('button[type="submit"], .btn-primary');
  submitBtn && (submitBtn.disabled = true);
  try {
    const res = await axios.post('/api/auth/register', payload);
    const token = res?.data?.data?.token?.access_token;
    const role = res?.data?.data?.user?.role;
    if (!token) throw new Error('Missing token');
    setAuthToken(token, false);
    const attached = await attachSession();
    if (!attached) {
      const errorBox = form.querySelector('[data-error]');
      if (errorBox) {
        errorBox.textContent = 'Registrasi berhasil, tetapi gagal membuat sesi. Coba ulangi atau refresh halaman.';
        errorBox.classList.remove('hidden');
      } else {
        alert('Registrasi berhasil, tetapi gagal membuat sesi. Coba ulangi atau refresh halaman.');
      }
      return;
    }
    if (role === 'subscriber') {
      window.location.href = '/blog';
    } else {
      window.location.href = '/posts';
    }
  } catch (e) {
    const message = e?.response?.data?.message || 'Registration failed';
    const errorBox = form.querySelector('[data-error]');
    if (errorBox) {
      errorBox.textContent = message;
      errorBox.classList.remove('hidden');
    } else {
      alert(message);
    }
  } finally {
    submitBtn && (submitBtn.disabled = false);
  }
}

async function handleApiLogout(anchorOrForm) {
  try {
    await axios.post('/api/auth/logout');
  } catch (e) {
    // ignore
  }
  clearAuthToken();
}

document.addEventListener('DOMContentLoaded', () => {
  const loginForm = document.getElementById('loginForm');
  if (loginForm) {
    loginForm.addEventListener('submit', (e) => {
      e.preventDefault();
      handleApiLogin(loginForm);
    });
  }

  const registerForm = document.getElementById('registerForm');
  if (registerForm) {
    registerForm.addEventListener('submit', (e) => {
      e.preventDefault();
      handleApiRegister(registerForm);
    });
  }

  // Intercept logout buttons with data-api-logout attribute
  document.querySelectorAll('[data-api-logout]')?.forEach((el) => {
    el.addEventListener('click', async (e) => {
      // if inside a form, prevent and do API logout then submit form to clear session
      const form = el.closest('form');
      if (form) {
        e.preventDefault();
        await handleApiLogout(form);
        form.submit();
      }
    });
  });
});
```

**Update file resources/js/app.js**
```js
import './bootstrap';
import './auth';
```

**Update file resources/js/bootstrap.js**
```js
import axios from 'axios';
window.axios = axios;

window.axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';

// CSRF for web POSTs
const csrf = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
if (csrf) {
  window.axios.defaults.headers.common['X-CSRF-TOKEN'] = csrf;
}

// Inject Authorization if token stored
const storedToken = (typeof window !== 'undefined') ? (localStorage.getItem('api_access_token') || sessionStorage.getItem('api_access_token')) : null;
if (storedToken) {
  window.axios.defaults.headers.common['Authorization'] = `Bearer ${storedToken}`;
}

```

**Update file resources/views/blog/category.blade.php**
```php
@extends('layouts.app')
@section('title', 'Category: '.$category->name)
@section('content')
  @auth
    @if(auth()->user()->role === 'subscriber')
      <div class="mb-4 flex justify-end">
        <form method="POST" action="{{ route('logout') }}">
          @csrf
          <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
        </form>
      </div>
    @endif
  @endauth
  <h1 class="text-2xl font-bold mb-4">Category: {{ $category->name }}</h1>
  @include('blog.partials.posts-grid', ['posts' => $posts])
@endsection
```

**Update file resources/views/blog/index.blade.php**
```php
@extends('layouts.app')
@section('title','Blog')
@section('content')
  @auth
    @if(auth()->user()->role === 'subscriber')
      <div class="mb-4 flex justify-end">
        <form method="POST" action="{{ route('logout') }}">
          @csrf
          <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
        </form>
      </div>
    @endif
  @endauth
  <div class="grid md:grid-cols-3 gap-6">
    @foreach($posts as $post)
      <a href="{{ route('blog.show', $post->slug) }}" class="card block hover:shadow-md transition">
        @if($post->featured_image)
          <img class="rounded-md mb-3 w-full h-40 object-cover" src="{{ Storage::url($post->featured_image) }}">
        @endif
        <h3 class="font-semibold text-lg">{{ $post->title }}</h3>
        <p class="text-sm text-gray-600 mt-1">
          {{ $post->category->name ?? 'Uncategorized' }} • {{ $post->published_at?->format('Y-m-d') }}
        </p>
        @if($post->excerpt)
          <p class="mt-2 text-gray-700">{{ Str::limit($post->excerpt, 120) }}</p>
        @endif
      </a>
    @endforeach
  </div>
  <div class="mt-6">{{ $posts->links() }}</div>
@endsection
```

**Update file resources/views/blog/search.blade.php**
```php
@extends('layouts.app')
@section('title','Search')
@section('content')
  @auth
    @if(auth()->user()->role === 'subscriber')
      <div class="mb-4 flex justify-end">
        <form method="POST" action="{{ route('logout') }}">
          @csrf
          <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
        </form>
      </div>
    @endif
  @endauth
  <form method="GET" action="{{ route('posts.search') }}" class="mb-4 flex gap-2">
    <input name="q" value="{{ $searchTerm }}" class="input-field flex-1" placeholder="Search posts...">
    <button class="btn-primary">Search</button>
  </form>

  @if($posts instanceof \Illuminate\Pagination\LengthAwarePaginator)
    @include('blog.partials.posts-grid', ['posts' => $posts])
    <div class="mt-6">{{ $posts->links() }}</div>
  @else
    <p class="text-gray-600">Masukkan kata kunci untuk mencari.</p>
  @endif
@endsection
```

**Update file resources/views/blog/show.blade.php**
```php
@extends('layouts.app')
@section('title', $post->title)
@section('content')
  @auth
    @if(auth()->user()->role === 'subscriber')
      <div class="mb-4 flex justify-end">
        <form method="POST" action="{{ route('logout') }}">
          @csrf
          <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
        </form>
      </div>
    @endif
  @endauth
  <article class="prose max-w-none card">
    <h1>{{ $post->title }}</h1>
    <p class="text-sm text-gray-600">
      {{ $post->category->name ?? 'Uncategorized' }} • by {{ $post->user->name ?? '-' }} •
      {{ $post->published_at?->format('Y-m-d H:i') }}
    </p>
    @if($post->featured_image)
      <img class="rounded-md my-4 w-full object-cover" src="{{ Storage::url($post->featured_image) }}">
    @endif
    @if(filled($post->excerpt))
      <div class="p-4 bg-gray-50 border rounded">{{ $post->excerpt }}</div>
    @endif
    <div class="mt-4">{!! nl2br(e($post->content)) !!}</div>

    @if($post->tags->count())
      <div class="mt-6 flex flex-wrap gap-2">
        @foreach($post->tags as $tag)
          <span class="px-2 py-1 text-xs rounded bg-blue-50 text-blue-700 border">#{{ $tag->name }}</span>
        @endforeach
      </div>
    @endif
  </article>
@endsection
```

**Update file resources/views/blog/tag.blade.php**
```php
@extends('layouts.app')
@section('title', 'Tag: '.$tag->name)
@section('content')
  @auth
    @if(auth()->user()->role === 'subscriber')
      <div class="mb-4 flex justify-end">
        <form method="POST" action="{{ route('logout') }}">
          @csrf
          <button type="submit" class="btn-secondary" data-api-logout>Logout</button>
        </form>
      </div>
    @endif
  @endauth
  <h1 class="text-2xl font-bold mb-4">Tag: #{{ $tag->name }}</h1>
  @include('blog.partials.posts-grid', ['posts' => $posts])
@endsection
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Breeze Installation Conflicts
**Gejala:** `Class 'Laravel\Breeze\...' not found` errors
**Solusi:**
```bash
# Remove existing auth scaffolding
php artisan breeze:install --force

# Clear all caches
php artisan config:clear
php artisan view:clear
php artisan route:clear
php artisan cache:clear

# Reinstall NPM dependencies
rm -rf node_modules package-lock.json
npm install
npm run build

# Check if Breeze service provider is registered
grep -r "BreezeServiceProvider" config/app.php
```

#### Problem 2: Policy Authorization Not Working
**Gejala:** Policies tidak ter-trigger atau selalu return false
**Solusi:**
```bash
# Check if policies are registered in AuthServiceProvider
grep -A 10 "protected \$policies" app/Providers/AuthServiceProvider.php

# Test policy registration
php artisan tinker --execute="
echo 'Registered policies:' . PHP_EOL;
\$policies = app(\Illuminate\Contracts\Auth\Access\Gate::class)->policies();
foreach (\$policies as \$model => \$policy) {
    echo '  ' . \$model . ' => ' . \$policy . PHP_EOL;
}
"

# Clear policy cache
php artisan cache:clear
php artisan config:cache

# Test specific policy method
php artisan tinker --execute="
\$user = \App\Models\User::first();
\$post = \App\Models\Post::first();

if (\$user && \$post) {
    try {
        \$result = \$user->can('update', \$post);
        echo 'Policy result: ' . (\$result ? 'Allowed' : 'Denied') . PHP_EOL;
        
        // Test policy response
        \$response = \Gate::forUser(\$user)->inspect('update', \$post);
        echo 'Policy response: ' . \$response->message() . PHP_EOL;
    } catch (\Exception \$e) {
        echo 'Policy error: ' . \$e->getMessage() . PHP_EOL;
    }
}
"
```

#### Problem 3: Dashboard Data Not Loading
**Gejala:** Dashboard menampilkan error atau data kosong
**Solusi:**
```bash
# Check database connections
php artisan tinker --execute="
try {
    \$count = \App\Models\User::count();
    echo 'Database connection: OK (' . \$count . ' users)' . PHP_EOL;
} catch (\Exception \$e) {
    echo 'Database error: ' . \$e->getMessage() . PHP_EOL;
}
"

# Test dashboard controller methods manually
php artisan tinker --execute "
\$user = \App\Models\User::first();
if (\$user) {
    echo 'Testing user statistics...' . PHP_EOL;
    echo 'User posts: ' . \$user->posts()->count() . PHP_EOL;
    echo 'User comments: ' . \$user->comments()->count() . PHP_EOL;
    echo 'User role: ' . \$user->role . PHP_EOL;
} else {
    echo 'No users found' . PHP_EOL;
}
"

# Check for missing relationships
php artisan tinker --execute="
\$user = \App\Models\User::first();
if (\$user) {
    // Test relationships exist
    try {
        \$posts = \$user->posts;
        echo '✓ User->posts relationship working' . PHP_EOL;
    } catch (\Exception \$e) {
        echo '✗ User->posts relationship error: ' . \$e->getMessage() . PHP_EOL;
    }
    
    try {
        \$comments = \$user->comments;
        echo '✓ User->comments relationship working' . PHP_EOL;
    } catch (\Exception \$e) {
        echo '✗ User->comments relationship error: ' . \$e->getMessage() . PHP_EOL;
    }
}
"

# Check view file for syntax errors
php artisan view:clear
php -l resources/views/dashboard.blade.php
```

#### Problem 4: Email Verification Not Working
**Gejala:** Verification emails tidak terkirim atau link tidak valid
**Solusi:**
```bash
# Test email configuration
php artisan tinker --execute="
try {
    \Mail::raw('Test email from Laravel', function (\$message) {
        \$message->to('test@example.com')->subject('Test Email');
    });
    echo 'Email configuration: OK' . PHP_EOL;
} catch (\Exception \$e) {
    echo 'Email error: ' . \$e->getMessage() . PHP_EOL;
}
"

# Check mail configuration
cat .env | grep MAIL

# Test verification URL generation
php artisan tinker --execute="
\$user = \App\Models\User::where('email_verified_at', null)->first();
if (\$user) {
    \$url = \URL::temporarySignedRoute(
        'verification.verify',
        now()->addMinutes(60),
        ['id' => \$user->id, 'hash' => sha1(\$user->email)]
    );
    echo 'Verification URL: ' . \$url . PHP_EOL;
} else {
    echo 'No unverified users found' . PHP_EOL;
}
"

# Check verification middleware
php artisan route:list | grep verification

# Test custom notification
php artisan tinker --execute="
\$user = \App\Models\User::first();
if (\$user) {
    try {
        \$user->sendEmailVerificationNotification();
        echo 'Verification email sent successfully' . PHP_EOL;
    } catch (\Exception \$e) {
        echo 'Verification email error: ' . \$e->getMessage() . PHP_EOL;
    }
}
"
```

#### Problem 5: Session Management Issues
**Gejala:** User ter-logout otomatis atau session tidak persistent
**Solusi:**
```bash
# Check session configuration
php artisan tinker --execute="
echo 'Session driver: ' . config('session.driver') . PHP_EOL;
echo 'Session lifetime: ' . config('session.lifetime') . ' minutes' . PHP_EOL;
echo 'Session cookie name: ' . config('session.cookie') . PHP_EOL;
echo 'Session secure: ' . (config('session.secure') ? 'Yes' : 'No') . PHP_EOL;
"

# Check session storage permissions
ls -la storage/framework/sessions/
chmod -R 775 storage/framework/sessions/

# Test session functionality
php artisan tinker --execute="
// Test session store
session(['test_key' => 'test_value']);
echo 'Session stored: ' . session('test_key') . PHP_EOL;

// Test session driver
try {
    \$store = app('session.store');
    echo 'Session store class: ' . get_class(\$store) . PHP_EOL;
} catch (\Exception \$e) {
    echo 'Session store error: ' . \$e->getMessage() . PHP_EOL;
}
"

# Check APP_KEY
grep APP_KEY .env
php artisan key:generate --show

# Clear session data
php artisan session:table 2>/dev/null || echo "Using file-based sessions"
rm -f storage/framework/sessions/*
```

#### Problem 6: Middleware Conflicts
**Gejala:** Authorization middleware tidak berjalan atau conflict dengan existing middleware
**Solusi:**
```bash
# Check middleware registration order
grep -A 20 "protected \$middlewareAliases" app/Http/Kernel.php

# Test middleware execution order
php artisan route:list | grep -E "(auth|verified|can)"

# Debug specific route middleware
php artisan route:list --name=dashboard

# Test middleware manually
php artisan tinker --execute="
\$request = \Illuminate\Http\Request::create('/dashboard', 'GET');
\$user = \App\Models\User::first();
\$request->setUserResolver(function() use (\$user) { return \$user; });

// Test auth middleware
\$authMiddleware = new \Illuminate\Auth\Middleware\Authenticate();
echo 'Auth middleware exists: Yes' . PHP_EOL;

// Test verified middleware  
\$verifiedMiddleware = new \Illuminate\Auth\Middleware\EnsureEmailIsVerified();
echo 'Email verified middleware exists: Yes' . PHP_EOL;
"

# Check for conflicting route definitions
grep -r "Route::get('/dashboard'" routes/
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

#### ✅ Custom Session Authentication
- [ ] Screenshot File AuthController dan SessionBridgeController
- [ ] Screenshots `routes/web.php` dengan Resource routes (posts, categories, tags, comments) terproteksi middleware auth

#### ✅ Authentication Pages
- [ ] Screenshot Login views sesuai layout guest
- [ ] Screenshot Register views sesuai layout guest

#### ✅ Components Form Validation
- [ ] Screenshot Register tidak valid untuk registered email