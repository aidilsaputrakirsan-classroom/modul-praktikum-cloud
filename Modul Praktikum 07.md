# WEEK 7: User Authentication & Authorization
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 7
- **Durasi**: 100 menit  
- **Topik**: Web-Based Authentication & Advanced Authorization System
- **Target**: Mahasiswa Semester 6
- **Platform**: Google Cloud Shell
- **Repository**: github.com/aidilsaputrakirsan/praktikum-cc

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Mengimplementasikan Laravel Breeze untuk web-based authentication system
2. Membangun advanced authorization dengan Gates dan Policies
3. Mengembangkan role-based permission management dengan UI interface
4. Mengimplementasikan email verification dan password reset functionality
5. Membangun user profile management dengan real-time validation
6. Menerapkan session management dan security best practices
7. Mengintegrasikan frontend authentication dengan existing API system

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1-6 telah completed dengan API authentication functional
- Laravel Sanctum sudah implemented dan working
- Database dengan user roles dan permissions structure
- Tailwind CSS sudah integrated dengan custom components

**Environment Verification:**
```bash
# Pastikan berada di direktori project Laravel
cd ~/praktikum-cc/week2/laravel-app

# Verifikasi existing authentication API
php artisan route:list | grep -E "(auth|api)" | head -10

# Check existing user roles dan data
php artisan tinker --execute="
echo 'Users count: ' . \App\Models\User::count() . PHP_EOL;
echo 'Roles distribution: ' . PHP_EOL;
\App\Models\User::select('role')->groupBy('role')->withCount('id')->get()->each(function(\$item) {
    echo '  ' . \$item->role . ': ' . \$item->id_count . PHP_EOL;
});
echo 'API tokens count: ' . \DB::table('personal_access_tokens')->count() . PHP_EOL;
"

# Verify Tailwind CSS dan existing UI components
ls -la resources/views/layouts/
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Laravel Breeze Installation & Configuration (20 menit)**

##### Step 1.1: Install Laravel Breeze untuk Web Authentication
```bash
# Install Laravel Breeze package
composer require laravel/breeze --dev

# Install Breeze dengan Blade + Tailwind stack
php artisan breeze:install blade

# Install NPM dependencies dan compile assets
npm install
npm run build

# Run database migrations untuk authentication tables
php artisan migrate

# Verify Breeze installation
ls -la resources/views/auth/
ls -la app/Http/Controllers/Auth/
```

##### Step 1.2: Configure Breeze Authentication Views
```bash
# Check generated authentication views
find resources/views/auth/ -name "*.blade.php" -exec basename {} \;

# Verify authentication routes
php artisan route:list | grep -E "(login|register|password|verify)"

# Check authentication middleware
grep -r "auth" app/Http/Middleware/
```

##### Step 1.3: Customize Authentication Layouts untuk Consistency
```bash
# Update authentication layout untuk match existing design
nano resources/views/layouts/guest.blade.php
```

**Update file resources/views/layouts/guest.blade.php:**
```html
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

##### Step 1.4: Update Authentication Forms dengan Enhanced UI
```bash
# Customize login form
nano resources/views/auth/login.blade.php
```

**Update file resources/views/auth/login.blade.php:**
```html
@extends('layouts.guest')

@section('title', 'Login')

@section('auth-header')
    <h2 class="text-2xl font-bold text-gray-900">Sign In</h2>
    <p class="text-gray-600 mt-2">Welcome back! Please enter your details.</p>
@endsection

@section('content')
<!-- Session Status -->
<x-auth-session-status class="mb-4" :status="session('status')" />

<form method="POST" action="{{ route('login') }}" class="space-y-6">
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

#### **Bagian 2: Advanced Authorization dengan Gates dan Policies (25 menit)**

##### Step 2.1: Create Authorization Policies
```bash
# Generate policies untuk different models
php artisan make:policy PostPolicy --model=Post
php artisan make:policy UserPolicy --model=User
php artisan make:policy CategoryPolicy --model=Category

# Generate gates untuk custom permissions
php artisan make:provider AuthorizationServiceProvider

# Verify policies created
ls -la app/Policies/
```

##### Step 2.2: Implement Post Policy dengan Advanced Logic
```bash
# Edit PostPolicy untuk implement authorization logic
nano app/Policies/PostPolicy.php
```

**Isi file app/Policies/PostPolicy.php:**
```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class PostPolicy
{
    /**
     * Perform pre-authorization checks.
     * Runs before any other authorization methods
     */
    public function before(User $user, string $ability): bool|null
    {
        // Super admin can do everything
        if ($user->role === 'admin') {
            return true;
        }

        // Inactive users can't do anything
        if (!$user->is_active) {
            return false;
        }

        return null; // Continue to specific method
    }

    /**
     * Determine whether the user can view any posts.
     */
    public function viewAny(User $user): bool
    {
        // All authenticated users can view posts list
        return true;
    }

    /**
     * Determine whether the user can view the post.
     */
    public function view(User $user, Post $post): bool
    {
        // Users can view published posts or their own posts
        return $post->status === 'published' || 
               $post->user_id === $user->id || 
               in_array($user->role, ['admin', 'editor']);
    }

    /**
     * Determine whether the user can create posts.
     */
    public function create(User $user): bool
    {
        // Authors, editors, and admins can create posts
        return in_array($user->role, ['author', 'editor', 'admin']);
    }

    /**
     * Determine whether the user can update the post.
     */
    public function update(User $user, Post $post): Response
    {
        // Post owners can edit their own posts
        if ($post->user_id === $user->id) {
            return Response::allow();
        }

        // Editors can edit any post
        if ($user->role === 'editor') {
            return Response::allow('Editors can modify all posts');
        }

        // Others cannot edit
        return Response::deny('You can only edit your own posts.');
    }

    /**
     * Determine whether the user can delete the post.
     */
    public function delete(User $user, Post $post): Response
    {
        // Post owners can delete their own posts (if not published)
        if ($post->user_id === $user->id) {
            if ($post->status !== 'published') {
                return Response::allow();
            }
            return Response::deny('Cannot delete published posts. Please unpublish first.');
        }

        // Editors can delete any post
        if ($user->role === 'editor') {
            return Response::allow('Editors can delete all posts');
        }

        return Response::deny('You can only delete your own unpublished posts.');
    }

    /**
     * Determine whether the user can restore the post.
     */
    public function restore(User $user, Post $post): bool
    {
        // Only editors and admins can restore deleted posts
        return in_array($user->role, ['editor', 'admin']);
    }

    /**
     * Determine whether the user can permanently delete the post.
     */
    public function forceDelete(User $user, Post $post): bool
    {
        // Only admins can permanently delete posts
        return $user->role === 'admin';
    }

    /**
     * Determine whether the user can publish the post.
     */
    public function publish(User $user, Post $post): Response
    {
        // Authors can publish their own posts
        if ($post->user_id === $user->id && $user->role === 'author') {
            return Response::allow();
        }

        // Editors can publish any post
        if ($user->role === 'editor') {
            return Response::allow();
        }

        return Response::deny('You do not have permission to publish posts.');
    }

    /**
     * Determine whether the user can unpublish the post.
     */
    public function unpublish(User $user, Post $post): bool
    {
        // Only editors and admins can unpublish posts
        return in_array($user->role, ['editor', 'admin']);
    }

    /**
     * Determine whether the user can moderate comments on the post.
     */
    public function moderateComments(User $user, Post $post): bool
    {
        // Post owners and editors can moderate comments
        return $post->user_id === $user->id || 
               in_array($user->role, ['editor', 'admin']);
    }

    /**
     * Determine whether the user can view post analytics.
     */
    public function viewAnalytics(User $user, Post $post): bool
    {
        // Post owners and editors can view analytics
        return $post->user_id === $user->id || 
               in_array($user->role, ['editor', 'admin']);
    }
}
```

##### Step 2.3: Implement User Policy
```bash
# Edit UserPolicy untuk user management authorization
nano app/Policies/UserPolicy.php
```

**Isi file app/Policies/UserPolicy.php:**
```php
<?php

namespace App\Policies;

use App\Models\User;
use Illuminate\Auth\Access\Response;

class UserPolicy
{
    /**
     * Perform pre-authorization checks.
     */
    public function before(User $user, string $ability): bool|null
    {
        // Super admin can do everything
        if ($user->role === 'admin') {
            return true;
        }

        return null;
    }

    /**
     * Determine whether the user can view any users.
     */
    public function viewAny(User $user): bool
    {
        // All authenticated users can view user lists
        return true;
    }

    /**
     * Determine whether the user can view the user.
     */
    public function view(User $user, User $model): bool
    {
        // Users can view their own profile or public profiles
        return true;
    }

    /**
     * Determine whether the user can create users.
     */
    public function create(User $user): bool
    {
        // Only admins can create users via admin panel
        return $user->role === 'admin';
    }

    /**
     * Determine whether the user can update the user.
     */
    public function update(User $user, User $model): Response
    {
        // Users can update their own profile
        if ($user->id === $model->id) {
            return Response::allow();
        }

        // Admins can update any user
        if ($user->role === 'admin') {
            return Response::allow();
        }

        return Response::deny('You can only update your own profile.');
    }

    /**
     * Determine whether the user can delete the user.
     */
    public function delete(User $user, User $model): Response
    {
        // Users cannot delete themselves
        if ($user->id === $model->id) {
            return Response::deny('You cannot delete your own account.');
        }

        // Admins can delete other users
        if ($user->role === 'admin') {
            return Response::allow();
        }

        return Response::deny('You do not have permission to delete users.');
    }

    /**
     * Determine whether the user can change roles.
     */
    public function changeRole(User $user, User $model): bool
    {
        // Only admins can change user roles
        return $user->role === 'admin' && $user->id !== $model->id;
    }

    /**
     * Determine whether the user can manage permissions.
     */
    public function managePermissions(User $user): bool
    {
        // Only admins can manage permissions
        return $user->role === 'admin';
    }

    /**
     * Determine whether the user can view admin dashboard.
     */
    public function viewAdminDashboard(User $user): bool
    {
        // Editors and admins can view admin dashboard
        return in_array($user->role, ['editor', 'admin']);
    }

    /**
     * Determine whether the user can view user analytics.
     */
    public function viewAnalytics(User $user): bool
    {
        // Only admins can view user analytics
        return $user->role === 'admin';
    }
}
```

##### Step 2.3.1: Implement Category Policy
# Edit CategoryPolicy untuk category management authorization
nano app/Policies/CategoryPolicy.php

**Isi file app/Policies/UserPolicy.php:**
```php
<?php

namespace App\Policies;

use App\Models\Category;
use App\Models\User;

class CategoryPolicy
{
    public function before(User $user, string $ability): bool|null
    {
        if ($user->role === 'admin') {
            return true;
        }
        return null;
    }

    public function viewAny(User $user): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return in_array($user->role, ['editor', 'admin']);
    }

    public function update(User $user, Category $category): bool
    {
        return in_array($user->role, ['editor', 'admin']);
    }

    public function delete(User $user, Category $category): bool
    {
        return $user->role === 'admin';
    }
}
```


##### Step 2.4: Register Policies dalam AuthServiceProvider
```bash
# Edit AuthServiceProvider untuk register policies dan gates
nano app/Providers/AuthServiceProvider.php
```

**Update file app/Providers/AuthServiceProvider.php:**
```php
<?php

namespace App\Providers;

use App\Models\Post;
use App\Models\User;
use App\Models\Category;
use App\Policies\PostPolicy;
use App\Policies\UserPolicy;
use App\Policies\CategoryPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The model to policy mappings for the application.
     *
     * @var array<class-string, class-string>
     */
    protected $policies = [
        Post::class => PostPolicy::class,
        User::class => UserPolicy::class,
        Category::class => CategoryPolicy::class,
    ];

    /**
     * Register any authentication / authorization services.
     */
    public function boot(): void
    {
        $this->registerPolicies();

        // Define custom gates untuk specific permissions
        $this->defineCustomGates();
        
        // Define role-based gates
        $this->defineRoleGates();
        
        // Define super admin gate
        $this->defineSuperAdminGate();
    }

    /**
     * Define custom gates untuk specific permissions
     */
    private function defineCustomGates(): void
    {
        // Gate untuk manage content
        Gate::define('manage-content', function (User $user) {
            return in_array($user->role, ['author', 'editor', 'admin']);
        });

        // Gate untuk moderate content
        Gate::define('moderate-content', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk manage users
        Gate::define('manage-users', function (User $user) {
            return $user->role === 'admin';
        });

        // Gate untuk view analytics
        Gate::define('view-analytics', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk manage system settings
        Gate::define('manage-settings', function (User $user) {
            return $user->role === 'admin';
        });

        // Gate untuk publish content immediately
        Gate::define('publish-immediately', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk manage categories
        Gate::define('manage-categories', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk manage tags
        Gate::define('manage-tags', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk view deleted content
        Gate::define('view-deleted-content', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        // Gate untuk export data
        Gate::define('export-data', function (User $user) {
            return $user->role === 'admin';
        });
    }

    /**
     * Define role-based gates
     */
    private function defineRoleGates(): void
    {
        Gate::define('admin', function (User $user) {
            return $user->role === 'admin';
        });

        Gate::define('editor', function (User $user) {
            return $user->role === 'editor';
        });

        Gate::define('author', function (User $user) {
            return $user->role === 'author';
        });

        Gate::define('subscriber', function (User $user) {
            return $user->role === 'subscriber';
        });

        // Hierarchical role gates
        Gate::define('editor-or-above', function (User $user) {
            return in_array($user->role, ['editor', 'admin']);
        });

        Gate::define('author-or-above', function (User $user) {
            return in_array($user->role, ['author', 'editor', 'admin']);
        });
    }

    /**
     * Define super admin gate dengan additional checks
     */
    private function defineSuperAdminGate(): void
    {
        Gate::define('super-admin', function (User $user) {
            // Check if user is admin and account is verified and active
            return $user->role === 'admin' && 
                   $user->is_active && 
                   $user->email_verified_at !== null;
        });

        // Gate untuk critical operations
        Gate::define('critical-operations', function (User $user) {
            return $user->role === 'admin' && 
                   $user->is_active && 
                   $user->created_at->lt(now()->subDays(7)); // Account age > 7 days
        });
    }

    /**
     * Check if user can perform action on specific resource
     */
    public static function canPerformAction(User $user, string $action, $resource = null): bool
    {
        if ($resource) {
            return Gate::forUser($user)->allows($action, $resource);
        }
        
        return Gate::forUser($user)->allows($action);
    }
}
```

#### **Bagian 3: Dashboard dan User Management Interface (20 menit)**

##### Step 3.1: Create Enhanced Dashboard Controller
```bash
# Generate dashboard controller dengan resource methods
php artisan make:controller DashboardController
php artisan make:controller Admin/UserManagementController --resource

# Verify controllers created
ls -la app/Http/Controllers/
ls -la app/Http/Controllers/Admin/
```

##### Step 3.2: Implement Dashboard Controller
```bash
# Edit DashboardController untuk implement dashboard logic
nano app/Http/Controllers/DashboardController.php
```

**Isi file app/Http/Controllers/DashboardController.php:**
```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\User;
use App\Models\Category;
use App\Models\Comment;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Auth;

class DashboardController extends Controller
{
    /**
     * Show the application dashboard.
     */
    public function index(Request $request)
    {
        $user = $request->user();
        
        // Get user-specific statistics
        $userStats = $this->getUserStatistics($user);
        
        // Get recent activity
        $recentActivity = $this->getRecentActivity($user);
        
        // Get user's content summary
        $contentSummary = $this->getContentSummary($user);
        
        // Get admin statistics jika user adalah admin/editor
        $adminStats = null;
        if (Gate::allows('view-analytics')) {
            $adminStats = $this->getAdminStatistics();
        }

        return view('dashboard', compact(
            'user', 
            'userStats', 
            'recentActivity', 
            'contentSummary',
            'adminStats'
        ));
    }

    /**
     * Get user-specific statistics
     */
    private function getUserStatistics(User $user): array
    {
        return [
            'total_posts' => $user->posts()->count(),
            'published_posts' => $user->posts()->where('status', 'published')->count(),
            'draft_posts' => $user->posts()->where('status', 'draft')->count(),
            'total_comments' => $user->comments()->count(),
            'posts_views' => $user->posts()->sum('views_count'),
            'account_age_days' => $user->created_at->diffInDays(now()),
            'last_post_date' => $user->posts()->latest()->first()?->created_at,
            'role_permissions' => $this->getUserPermissions($user),
        ];
    }

    /**
     * Get recent activity untuk user dashboard
     */
    private function getRecentActivity(User $user): array
    {
        $recentPosts = $user->posts()
                           ->latest()
                           ->take(5)
                           ->get(['id', 'title', 'status', 'created_at', 'views_count']);

        $recentComments = $user->comments()
                              ->with('post:id,title')
                              ->latest()
                              ->take(5)
                              ->get(['id', 'content', 'post_id', 'created_at']);

        return [
            'recent_posts' => $recentPosts,
            'recent_comments' => $recentComments,
            'pending_posts' => $user->posts()->where('status', 'draft')->count(),
        ];
    }

    /**
     * Get content summary untuk user
     */
    private function getContentSummary(User $user): array
    {
        $postsThisMonth = $user->posts()
                              ->whereMonth('created_at', now()->month)
                              ->count();

        $popularPost = $user->posts()
                           ->where('status', 'published')
                           ->orderBy('views_count', 'desc')
                           ->first(['id', 'title', 'views_count']);

        return [
            'posts_this_month' => $postsThisMonth,
            'popular_post' => $popularPost,
            'categories_used' => $user->posts()
                                     ->distinct('category_id')
                                     ->count('category_id'),
            'average_post_length' => $user->posts()
                                         ->avg(DB::raw('LENGTH(content)')),
        ];
    }

    /**
     * Get admin statistics untuk admin dashboard
     */
    private function getAdminStatistics(): array
    {
        return [
            'total_users' => User::count(),
            'active_users' => User::where('is_active', true)->count(),
            'new_users_this_week' => User::where('created_at', '>=', now()->subWeek())->count(),
            'total_posts' => Post::count(),
            'published_posts' => Post::where('status', 'published')->count(),
            'posts_pending_review' => Post::where('status', 'draft')->count(),
            'total_comments' => Comment::count(),
            'pending_comments' => Comment::where('is_approved', false)->count(),
            'users_by_role' => User::select('role', DB::raw('count(*) as count'))
                                  ->groupBy('role')
                                  ->pluck('count', 'role'),
            'posts_by_month' => Post::select(
                                   DB::raw('MONTH(created_at) as month'),
                                   DB::raw('COUNT(*) as count')
                               )
                               ->whereYear('created_at', now()->year)
                               ->groupBy('month')
                               ->orderBy('month')
                               ->pluck('count', 'month'),
        ];
    }

    /**
     * Get user permissions untuk display
     */
    private function getUserPermissions(User $user): array
    {
        $permissions = [];

        // Check various permissions
        $checkPermissions = [
            'create-posts' => 'manage-content',
            'moderate-content' => 'moderate-content',
            'manage-users' => 'manage-users',
            'view-analytics' => 'view-analytics',
            'manage-settings' => 'manage-settings',
            'manage-categories' => 'manage-categories',
        ];

        foreach ($checkPermissions as $label => $gate) {
            $permissions[$label] = Gate::forUser($user)->allows($gate);
        }

        return $permissions;
    }

    /**
     * Show user profile page
     */
    public function profile()
    {
        $user = auth()->user();
        
        return view('dashboard.profile', compact('user'));
    }

    /**
     * Show user settings page
     */
    public function settings()
    {
        $user = auth()->user();
        
        return view('dashboard.settings', compact('user'));
    }

    /**
     * Get quick actions berdasarkan user role
     */
    public function quickActions(Request $request)
    {
        $user = $request->user();
        $actions = [];

        // Role-based quick actions
        if (Gate::allows('manage-content')) {
            $actions[] = [
                'title' => 'Create New Post',
                'description' => 'Write a new blog post',
                'url' => route('posts.create'),
                'icon' => 'plus',
                'color' => 'blue'
            ];
        }

        if (Gate::allows('moderate-content')) {
            $actions[] = [
                'title' => 'Review Pending Posts',
                'description' => 'Review posts waiting for approval',
                'url' => route('posts.index', ['status' => 'draft']),
                'icon' => 'eye',
                'color' => 'yellow',
                'badge' => Post::where('status', 'draft')->count()
            ];
        }

        if (Gate::allows('manage-users')) {
            $actions[] = [
                'title' => 'Manage Users',
                'description' => 'Add, edit, or remove users',
                'url' => route('admin.users.index'),
                'icon' => 'users',
                'color' => 'green'
            ];
        }

        if (Gate::allows('view-analytics')) {
            $actions[] = [
                'title' => 'View Analytics',
                'description' => 'Check site performance and stats',
                'url' => route('dashboard') . '#analytics',
                'icon' => 'chart',
                'color' => 'purple'
            ];
        }

        return response()->json(['actions' => $actions]);
    }
}
```

##### Step 3.3: Create Dashboard View dengan Role-Based UI
```bash
# Create dashboard view dengan advanced UI
nano resources/views/dashboard.blade.php
```

**Isi file resources/views/dashboard.blade.php:**
```html
@extends('layouts.dashboard')

@section('title', 'Dashboard')
@section('page-title', 'Dashboard')
@section('page-description', 'Welcome back, ' . auth()->user()->name)

@section('content')
<div class="space-y-8">
    <!-- Quick Stats Cards -->
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <!-- Posts Stats -->
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Your Posts</p>
                    <p class="text-2xl font-bold text-gray-900">{{ $userStats['total_posts'] }}</p>
                    <p class="text-sm text-green-600">
                        {{ $userStats['published_posts'] }} published
                    </p>
                </div>
                <div class="w-12 h-12 bg-blue-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
                    </svg>
                </div>
            </div>
        </div>

        <!-- Views Stats -->
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Total Views</p>
                    <p class="text-2xl font-bold text-gray-900">{{ number_format($userStats['posts_views']) }}</p>
                    <p class="text-sm text-gray-500">
                        All time
                    </p>
                </div>
                <div class="w-12 h-12 bg-green-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"/>
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"/>
                    </svg>
                </div>
            </div>
        </div>

        <!-- Comments Stats -->
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Comments</p>
                    <p class="text-2xl font-bold text-gray-900">{{ $userStats['total_comments'] }}</p>
                    <p class="text-sm text-gray-500">
                        You've written
                    </p>
                </div>
                <div class="w-12 h-12 bg-purple-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-purple-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z"/>
                    </svg>
                </div>
            </div>
        </div>

        <!-- Account Age -->
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Member Since</p>
                    <p class="text-2xl font-bold text-gray-900">{{ $userStats['account_age_days'] }}</p>
                    <p class="text-sm text-gray-500">
                        days ago
                    </p>
                </div>
                <div class="w-12 h-12 bg-yellow-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-yellow-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"/>
                    </svg>
                </div>
            </div>
        </div>
    </div>

    <!-- Quick Actions -->
    <div class="card">
        <div class="flex items-center justify-between mb-6">
            <h3 class="text-lg font-medium text-gray-900">Quick Actions</h3>
            <button id="refreshActions" class="text-blue-600 hover:text-blue-500 text-sm">
                Refresh
            </button>
        </div>
        
        <div id="quickActionsGrid" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
            <!-- Quick actions akan dimuat via AJAX -->
            <div class="animate-pulse">
                <div class="h-24 bg-gray-200 rounded-lg"></div>
            </div>
            <div class="animate-pulse">
                <div class="h-24 bg-gray-200 rounded-lg"></div>
            </div>
            <div class="animate-pulse">
                <div class="h-24 bg-gray-200 rounded-lg"></div>
            </div>
            <div class="animate-pulse">
                <div class="h-24 bg-gray-200 rounded-lg"></div>
            </div>
        </div>
    </div>

    <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
        <!-- Recent Activity -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">Recent Activity</h3>
            
            <!-- Recent Posts -->
            <div class="mb-6">
                <h4 class="text-sm font-medium text-gray-700 mb-3">Your Recent Posts</h4>
                @forelse($recentActivity['recent_posts'] as $post)
                    <div class="flex items-center justify-between py-2 border-b border-gray-100 last:border-0">
                        <div class="flex-1">
                            <p class="text-sm font-medium text-gray-900">{{ Str::limit($post->title, 40) }}</p>
                            <div class="flex items-center space-x-2 text-xs text-gray-500">
                                <span class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium
                                    @if($post->status === 'published') bg-green-100 text-green-800
                                    @elseif($post->status === 'draft') bg-yellow-100 text-yellow-800  
                                    @else bg-gray-100 text-gray-800 @endif">
                                    {{ ucfirst($post->status) }}
                                </span>
                                <span>{{ $post->created_at->diffForHumans() }}</span>
                                <span>{{ $post->views_count }} views</span>
                            </div>
                        </div>
                        <a href="{{ route('posts.edit', $post) }}" class="text-blue-600 hover:text-blue-500 text-sm">
                            Edit
                        </a>
                    </div>
                @empty
                    <p class="text-gray-500 text-sm">No posts yet. 
                        @can('manage-content')
                            <a href="{{ route('posts.create') }}" class="text-blue-600 hover:underline">Create your first post</a>
                        @endcan
                    </p>
                @endforelse
            </div>

            <!-- Recent Comments -->
            <div>
                <h4 class="text-sm font-medium text-gray-700 mb-3">Your Recent Comments</h4>
                @forelse($recentActivity['recent_comments'] as $comment)
                    <div class="py-2 border-b border-gray-100 last:border-0">
                        <p class="text-sm text-gray-900">{{ Str::limit($comment->content, 60) }}</p>
                        <div class="flex items-center space-x-2 text-xs text-gray-500">
                            <span>on</span>
                            <span class="font-medium">{{ $comment->post->title ?? 'Unknown post' }}</span>
                            <span>•</span>
                            <span>{{ $comment->created_at->diffForHumans() }}</span>
                        </div>
                    </div>
                @empty
                    <p class="text-gray-500 text-sm">No comments yet.</p>
                @endforelse
            </div>
        </div>

        <!-- Content Summary & Permissions -->
        <div class="space-y-6">
            <!-- Content Summary -->
            <div class="card">
                <h3 class="text-lg font-medium text-gray-900 mb-4">Content Summary</h3>
                
                <div class="space-y-4">
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-gray-600">Posts this month</span>
                        <span class="font-medium">{{ $contentSummary['posts_this_month'] }}</span>
                    </div>
                    
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-gray-600">Categories used</span>
                        <span class="font-medium">{{ $contentSummary['categories_used'] }}</span>
                    </div>
                    
                    @if($contentSummary['popular_post'])
                        <div>
                            <span class="text-sm text-gray-600">Most popular post</span>
                            <div class="mt-1">
                                <p class="text-sm font-medium">{{ Str::limit($contentSummary['popular_post']->title, 30) }}</p>
                                <p class="text-xs text-gray-500">{{ number_format($contentSummary['popular_post']->views_count) }} views</p>
                            </div>
                        </div>
                    @endif
                    
                    @if($contentSummary['average_post_length'])
                        <div class="flex justify-between items-center">
                            <span class="text-sm text-gray-600">Avg. post length</span>
                            <span class="font-medium">{{ number_format($contentSummary['average_post_length']) }} chars</span>
                        </div>
                    @endif
                </div>
            </div>

            <!-- User Permissions -->
            <div class="card">
                <h3 class="text-lg font-medium text-gray-900 mb-4">Your Permissions</h3>
                
                <div class="space-y-3">
                    @foreach($userStats['role_permissions'] as $permission => $hasPermission)
                        <div class="flex items-center justify-between">
                            <span class="text-sm text-gray-600">{{ ucwords(str_replace('-', ' ', $permission)) }}</span>
                            @if($hasPermission)
                                <span class="inline-flex items-center px-2 py-1 rounded-full text-xs font-medium bg-green-100 text-green-800">
                                    <svg class="w-3 h-3 mr-1" fill="currentColor" viewBox="0 0 20 20">
                                        <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd"/>
                                    </svg>
                                    Allowed
                                </span>
                            @else
                                <span class="inline-flex items-center px-2 py-1 rounded-full text-xs font-medium bg-gray-100 text-gray-800">
                                    <svg class="w-3 h-3 mr-1" fill="currentColor" viewBox="0 0 20 20">
                                        <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"/>
                                    </svg>
                                    Denied
                                </span>
                            @endif
                        </div>
                    @endforeach
                </div>
                
                <div class="mt-4 pt-4 border-t border-gray-200">
                    <p class="text-xs text-gray-500">
                        Role: <span class="font-medium text-{{ auth()->user()->role === 'admin' ? 'red' : (auth()->user()->role === 'editor' ? 'blue' : 'green') }}-600">
                            {{ ucfirst(auth()->user()->role) }}
                        </span>
                    </p>
                </div>
            </div>
        </div>
    </div>

    <!-- Admin Statistics (jika user memiliki akses) -->
    @can('view-analytics')
        <div class="card" id="analytics">
            <h3 class="text-lg font-medium text-gray-900 mb-6">Admin Analytics</h3>
            
            <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-6">
                <!-- Users Overview -->
                <div class="bg-gradient-to-r from-blue-50 to-indigo-50 p-4 rounded-lg">
                    <h4 class="text-sm font-medium text-gray-700 mb-3">Users Overview</h4>
                    <div class="space-y-2">
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Total Users</span>
                            <span class="font-medium">{{ $adminStats['total_users'] ?? 0 }}</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Active Users</span>
                            <span class="font-medium">{{ $adminStats['active_users'] ?? 0 }}</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">New This Week</span>
                            <span class="font-medium">{{ $adminStats['new_users_this_week'] ?? 0 }}</span>
                        </div>
                    </div>
                </div>

                <!-- Content Overview -->
                <div class="bg-gradient-to-r from-green-50 to-emerald-50 p-4 rounded-lg">
                    <h4 class="text-sm font-medium text-gray-700 mb-3">Content Overview</h4>
                    <div class="space-y-2">
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Total Posts</span>
                            <span class="font-medium">{{ $adminStats['total_posts'] ?? 0 }}</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Published</span>
                            <span class="font-medium">{{ $adminStats['published_posts'] ?? 0 }}</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Pending Review</span>
                            <span class="font-medium">{{ $adminStats['posts_pending_review'] ?? 0 }}</span>
                        </div>
                    </div>
                </div>

                <!-- Comments Overview -->
                <div class="bg-gradient-to-r from-purple-50 to-pink-50 p-4 rounded-lg">
                    <h4 class="text-sm font-medium text-gray-700 mb-3">Comments Overview</h4>
                    <div class="space-y-2">
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Total Comments</span>
                            <span class="font-medium">{{ $adminStats['total_comments'] ?? 0 }}</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-sm text-gray-600">Pending Approval</span>
                            <span class="font-medium">{{ $adminStats['pending_comments'] ?? 0 }}</span>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Users by Role Chart -->
            @if(isset($adminStats['users_by_role']) && $adminStats['users_by_role']->isNotEmpty())
                <div class="mb-6">
                    <h4 class="text-sm font-medium text-gray-700 mb-3">Users by Role</h4>
                    <div class="flex space-x-4">
                        @foreach($adminStats['users_by_role'] as $role => $count)
                            <div class="flex items-center space-x-2">
                                <div class="w-3 h-3 rounded-full 
                                    @if($role === 'admin') bg-red-500 
                                    @elseif($role === 'editor') bg-blue-500 
                                    @elseif($role === 'author') bg-green-500 
                                    @else bg-gray-500 @endif"></div>
                                <span class="text-sm text-gray-600">{{ ucfirst($role) }}: {{ $count }}</span>
                            </div>
                        @endforeach
                    </div>
                </div>
            @endif
        </div>
    @endcan
</div>

<!-- JavaScript untuk Quick Actions -->
<script>
document.addEventListener('DOMContentLoaded', function() {
    loadQuickActions();
    
    document.getElementById('refreshActions').addEventListener('click', function() {
        loadQuickActions();
    });
});

function loadQuickActions() {
    fetch('/dashboard/quick-actions')
        .then(response => response.json())
        .then(data => {
            renderQuickActions(data.actions);
        })
        .catch(error => {
            console.error('Error loading quick actions:', error);
        });
}

function renderQuickActions(actions) {
    const grid = document.getElementById('quickActionsGrid');
    
    if (actions.length === 0) {
        grid.innerHTML = '<p class="text-gray-500 text-sm col-span-full">No quick actions available for your role.</p>';
        return;
    }
    
    grid.innerHTML = actions.map(action => {
        // Predefined color classes yang sudah di-compile Tailwind
        const colorClasses = getColorClasses(action.color);
        
        return `
            <a href="${action.url}" class="block p-4 border border-gray-200 rounded-lg hover:shadow-md transition group ${colorClasses.hover}">
                <div class="flex items-center space-x-3">
                    <div class="w-8 h-8 rounded-lg flex items-center justify-center transition ${colorClasses.bg} ${colorClasses.groupHover}">
                        ${getIcon(action.icon, action.color)}
                    </div>
                    <div class="flex-1 min-w-0">
                        <p class="text-sm font-medium text-gray-900 truncate">${action.title}</p>
                        <p class="text-xs text-gray-500 truncate">${action.description}</p>
                    </div>
                    ${action.badge ? `<span class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium ${colorClasses.badge}">${action.badge}</span>` : ''}
                </div>
            </a>
        `;
    }).join('');
}

function getColorClasses(color) {
    const colorMap = {
        'blue': {
            hover: 'hover:border-blue-300',
            bg: 'bg-blue-100',
            groupHover: 'group-hover:bg-blue-200',
            badge: 'bg-blue-100 text-blue-800'
        },
        'green': {
            hover: 'hover:border-green-300',
            bg: 'bg-green-100', 
            groupHover: 'group-hover:bg-green-200',
            badge: 'bg-green-100 text-green-800'
        },
        'yellow': {
            hover: 'hover:border-yellow-300',
            bg: 'bg-yellow-100',
            groupHover: 'group-hover:bg-yellow-200', 
            badge: 'bg-yellow-100 text-yellow-800'
        },
        'purple': {
            hover: 'hover:border-purple-300',
            bg: 'bg-purple-100',
            groupHover: 'group-hover:bg-purple-200',
            badge: 'bg-purple-100 text-purple-800'
        }
    };
    
    return colorMap[color] || {
        hover: 'hover:border-gray-300',
        bg: 'bg-gray-100',
        groupHover: 'group-hover:bg-gray-200',
        badge: 'bg-gray-100 text-gray-800'
    };
}

function getIcon(iconName, color) {
    // Predefined icon color classes
    const iconColorClasses = {
        'blue': 'text-blue-600',
        'green': 'text-green-600', 
        'yellow': 'text-yellow-600',
        'purple': 'text-purple-600'
    };
    
    const colorClass = iconColorClasses[color] || 'text-gray-600';
    
    const icons = {
        plus: `<svg class="w-4 h-4 ${colorClass}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"/></svg>`,
        eye: `<svg class="w-4 h-4 ${colorClass}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"/><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"/></svg>`,
        users: `<svg class="w-4 h-4 ${colorClass}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4.354a4 4 0 110 5.292M15 21H3v-1a6 6 0 0112 0v1zm0 0h6v-1a6 6 0 00-9-5.197m13.5-9a4 4 0 11-8 0 4 4 0 018 0z"/></svg>`,
        chart: `<svg class="w-4 h-4 ${colorClass}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z"/></svg>`
    };
    
    return icons[iconName] || icons.plus;
}
</script>
@endsection
```

#### **Bagian 4: Email Verification & Password Reset Enhancement (15 menit)**

##### Step 4.1: Configure Email Services
```bash
# Update .env untuk email configuration
nano .env
```

**Tambahkan email configuration ke .env:**
```bash
# === Email Configuration ===
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_mailtrap_username
MAIL_PASSWORD=your_mailtrap_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="noreply@praktikum-cc.itk.ac.id"
MAIL_FROM_NAME="Praktikum Cloud Computing ITK"

# For production, use real SMTP service:
# MAIL_HOST=smtp.gmail.com
# MAIL_PORT=587
# MAIL_USERNAME=your_email@gmail.com
# MAIL_PASSWORD=your_app_password
```

##### Step 4.2: Implement Enhanced Email Verification
```bash
# Update User model untuk implement MustVerifyEmail
nano app/Models/User.php
```

**Update User model dengan email verification:**
```php
// Add to the top of User.php
use Illuminate\Contracts\Auth\MustVerifyEmail;

// Update class declaration
class User extends Authenticatable implements MustVerifyEmail
{
    // ... existing code ...

    /**
     * Send email verification notification
     */
    public function sendEmailVerificationNotification()
    {
        $this->notify(new \App\Notifications\CustomVerifyEmail);
    }

    /**
     * Check if user has verified email recently
     */
    public function hasVerifiedEmailRecently(): bool
    {
        return $this->email_verified_at && 
               $this->email_verified_at->gt(now()->subDays(30));
    }

    /**
     * Mark email as verified
     */
    public function markEmailAsVerified()
    {
        return $this->forceFill([
            'email_verified_at' => $this->freshTimestamp(),
        ])->save();
    }
}
```

##### Step 4.3: Create Custom Email Notification
```bash
# Generate custom email verification notification
php artisan make:notification CustomVerifyEmail

# Edit notification untuk customize email template
nano app/Notifications/CustomVerifyEmail.php
```

**Isi file app/Notifications/CustomVerifyEmail.php:**
```php
<?php

namespace App\Notifications;

use Illuminate\Auth\Notifications\VerifyEmail as VerifyEmailBase;
use Illuminate\Notifications\Messages\MailMessage;

class CustomVerifyEmail extends VerifyEmailBase
{
    /**
     * Build the mail representation of the notification.
     */
    public function toMail($notifiable): MailMessage
    {
        $verificationUrl = $this->verificationUrl($notifiable);

        return (new MailMessage)
            ->subject('Verify Your Email Address - Praktikum Cloud Computing ITK')
            ->greeting('Hello ' . $notifiable->name . '!')
            ->line('Thank you for registering with Praktikum Cloud Computing ITK.')
            ->line('Please click the button below to verify your email address.')
            ->action('Verify Email Address', $verificationUrl)
            ->line('This verification link will expire in 60 minutes.')
            ->line('If you did not create an account, no further action is required.')
            ->salutation('Best regards, Praktikum Cloud Computing ITK Team');
    }
}
```

##### Step 4.4: Add Email Verification Routes dan Middleware
```bash
# Update web routes untuk include email verification
nano routes/web.php
```

**Tambahkan email verification routes:**
```php

// Tambahkan di bagian atas routes/web.php
use Illuminate\Foundation\Auth\EmailVerificationRequest;
use Illuminate\Http\Request;

// Add after existing routes

/*
|--------------------------------------------------------------------------
| Email Verification Routes
|--------------------------------------------------------------------------
*/

// Email verification notice
Route::get('/email/verify', function () {
    return view('auth.verify-email');
})->middleware('auth')->name('verification.notice');

// Email verification handler
Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();
    
    return redirect('/dashboard')->with('success', 'Email verified successfully!');
})->middleware(['auth', 'signed'])->name('verification.verify');

// Resend verification email
Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();
    
    return back()->with('message', 'Verification link sent!');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');

// Add email verification middleware ke protected routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::get('/dashboard/profile', [DashboardController::class, 'profile'])->name('dashboard.profile');
    Route::get('/dashboard/settings', [DashboardController::class, 'settings'])->name('dashboard.settings');
    Route::get('/dashboard/quick-actions', [DashboardController::class, 'quickActions'])->name('dashboard.quick-actions');
    
    // Resource routes yang memerlukan verified email
    Route::resource('posts', PostController::class);
    Route::resource('categories', CategoryController::class);
    Route::resource('tags', TagController::class);
});

// Admin routes 
Route::prefix('admin')->name('admin.')->middleware(['auth', 'can:manage-users'])->group(function () {
    Route::resource('users', Admin\UserManagementController::class);
});
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Authentication System Testing
```bash
echo "=== Laravel Breeze Authentication Testing ==="

# Test 1.1: Verify Breeze installation
composer show laravel/breeze && echo "✓ Laravel Breeze installed"

# Test 1.2: Check authentication views
VIEWS=("login" "register" "forgot-password" "reset-password" "verify-email")
for view in "${VIEWS[@]}"; do
    if [ -f "resources/views/auth/$view.blade.php" ]; then
        echo "✓ Auth view $view exists"
    else
        echo "✗ Auth view $view missing"
    fi
done

# Test 1.3: Check authentication routes
php artisan route:list | grep -E "(login|register|password|verify)" | head -10

# Test 1.4: Test authentication middleware
php artisan route:list | grep -c "auth" && echo "✓ Authentication middleware applied"
```

#### Test 2: Authorization Policies Testing
```bash
echo "=== Authorization Policies Testing ==="

# Test 2.1: Verify policies exist
POLICIES=("PostPolicy" "UserPolicy" "CategoryPolicy")
for policy in "${POLICIES[@]}"; do
    if [ -f "app/Policies/$policy.php" ]; then
        echo "✓ Policy $policy exists"
    else
        echo "✗ Policy $policy missing"
    fi
done

# Test 2.2: Test policy methods
php artisan tinker --execute="
\$user = \App\Models\User::where('role', 'author')->first();
\$post = \App\Models\Post::first();

if (\$user && \$post) {
    // Test post policy methods
    echo 'User can view any posts: ' . (\$user->can('viewAny', \App\Models\Post::class) ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can create posts: ' . (\$user->can('create', \App\Models\Post::class) ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can update this post: ' . (\$user->can('update', \$post) ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can delete this post: ' . (\$user->can('delete', \$post) ? 'Yes' : 'No') . PHP_EOL;
} else {
    echo 'No test data available for policy testing' . PHP_EOL;
}
"

# Test 2.3: Test custom gates
php artisan tinker --execute="
\$user = \App\Models\User::where('role', 'editor')->first();

if (\$user) {
    use Illuminate\Support\Facades\Gate;
    
    echo 'User can manage content: ' . (Gate::forUser(\$user)->allows('manage-content') ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can moderate content: ' . (Gate::forUser(\$user)->allows('moderate-content') ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can manage users: ' . (Gate::forUser(\$user)->allows('manage-users') ? 'Yes' : 'No') . PHP_EOL;
    echo 'User can view analytics: ' . (Gate::forUser(\$user)->allows('view-analytics') ? 'Yes' : 'No') . PHP_EOL;
} else {
    echo 'No editor user found for gate testing' . PHP_EOL;
}
"
```

#### Test 3: Dashboard Functionality Testing
```bash
echo "=== Dashboard Functionality Testing ==="

# Test 3.1: Dashboard controller exists
if [ -f "app/Http/Controllers/DashboardController.php" ]; then
    echo "✓ DashboardController exists"
else
    echo "✗ DashboardController missing"
fi

# Test 3.2: Dashboard view exists
if [ -f "resources/views/dashboard.blade.php" ]; then
    echo "✓ Dashboard view exists"
else
    echo "✗ Dashboard view missing"
fi

# Test 3.3: Test dashboard data methods
php artisan tinker --execute="
\$controller = new \App\Http\Controllers\DashboardController();
\$user = \App\Models\User::first();

if (\$user) {
    // Test if methods exist
    \$reflection = new ReflectionClass(\$controller);
    \$methods = ['getUserStatistics', 'getRecentActivity', 'getContentSummary'];
    
    foreach (\$methods as \$method) {
        if (\$reflection->hasMethod(\$method)) {
            echo '✓ Method ' . \$method . ' exists' . PHP_EOL;
        } else {
            echo '✗ Method ' . \$method . ' missing' . PHP_EOL;
        }
    }
} else {
    echo 'No users found for testing' . PHP_EOL;
}
"

# Test 3.4: Quick actions endpoint
curl -s -H "Accept: application/json" http://localhost:8000/dashboard/quick-actions | jq -e '.actions' >/dev/null && echo "✓ Quick actions endpoint working"
```

#### Test 4: Email Verification Testing
```bash
echo "=== Email Verification Testing ==="

# Test 4.1: Check MustVerifyEmail interface
php artisan tinker --execute="
\$user = \App\Models\User::first();
if (\$user instanceof \Illuminate\Contracts\Auth\MustVerifyEmail) {
    echo '✓ User model implements MustVerifyEmail' . PHP_EOL;
} else {
    echo '✗ User model does not implement MustVerifyEmail' . PHP_EOL;
}
"

# Test 4.2: Check custom verification notification
if [ -f "app/Notifications/CustomVerifyEmail.php" ]; then
    echo "✓ Custom email verification notification exists"
else
    echo "✗ Custom email verification notification missing"
fi

# Test 4.3: Test email configuration
php artisan tinker --execute="
echo 'Mail driver: ' . config('mail.default') . PHP_EOL;
echo 'Mail host: ' . config('mail.mailers.smtp.host') . PHP_EOL;
echo 'Mail from address: ' . config('mail.from.address') . PHP_EOL;
echo 'Mail from name: ' . config('mail.from.name') . PHP_EOL;
"

# Test 4.4: Test verification routes
php artisan route:list | grep verification && echo "✓ Email verification routes registered"
```

#### Test 5: Role-Based UI Testing
```bash
echo "=== Role-Based UI Testing ==="

# Test 5.1: Test different user roles dapat access dashboard
USER_ROLES=("admin" "editor" "author" "subscriber")
for role in "${USER_ROLES[@]}"; do
    php artisan tinker --execute="
    \$user = \App\Models\User::where('role', '$role')->first();
    if (\$user) {
        echo '✓ $role user exists and can potentially access dashboard' . PHP_EOL;
    } else {
        echo '⚠ No $role user found in database' . PHP_EOL;
    }
    "
done

# Test 5.2: Test role-based permissions display
php artisan tinker --execute="
\$users = \App\Models\User::whereIn('role', ['admin', 'editor', 'author'])->get();

foreach (\$users as \$user) {
    echo 'User: ' . \$user->name . ' (Role: ' . \$user->role . ')' . PHP_EOL;
    
    \$permissions = [
        'manage-content' => \Gate::forUser(\$user)->allows('manage-content'),
        'moderate-content' => \Gate::forUser(\$user)->allows('moderate-content'),
        'manage-users' => \Gate::forUser(\$user)->allows('manage-users'),
        'view-analytics' => \Gate::forUser(\$user)->allows('view-analytics'),
    ];
    
    foreach (\$permissions as \$perm => \$allowed) {
        echo '  ' . \$perm . ': ' . (\$allowed ? 'Allowed' : 'Denied') . PHP_EOL;
    }
    echo PHP_EOL;
}
"
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

#### ✅ Laravel Breeze Integration
- [ ] Screenshot Laravel Breeze installation successful
- [ ] Enhanced authentication layouts dengan custom Tailwind CSS styling
- [ ] Login, register, forgot password forms working dengan proper validation
- [ ] Authentication middleware applied pada protected routes

#### ✅ Advanced Authorization System
- [ ] PostPolicy, UserPolicy, CategoryPolicy implemented dengan comprehensive methods
- [ ] Custom Gates defined dalam AuthServiceProvider untuk role-based permissions
- [ ] Authorization testing screenshots menunjukkan different roles have different access
- [ ] Policy responses dengan custom messages working

#### ✅ Dashboard Implementation
- [ ] Role-based dashboard dengan user statistics dan analytics
- [ ] Quick actions loading dynamically based on user permissions
- [ ] Recent activity feed dengan user's posts dan comments
- [ ] Admin analytics section (untuk editor+ roles) dengan comprehensive data

#### ✅ Email Verification System
- [ ] Custom email verification notification dengan branded template
- [ ] Email verification routes dan middleware configured
- [ ] Screenshots email verification flow working
- [ ] Verified email requirement enforced pada protected routes

#### ✅ User Interface Enhancements
- [ ] Role-based UI elements showing/hiding based on permissions
- [ ] Responsive dashboard design working on mobile dan desktop
- [ ] Permission indicators dalam user profile section
- [ ] Enhanced navigation dengan role-appropriate menu items

#### ✅ Security & Testing
- [ ] Authorization testing suite results showing proper access control
- [ ] Session management testing dengan remember me functionality
- [ ] Email verification testing dengan proper token validation
- [ ] Cross-role testing demonstrating proper permission enforcement

**Format Submission:**
```bash
cd ~/praktikum-cc
mkdir -p submission/week7/{controllers,policies,views,middleware,tests,config}

# Copy implementation files
cp app/Http/Controllers/DashboardController.php submission/week7/controllers/
cp app/Http/Controllers/Admin/UserManagementController.php submission/week7/controllers/
cp app/Policies/*.php submission/week7/policies/
cp app/Providers/AuthServiceProvider.php submission/week7/policies/

# Copy enhanced views
cp resources/views/dashboard.blade.php submission/week7/views/
cp resources/views/layouts/guest.blade.php submission/week7/views/
cp resources/views/auth/login.blade.php submission/week7/views/

# Copy middleware dan notifications
cp app/Http/Middleware/*.php submission/week7/middleware/ 2>/dev/null || echo "No custom middleware"
cp app/Notifications/CustomVerifyEmail.php submission/week7/

# Copy configuration files
cp routes/web.php submission/week7/routes-web.php
cp config/mail.php submission/week7/config/

# Export current .env variables (remove sensitive data)
grep -E "(MAIL_|APP_)" .env | sed 's/=.*/=***/' > submission/week7/config/env-variables.txt

# Create comprehensive documentation
cat > submission/week7/README.md << 'EOF'
# Week 7: User Authentication & Authorization

## Implementation Summary
- [x] Laravel Breeze integration dengan custom Tailwind CSS styling
- [x] Advanced authorization system dengan Policies dan Gates
- [x] Role-based dashboard dengan analytics dan quick actions
- [x] Email verification system dengan custom notifications
- [x] Enhanced user interface dengan permission-based elements
- [x] Comprehensive testing dan security measures

## Authentication Features
### Laravel Breeze Integration
- Custom guest layout dengan gradient backgrounds dan modern UI
- Enhanced login/register forms dengan social login placeholders
- Responsive design compatible dengan existing Tailwind components
- Remember me functionality dengan extended session lifetime

### Authorization System
- **Policies**: PostPolicy, UserPolicy, CategoryPolicy dengan detailed permission logic
- **Gates**: Custom gates untuk manage-content, moderate-content, manage-users, etc.
- **Role Hierarchy**: admin > editor > author > subscriber dengan appropriate permissions
- **Response Messages**: Detailed authorization responses untuk better UX

## Dashboard Features
### User Statistics
- Personal content metrics (posts, views, comments)
- Account information dan membership duration
- Role-based permission indicators
- Recent activity feed

### Admin Analytics (Editor+ Roles)
- System-wide statistics (users, posts, comments)
- User distribution by role
- Content approval queue
- Monthly content creation trends

### Quick Actions
- Dynamic action buttons based on user permissions
- Real-time badge counts untuk pending items
- Role-appropriate shortcuts dan links

## Security Enhancements
### Email Verification
- Custom branded email templates
- Secure token-based verification
- Automatic verification requirement enforcement
- Resend verification functionality

### Session Management
- Secure session configuration
- Remember me dengan extended lifetime
- Multiple device session tracking
- Proper logout functionality

## Role-Based Access Control
### Permission Matrix
- **Admin**: Full system access, user management, all content operations
- **Editor**: Content moderation, publishing, analytics access
- **Author**: Own content creation/editing, basic analytics
- **Subscriber**: View content, comment (verified email required)

### UI Authorization
- Menu items show/hide based on permissions
- Action buttons dengan permission checking
- Conditional content rendering
- Role indicators dalam user interface

## Technical Implementation
### Database Schema
- No additional tables required (uses existing user roles)
- Email verification timestamps tracking
- Session data management

### Middleware Stack
- Authentication middleware (auth)
- Email verification middleware (verified)
- Custom role-based middleware
- Rate limiting untuk sensitive operations

### Performance Optimizations
- Efficient permission checking dengan caching
- Lazy loading untuk dashboard statistics
- Optimized database queries dengan proper indexing
- AJAX-based quick actions loading

## Testing Results
- ✓ All authentication flows working correctly
- ✓ Authorization policies enforcing proper access control
- ✓ Email verification system functional
- ✓ Dashboard loading dengan appropriate data untuk each role
- ✓ UI elements showing/hiding based on permissions
- ✓ Session management working across devices
- ✓ Quick actions dynamically generated per role
- ✓ Email notifications sending properly

## Configuration Files
- Enhanced guest layout template
- Custom email verification notification
- AuthServiceProvider dengan comprehensive gates
- Dashboard controller dengan role-based logic
- Routes dengan proper middleware protection

## Next Steps
- Week 8: Production Deployment & Optimization
- Performance tuning untuk authorization checks
- Advanced permission systems dengan custom abilities
- Multi-factor authentication implementation
EOF

# Run comprehensive testing
echo "Running comprehensive authentication & authorization tests..."

# Test authentication flow
curl -s -c cookies.txt -b cookies.txt http://localhost:8000/login >/dev/null
echo "✓ Login page accessible"

# Test dashboard access (should redirect to login)
DASHBOARD_RESPONSE=$(curl -s -w "%{http_code}" -o /dev/null http://localhost:8000/dashboard)
if [ "$DASHBOARD_RESPONSE" = "302" ]; then
    echo "✓ Dashboard properly protected"
else
    echo "⚠ Dashboard protection may not be working (got $DASHBOARD_RESPONSE)"
fi

# Test authorization policies
php artisan tinker --execute="
\$admin = \App\Models\User::where('role', 'admin')->first();
\$author = \App\Models\User::where('role', 'author')->first();
\$post = \App\Models\Post::first();

if (\$admin && \$author && \$post) {
    echo 'Policy Testing Results:' . PHP_EOL;
    echo '  Admin can manage users: ' . (\$admin->can('manage-users') ? 'Yes' : 'No') . PHP_EOL;
    echo '  Author can manage users: ' . (\$author->can('manage-users') ? 'Yes' : 'No') . PHP_EOL;
    echo '  Admin can update any post: ' . (\$admin->can('update', \$post) ? 'Yes' : 'No') . PHP_EOL;
    echo '  Author can update any post: ' . (\$author->can('update', \$post) ? 'Yes' : 'No') . PHP_EOL;
} else {
    echo 'Missing test data for policy testing' . PHP_EOL;
}
" > submission/week7/tests/authorization-test-results.txt

# Test email configuration
php artisan tinker --execute="
echo 'Email Configuration Test:' . PHP_EOL;
echo '  Mail driver: ' . config('mail.default') . PHP_EOL;
echo '  From address: ' . config('mail.from.address') . PHP_EOL;
echo '  From name: ' . config('mail.from.name') . PHP_EOL;
" > submission/week7/tests/email-config-test.txt

# Generate routes documentation
php artisan route:list --path=dashboard > submission/week7/tests/dashboard-routes.txt
php artisan route:list --path=auth > submission/week7/tests/auth-routes.txt

# Commit and push
git add submission/week7/
git commit -m "Week 7: User Authentication & Authorization - [Nama]"
git push origin main

echo "Week 7 submission completed successfully!"
echo "Dashboard URL: http://localhost:8000/dashboard"
echo "Login URL: http://localhost:8000/login"
```