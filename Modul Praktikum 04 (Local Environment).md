# WEEK 4: CRUD Operations - Create & Read
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 4
- **Durasi**: 100 menit  
- **Topik**: Implementasi CRUD Operations (Create & Read) dengan Laravel
- **Target**: Mahasiswa Semester 6
- **Platform**: Laptop/PC (Windows, macOS, atau Linux)

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Mengimplementasikan Laravel Controllers dengan resource methods yang optimal
2. Membuat form handling dengan validation yang comprehensive
3. Melakukan operasi database Create dan Read menggunakan Eloquent ORM
4. Mengintegrasikan Tailwind CSS untuk UI forms dan data display
5. Membangun API endpoints untuk frontend communication
6. Implementasi error handling dan user feedback systems
7. Menerapkan best practices untuk CRUD operations security

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1, 2, dan 3 telah completed successfully
- Database schema sudah implemented dengan proper relationships
- Laravel models sudah configured dengan fillable attributes
- Tailwind CSS sudah functional dengan custom components

**Environment Verification:**
```bash
# Pastikan berada di direktori project Laravel
cd laravel-app

# Masuk ke tinker untuk verifikasi database connection dan models
php artisan tinker

# Isi manual di tinker
DB::connection()->getPdo();
App\Models\User::count();
App\Models\Category::count();

# Verifikasi Tailwind CSS build
ls public/build/assets | findstr ".css"
ls public/build/assets | findstr ".js"
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Setup Controllers dan Routes (20 menit)**

##### Step 1.1: Generate Resource Controllers
```bash
# Generate resource controllers untuk semua models utama
php artisan make:controller PostController --resource --model=Post
php artisan make:controller CategoryController --resource --model=Category
php artisan make:controller TagController --resource --model=Tag
php artisan make:controller CommentController --resource --model=Comment

# Generate API controllers untuk AJAX operations
php artisan make:controller Api/PostApiController --api --model=Post
php artisan make:controller Api/CategoryApiController --api --model=Category

# Verifikasi controllers telah dibuat
dir app/Http/Controllers/
dir app/Http/Controllers/Api/
```

##### Step 1.2: Konfigurasi Web Routes

**Isi file routes/web.php:**
```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\TagController;
use App\Http\Controllers\CommentController;

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

// Dashboard route untuk admin interface
Route::get('/dashboard', function () {
    return view('dashboard');
})->name('dashboard');

/*
|--------------------------------------------------------------------------
| Resource Routes untuk CRUD Operations
|--------------------------------------------------------------------------
*/

// Posts management routes
Route::resource('posts', PostController::class)->names([
    'index' => 'posts.index',       // GET /posts - List all posts
    'create' => 'posts.create',     // GET /posts/create - Show create form
    'store' => 'posts.store',       // POST /posts - Store new post
    'show' => 'posts.show',         // GET /posts/{post} - Show single post
    'edit' => 'posts.edit',         // GET /posts/{post}/edit - Show edit form
    'update' => 'posts.update',     // PUT/PATCH /posts/{post} - Update post
    'destroy' => 'posts.destroy',   // DELETE /posts/{post} - Delete post
]);

// Categories management routes
Route::resource('categories', CategoryController::class)->except([
    'show' // Categories tidak memerlukan show page individual
]);

// Tags management routes  
Route::resource('tags', TagController::class)->except([
    'show' // Tags tidak memerlukan show page individual
]);

// Comments routes (nested dalam posts)
Route::resource('posts.comments', CommentController::class)->except([
    'index', 'show' // Comments di-handle dalam post show page
])->shallow(); // Shallow nesting untuk edit/update/delete

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

##### Step 1.3: Konfigurasi API Routes

**Isi file routes/api.php:**
```php

<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\PostApiController;
use App\Http\Controllers\Api\CategoryApiController;

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

```
##### Step 1.3: Konfigurasi Bootstrap/app.php

**Isi file bootstrap/app.php:**
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
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })
    ->create();
```


#### **Bagian 2: Implementasi Controller (25 menit)**


**Isi file app/Http/Controllers/PostController.php:**
```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\Category;
use App\Models\Tag;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Illuminate\Validation\Rule;

class PostController extends Controller
{
    /**
     * Display listing of posts dengan pagination dan filtering
     * Method ini untuk admin interface - menampilkan semua posts
     */
    public function index(Request $request)
    {
        // Query builder untuk posts dengan eager loading relationships
        $query = Post::with(['user', 'category', 'tags'])
                    ->withCount('comments'); // Include comment count

        // Filtering berdasarkan status
        if ($request->filled('status')) {
            $query->where('status', $request->status);
        }

        // Filtering berdasarkan category
        if ($request->filled('category')) {
            $query->where('category_id', $request->category);
        }

        // Filtering berdasarkan author
        if ($request->filled('author')) {
            $query->where('user_id', $request->author);
        }

        // Search functionality
        if ($request->filled('search')) {
            $searchTerm = $request->search;
            $query->where(function ($q) use ($searchTerm) {
                $q->where('title', 'LIKE', "%{$searchTerm}%")
                  ->orWhere('content', 'LIKE', "%{$searchTerm}%")
                  ->orWhere('excerpt', 'LIKE', "%{$searchTerm}%");
            });
        }

        // Sorting - default by latest created
        $sortBy = $request->get('sort', 'created_at');
        $sortOrder = $request->get('order', 'desc');
        $query->orderBy($sortBy, $sortOrder);

        // Pagination dengan 10 posts per page
        $posts = $query->paginate(10)->withQueryString();

        // Data untuk filter dropdowns
        $categories = Category::active()->orderBy('name')->get();
        $authors = User::active()->orderBy('name')->get();

        // Statistics untuk dashboard
        $stats = [
            'total' => Post::count(),
            'published' => Post::published()->count(),
            'draft' => Post::where('status', 'draft')->count(),
            'archived' => Post::where('status', 'archived')->count(),
        ];

        return view('posts.index', compact('posts', 'categories', 'authors', 'stats'));
    }

    /**
     * Show form untuk create new post
     */
    public function create()
    {
        // Data untuk form dropdowns
        $categories = Category::active()->orderBy('name')->get();
        $tags = Tag::orderBy('name')->get();
        $users = User::active()->role(['admin', 'editor', 'author'])->orderBy('name')->get();

        return view('posts.create', compact('categories', 'tags', 'users'));
    }

    /**
     * Store new post ke database
     */
    public function store(Request $request)
    {
        // Validation rules untuk create post
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'excerpt' => 'nullable|string|max:500',
            'featured_image' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
            'status' => 'required|in:draft,published,archived',
            'category_id' => 'required|exists:categories,id',
            'user_id' => 'required|exists:users,id',
            'tags' => 'nullable|array',
            'tags.*' => 'exists:tags,id',
            'meta_title' => 'nullable|string|max:255',
            'meta_description' => 'nullable|string|max:500',
            'meta_keywords' => 'nullable|string|max:255',
            'is_featured' => 'boolean',
            'allow_comments' => 'boolean',
        ]);

        // Generate unique slug dari title
        $validated['slug'] = $this->generateUniqueSlug($validated['title']);

        // Handle featured image upload
        if ($request->hasFile('featured_image')) {
            $validated['featured_image'] = $request->file('featured_image')
                                                 ->store('posts/featured-images', 'public');
        }

        // Set published_at jika status published
        if ($validated['status'] === 'published') {
            $validated['published_at'] = now();
        }

        // Create post record
        $post = Post::create($validated);

        // Attach tags jika ada
        if (!empty($validated['tags'])) {
            $post->tags()->attach($validated['tags']);
        }

        // Flash message success
        return redirect()->route('posts.index')
                        ->with('success', "Post '{$post->title}' berhasil dibuat!");
    }

    /**
     * Display single post untuk admin view
     */
    public function show(Post $post)
    {
        // Load relationships untuk display
        $post->load(['user', 'category', 'tags', 'comments.user']);

        // Increment views count (optional untuk admin view)
        $post->incrementViews();

        // Related posts berdasarkan category dan tags
        $relatedPosts = Post::published()
                           ->where('id', '!=', $post->id)
                           ->where(function ($query) use ($post) {
                               $query->where('category_id', $post->category_id)
                                     ->orWhereHas('tags', function ($q) use ($post) {
                                         $q->whereIn('tags.id', $post->tags->pluck('id'));
                                     });
                           })
                           ->withCount('comments')
                           ->take(3)
                           ->get();

        return view('posts.show', compact('post', 'relatedPosts'));
    }

    /**
     * Show edit form untuk existing post
     */
    public function edit(Post $post)
    {
        // Data untuk form dropdowns
        $categories = Category::active()->orderBy('name')->get();
        $tags = Tag::orderBy('name')->get();
        $users = User::active()->role(['admin', 'editor', 'author'])->orderBy('name')->get();

        // Selected tags untuk form
        $selectedTags = $post->tags->pluck('id')->toArray();

        return view('posts.edit', compact('post', 'categories', 'tags', 'users', 'selectedTags'));
    }

    /**
     * Update existing post
     */
    public function update(Request $request, Post $post)
    {
        // Validation rules untuk update post
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'excerpt' => 'nullable|string|max:500',
            'featured_image' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
            'status' => 'required|in:draft,published,archived',
            'category_id' => 'required|exists:categories,id',
            'user_id' => 'required|exists:users,id',
            'tags' => 'nullable|array',
            'tags.*' => 'exists:tags,id',
            'meta_title' => 'nullable|string|max:255',
            'meta_description' => 'nullable|string|max:500',
            'meta_keywords' => 'nullable|string|max:255',
            'is_featured' => 'boolean',
            'allow_comments' => 'boolean',
        ]);

        // Update slug jika title berubah
        if ($post->title !== $validated['title']) {
            $validated['slug'] = $this->generateUniqueSlug($validated['title'], $post->id);
        }

        // Handle featured image upload
        if ($request->hasFile('featured_image')) {
            // Delete old image jika ada
            if ($post->featured_image) {
                \Storage::disk('public')->delete($post->featured_image);
            }
            
            $validated['featured_image'] = $request->file('featured_image')
                                                 ->store('posts/featured-images', 'public');
        }

        // Set published_at jika status berubah ke published
        if ($validated['status'] === 'published' && $post->status !== 'published') {
            $validated['published_at'] = now();
        }

        // Update post record
        $post->update($validated);

        // Sync tags
        if (isset($validated['tags'])) {
            $post->tags()->sync($validated['tags']);
        } else {
            $post->tags()->detach();
        }

        // Flash message success
        return redirect()->route('posts.index')
                        ->with('success', "Post '{$post->title}' berhasil diupdate!");
    }

    /**
     * Delete post dari database
     */
    public function destroy(Post $post)
    {
        $title = $post->title;

        // Delete featured image jika ada
        if ($post->featured_image) {
            \Storage::disk('public')->delete($post->featured_image);
        }

        // Soft delete post (karena menggunakan SoftDeletes trait)
        $post->delete();

        // Flash message success
        return redirect()->route('posts.index')
                        ->with('success', "Post '{$title}' berhasil dihapus!");
    }

    /*
    |--------------------------------------------------------------------------
    | Public Frontend Methods
    |--------------------------------------------------------------------------
    */

    /**
     * Display published posts untuk public blog
     */
    public function publicIndex(Request $request)
    {
        $query = Post::published()
                    ->with(['user', 'category', 'tags'])
                    ->withCount('comments');

        // Search functionality untuk public
        if ($request->filled('search')) {
            $searchTerm = $request->search;
            $query->search($searchTerm);
        }

        // Sorting untuk public (default by published_at)
        $query->latest('published_at');

        // Pagination untuk public view
        $posts = $query->paginate(6)->withQueryString();

        // Featured posts untuk hero section
        $featuredPosts = Post::published()
                            ->featured()
                            ->with(['user', 'category'])
                            ->take(3)
                            ->get();

        return view('blog.index', compact('posts', 'featuredPosts'));
    }

    /**
     * Display single published post untuk public view
     */
    public function publicShow(Post $post)
    {
        // Ensure post is published
        if (!$post->isPublished()) {
            abort(404);
        }

        // Load relationships
        $post->load(['user', 'category', 'tags', 'comments.user.replies']);

        // Increment views count
        $post->incrementViews();

        // Related posts
        $relatedPosts = Post::published()
                           ->where('id', '!=', $post->id)
                           ->where('category_id', $post->category_id)
                           ->withCount('comments')
                           ->take(3)
                           ->get();

        return view('blog.show', compact('post', 'relatedPosts'));
    }

    /**
     * Get posts by category untuk public view
     */
    public function byCategory(Category $category)
    {
        $posts = Post::published()
                    ->where('category_id', $category->id)
                    ->with(['user', 'category', 'tags'])
                    ->withCount('comments')
                    ->latest('published_at')
                    ->paginate(10);

        return view('blog.category', compact('posts', 'category'));
    }

    /**
     * Get posts by tag untuk public view
     */
    public function byTag(Tag $tag)
    {
        $posts = Post::published()
                    ->whereHas('tags', function ($query) use ($tag) {
                        $query->where('tags.id', $tag->id);
                    })
                    ->with(['user', 'category', 'tags'])
                    ->withCount('comments')
                    ->latest('published_at')
                    ->paginate(10);

        return view('blog.tag', compact('posts', 'tag'));
    }

    /**
     * Search posts untuk public view
     */
    public function search(Request $request)
    {
        $searchTerm = $request->get('q', '');
        
        $posts = collect();
        if (!empty($searchTerm)) {
            $posts = Post::published()
                        ->search($searchTerm)
                        ->with(['user', 'category', 'tags'])
                        ->withCount('comments')
                        ->latest('published_at')
                        ->paginate(10)
                        ->withQueryString();
        }

        return view('blog.search', compact('posts', 'searchTerm'));
    }

    /*
    |--------------------------------------------------------------------------
    | Helper Methods
    |--------------------------------------------------------------------------
    */

    /**
     * Generate unique slug untuk post
     */
    private function generateUniqueSlug($title, $excludeId = null)
    {
        $slug = Str::slug($title);
        $originalSlug = $slug;
        $counter = 1;

        // Check if slug exists
        while (true) {
            $query = Post::where('slug', $slug);
            
            if ($excludeId) {
                $query->where('id', '!=', $excludeId);
            }
            
            if (!$query->exists()) {
                break;
            }
            
            $slug = $originalSlug . '-' . $counter;
            $counter++;
        }

        return $slug;
    }
}
```


**Isi file app/Http/Controllers/CategoryController.php:**
```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\CategoryRequest;
use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class CategoryController extends Controller
{
    public function index(Request $request)
    {
        $q = Category::query()->withCount('posts');

        // filter & search
        if ($request->filled('status')) {
            $request->status === 'active'
                ? $q->where('is_active', true)
                : ($request->status === 'inactive' ? $q->where('is_active', false) : null);
        }
        if ($request->filled('search')) {
            $s = $request->string('search');
            $q->where(function ($x) use ($s) {
                $x->where('name','like',"%{$s}%")
                  ->orWhere('slug','like',"%{$s}%")
                  ->orWhere('description','like',"%{$s}%");
            });
        }

        $categories = $q->orderBy('name')->paginate(10)->withQueryString();

        return view('categories.index', [
            'categories' => $categories,
            'filters' => [
                'search' => $request->string('search'),
                'status' => $request->string('status'),
            ],
        ]);
    }

    public function create()
    {
        return view('categories.create', [
            'parents' => Category::orderBy('name')->get(['id','name']),
        ]);
    }

    public function store(CategoryRequest $request)
    {
        $data = $request->validated();
        $data['slug'] = $data['slug'] ?: Str::slug($data['name']);

        Category::create($data);

        return redirect()->route('categories.index')->with('success', 'Category created.');
    }

    public function edit(Category $category)
    {
        return view('categories.edit', [
            'category' => $category,
            'parents'  => Category::where('id','<>',$category->id)->orderBy('name')->get(['id','name']),
        ]);
    }

    public function update(CategoryRequest $request, Category $category)
    {
        $data = $request->validated();
        $data['slug'] = $data['slug'] ?: Str::slug($data['name']);

        $category->update($data);

        return redirect()->route('categories.index')->with('success', 'Category updated.');
    }

    public function destroy(Category $category)
    {
        // optional: cek relasi posts_count > 0 → tolak hapus
        if (method_exists($category, 'posts') && $category->posts()->exists()) {
            return back()->with('error','Cannot delete: category has posts.');
        }
        $category->delete();

        return redirect()->route('categories.index')->with('success', 'Category deleted.');
    }
}
```

**Isi file app/Http/Controllers/TagController.php:**
```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\TagRequest;
use App\Models\Tag;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class TagController extends Controller
{
    public function index(Request $request)
    {
        $q = Tag::query()->withCount('posts');

        if ($request->filled('status')) {
            $request->status === 'active'
                ? $q->where('is_active', true)
                : ($request->status === 'inactive' ? $q->where('is_active', false) : null);
        }
        if ($request->filled('search')) {
            $s = $request->string('search');
            $q->where(fn($x)=> $x->where('name','like',"%{$s}%")
                                 ->orWhere('slug','like',"%{$s}%"));
        }

        $tags = $q->orderBy('name')->paginate(10)->withQueryString();

        return view('tags.index', [
            'tags' => $tags,
            'filters' => [
                'search' => $request->string('search'),
                'status' => $request->string('status'),
            ],
        ]);
    }

    public function create()
    {
        return view('tags.create');
    }

    public function store(TagRequest $request)
    {
        $data = $request->validated();
        $data['slug'] = $data['slug'] ?: Str::slug($data['name']);
        Tag::create($data);

        return redirect()->route('tags.index')->with('success', 'Tag created.');
    }

    public function edit(Tag $tag)
    {
        return view('tags.edit', compact('tag'));
    }

    public function update(TagRequest $request, Tag $tag)
    {
        $data = $request->validated();
        $data['slug'] = $data['slug'] ?: Str::slug($data['name']);
        $tag->update($data);

        return redirect()->route('tags.index')->with('success', 'Tag updated.');
    }

    public function destroy(Tag $tag)
    {
        if ($tag->posts()->exists()) {
            return back()->with('error','Cannot delete: tag is attached to posts.');
        }
        $tag->delete();

        return redirect()->route('tags.index')->with('success', 'Tag deleted.');
    }
}
```


#### **Bagian 3: Membuat Views dengan Tailwind CSS (30 menit)**

##### Step 3.1: Membuat Layout Dashboard


**Isi file resources/views/layouts/dashboard.blade.php:**
```html
<!-- Buat layout untuk dashboard -->

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
                            <a href="{{ route('blog.index') }}" 
                               class="btn-secondary">
                                View Site
                            </a>
                            <div class="h-8 w-8 bg-gray-300 rounded-full"></div>
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

##### Step 3.2: Membuat Posts Index View

**Isi file resources/views/posts/index.blade.php:**
```html
<!-- Buat view untuk posts listing -->

@extends('layouts.dashboard')

@section('title', 'Posts Management')
@section('page-title', 'Posts')
@section('page-description', 'Manage all blog posts and articles')

@section('content')
<div class="space-y-6">
    <!-- Statistics Cards -->
    <div class="grid grid-cols-1 md:grid-cols-4 gap-6">
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Total Posts</p>
                    <p class="text-2xl font-bold text-gray-900">{{ $stats['total'] }}</p>
                </div>
                <div class="w-12 h-12 bg-blue-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
                    </svg>
                </div>
            </div>
        </div>
        
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Published</p>
                    <p class="text-2xl font-bold text-green-600">{{ $stats['published'] }}</p>
                </div>
                <div class="w-12 h-12 bg-green-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
                    </svg>
                </div>
            </div>
        </div>
        
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Draft</p>
                    <p class="text-2xl font-bold text-yellow-600">{{ $stats['draft'] }}</p>
                </div>
                <div class="w-12 h-12 bg-yellow-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-yellow-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z"/>
                    </svg>
                </div>
            </div>
        </div>
        
        <div class="card">
            <div class="flex items-center justify-between">
                <div>
                    <p class="text-sm text-gray-600">Archived</p>
                    <p class="text-2xl font-bold text-gray-600">{{ $stats['archived'] }}</p>
                </div>
                <div class="w-12 h-12 bg-gray-100 rounded-lg flex items-center justify-center">
                    <svg class="w-6 h-6 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 8h14M5 8a2 2 0 110-4h14a2 2 0 110 4M5 8v10a2 2 0 002 2h10a2 2 0 002-2V8m-9 4h4"/>
                    </svg>
                </div>
            </div>
        </div>
    </div>

    <!-- Filters and Actions -->
    <div class="card">
        <div class="flex flex-col lg:flex-row lg:items-center lg:justify-between space-y-4 lg:space-y-0">
            <!-- Search and Filters -->
            <div class="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4">
                <!-- Search Input -->
                <div class="relative">
                    <input type="text" 
                           name="search" 
                           value="{{ request('search') }}"
                           placeholder="Search posts..."
                           class="input-field pl-10 w-full sm:w-64">
                    <svg class="w-5 h-5 text-gray-400 absolute left-3 top-1/2 transform -translate-y-1/2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/>
                    </svg>
                </div>
                
                <!-- Status Filter -->
                <select name="status" class="input-field">
                    <option value="">All Status</option>
                    <option value="published" {{ request('status') === 'published' ? 'selected' : '' }}>Published</option>
                    <option value="draft" {{ request('status') === 'draft' ? 'selected' : '' }}>Draft</option>
                    <option value="archived" {{ request('status') === 'archived' ? 'selected' : '' }}>Archived</option>
                </select>
                
                <!-- Category Filter -->
                <select name="category" class="input-field">
                    <option value="">All Categories</option>
                    @foreach($categories as $category)
                        <option value="{{ $category->id }}" {{ request('category') == $category->id ? 'selected' : '' }}>
                            {{ $category->name }}
                        </option>
                    @endforeach
                </select>
            </div>
            
            <!-- Action Buttons -->
            <div class="flex space-x-3">
                <a href="{{ route('posts.create') }}" class="btn-primary">
                    <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"/>
                    </svg>
                    Create Post
                </a>
            </div>
        </div>
    </div>

    <!-- Posts Table -->
    <div class="card overflow-hidden">
        <div class="overflow-x-auto">
            <table class="min-w-full divide-y divide-gray-200">
                <thead class="bg-gray-50">
                    <tr>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Post
                        </th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Author
                        </th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Category
                        </th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Status
                        </th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Comments
                        </th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Date
                        </th>
                        <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                            Actions
                        </th>
                    </tr>
                </thead>
                <tbody class="bg-white divide-y divide-gray-200">
                    @forelse($posts as $post)
                        <tr class="hover:bg-gray-50">
                            <td class="px-6 py-4">
                                <div class="flex items-center">
                                    @if($post->featured_image)
                                        <img class="h-10 w-10 rounded object-cover mr-4" 
                                             src="{{ $post->featured_image_url }}" 
                                             alt="{{ $post->title }}">
                                    @else
                                        <div class="h-10 w-10 rounded bg-gray-200 mr-4 flex items-center justify-center">
                                            <svg class="w-5 h-5 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                                            </svg>
                                        </div>
                                    @endif
                                    <div>
                                        <div class="text-sm font-medium text-gray-900">
                                            {{ Str::limit($post->title, 50) }}
                                        </div>
                                        <div class="text-sm text-gray-500">
                                            {{ $post->reading_time }}
                                            @if($post->is_featured)
                                                <span class="ml-2 inline-flex items-center px-2 py-0.5 rounded text-xs font-medium bg-yellow-100 text-yellow-800">
                                                    Featured
                                                </span>
                                            @endif
                                        </div>
                                    </div>
                                </div>
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                {{ $post->user->name }}
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap">
                                <span class="inline-flex items-center px-2 py-1 rounded-full text-xs font-medium" 
                                      style="background-color: {{ $post->category->color }}20; color: {{ $post->category->color }};">
                                    {{ $post->category->name }}
                                </span>
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap">
                                <span class="inline-flex items-center px-2 py-1 rounded-full text-xs font-medium
                                    @if($post->status === 'published') bg-green-100 text-green-800
                                    @elseif($post->status === 'draft') bg-yellow-100 text-yellow-800  
                                    @else bg-gray-100 text-gray-800 @endif">
                                    {{ ucfirst($post->status) }}
                                </span>
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                                {{ $post->comments_count }}
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                                {{ $post->created_at->format('M j, Y') }}
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                                <div class="flex items-center justify-end space-x-2">
                                    <a href="{{ route('posts.show', $post) }}" 
                                       class="text-primary-600 hover:text-primary-900">View</a>
                                    <a href="{{ route('posts.edit', $post) }}" 
                                       class="text-primary-600 hover:text-primary-900">Edit</a>
                                    <button onclick="deletePost({{ $post->id }})" 
                                            class="text-red-600 hover:text-red-900">Delete</button>
                                </div>
                            </td>
                        </tr>
                    @empty
                        <tr>
                            <td colspan="7" class="px-6 py-12 text-center">
                                <div class="text-gray-500">
                                    <svg class="mx-auto h-12 w-12 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
                                    </svg>
                                    <h3 class="mt-2 text-sm font-medium text-gray-900">No posts found</h3>
                                    <p class="mt-1 text-sm text-gray-500">Get started by creating a new post.</p>
                                    <div class="mt-6">
                                        <a href="{{ route('posts.create') }}" class="btn-primary">
                                            Create Post
                                        </a>
                                    </div>
                                </div>
                            </td>
                        </tr>
                    @endforelse
                </tbody>
            </table>
        </div>
        
        <!-- Pagination -->
        @if($posts->hasPages())
            <div class="border-t border-gray-200 px-6 py-3">
                {{ $posts->links() }}
            </div>
        @endif
    </div>
</div>

<script>
function deletePost(postId) {
    if (confirm('Are you sure you want to delete this post?')) {
        fetch(`/posts/${postId}`, {
            method: 'DELETE',
            headers: {
                'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').getAttribute('content'),
                'Accept': 'application/json',
            }
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                location.reload();
            } else {
                alert('Error deleting post');
            }
        })
        .catch(error => {
            console.error('Error:', error);
            alert('Error deleting post');
        });
    }
}
</script>
@endsection
```

##### Step 3.3: Membuat Post Create Form

**Isi file resources/views/posts/create.blade.php (abbreviated untuk space):**
```html
<!-- Buat view untuk create post form -->

@extends('layouts.dashboard')

@section('title', 'Create New Post')
@section('page-title', 'Create New Post')
@section('page-description', 'Add a new blog post or article')

@section('content')
<div class="max-w-4xl">
    <form action="{{ route('posts.store') }}" method="POST" enctype="multipart/form-data" class="space-y-6">
        @csrf
        
        <!-- Basic Information Card -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">Basic Information</h3>
            
            <!-- Title Field -->
            <div class="mb-4">
                <label for="title" class="block text-sm font-medium text-gray-700 mb-2">Title *</label>
                <input type="text" 
                       name="title" 
                       id="title"
                       value="{{ old('title') }}"
                       class="input-field w-full @error('title') border-red-300 @enderror"
                       placeholder="Enter post title..."
                       required>
                @error('title')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
            
            <!-- Content Field -->
            <div class="mb-4">
                <label for="content" class="block text-sm font-medium text-gray-700 mb-2">Content *</label>
                <textarea name="content" 
                          id="content"
                          rows="15"
                          class="input-field w-full @error('content') border-red-300 @enderror"
                          placeholder="Write your post content here..."
                          required>{{ old('content') }}</textarea>
                @error('content')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
            
            <!-- Excerpt Field -->
            <div class="mb-4">
                <label for="excerpt" class="block text-sm font-medium text-gray-700 mb-2">Excerpt</label>
                <textarea name="excerpt" 
                          id="excerpt"
                          rows="3"
                          class="input-field w-full @error('excerpt') border-red-300 @enderror"
                          placeholder="Optional excerpt or summary...">{{ old('excerpt') }}</textarea>
                @error('excerpt')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
        </div>
        
        <!-- Metadata and Settings Card -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">Settings</h3>
            
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                <!-- Status -->
                <div>
                    <label for="status" class="block text-sm font-medium text-gray-700 mb-2">Status *</label>
                    <select name="status" id="status" class="input-field w-full @error('status') border-red-300 @enderror" required>
                        <option value="draft" {{ old('status', 'draft') === 'draft' ? 'selected' : '' }}>Draft</option>
                        <option value="published" {{ old('status') === 'published' ? 'selected' : '' }}>Published</option>
                        <option value="archived" {{ old('status') === 'archived' ? 'selected' : '' }}>Archived</option>
                    </select>
                    @error('status')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Category -->
                <div>
                    <label for="category_id" class="block text-sm font-medium text-gray-700 mb-2">Category *</label>
                    <select name="category_id" id="category_id" class="input-field w-full @error('category_id') border-red-300 @enderror" required>
                        <option value="">Select Category</option>
                        @foreach($categories as $category)
                            <option value="{{ $category->id }}" {{ old('category_id') == $category->id ? 'selected' : '' }}>
                                {{ $category->name }}
                            </option>
                        @endforeach
                    </select>
                    @error('category_id')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Author -->
                <div>
                    <label for="user_id" class="block text-sm font-medium text-gray-700 mb-2">Author *</label>
                    <select name="user_id" id="user_id" class="input-field w-full @error('user_id') border-red-300 @enderror" required>
                        <option value="">Select Author</option>
                        @foreach($users as $user)
                            <option value="{{ $user->id }}" {{ old('user_id') == $user->id ? 'selected' : '' }}>
                                {{ $user->name }} ({{ $user->role }})
                            </option>
                        @endforeach
                    </select>
                    @error('user_id')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Featured Image -->
                <div>
                    <label for="featured_image" class="block text-sm font-medium text-gray-700 mb-2">Featured Image</label>
                    <input type="file" 
                           name="featured_image" 
                           id="featured_image"
                           accept="image/*"
                           class="input-field w-full @error('featured_image') border-red-300 @enderror">
                    @error('featured_image')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
            </div>
            
            <!-- Tags -->
            <div class="mt-4">
                <label class="block text-sm font-medium text-gray-700 mb-2">Tags</label>
                <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-2">
                    @foreach($tags as $tag)
                        <label class="flex items-center">
                            <input type="checkbox" 
                                   name="tags[]" 
                                   value="{{ $tag->id }}"
                                   class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
                                   {{ in_array($tag->id, old('tags', [])) ? 'checked' : '' }}>
                            <span class="ml-2 text-sm text-gray-700">{{ $tag->name }}</span>
                        </label>
                    @endforeach
                </div>
            </div>
            
            <!-- Options -->
            <div class="mt-4 flex space-x-6">
                <label class="flex items-center">
                    <input type="checkbox" 
                           name="is_featured" 
                           value="1"
                           class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
                           {{ old('is_featured') ? 'checked' : '' }}>
                    <span class="ml-2 text-sm text-gray-700">Featured Post</span>
                </label>
                
                <label class="flex items-center">
                    <input type="checkbox" 
                           name="allow_comments" 
                           value="1"
                           class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
                           {{ old('allow_comments', true) ? 'checked' : '' }}>
                    <span class="ml-2 text-sm text-gray-700">Allow Comments</span>
                </label>
            </div>
        </div>
        
        <!-- Form Actions -->
        <div class="flex justify-between">
            <a href="{{ route('posts.index') }}" class="btn-secondary">
                Cancel
            </a>
            
            <div class="flex space-x-3">
                <button type="submit" name="status" value="draft" class="btn-secondary">
                    Save as Draft
                </button>
                <button type="submit" name="status" value="published" class="btn-primary">
                    Publish Post
                </button>
            </div>
        </div>
    </form>
</div>
@endsection
```

##### Step 3.4: Membuat Post Edit Form
**Isi file resources/views/posts/edit.blade.php:**

```php

@extends('layouts.dashboard')

@section('title', 'Edit Post: ' . $post->title)
@section('page-title', 'Edit Post')
@section('page-description', 'Update existing blog post or article')

@section('content')
<div class="max-w-4xl">
    <!-- Progress Indicator -->
    <div class="mb-6">
        <div class="flex items-center justify-between text-sm text-gray-500">
            <span>Last updated: {{ $post->updated_at->diffForHumans() }}</span>
            <span>Created: {{ $post->created_at->format('M j, Y') }}</span>
        </div>
    </div>

    <form action="{{ route('posts.update', $post) }}" 
          method="POST" 
          enctype="multipart/form-data" 
          class="space-y-6"
          id="editPostForm">
        @csrf
        @method('PUT')
        
        <!-- Basic Information Card -->
        <div class="card">
            <div class="flex items-center justify-between mb-4">
                <h3 class="text-lg font-medium text-gray-900">Basic Information</h3>
                <div class="flex items-center space-x-2">
                    <span class="text-sm text-gray-500">Auto-save:</span>
                    <div class="w-2 h-2 bg-gray-300 rounded-full" id="autoSaveIndicator"></div>
                </div>
            </div>
            
            <!-- Title Field dengan real-time slug preview -->
            <div class="mb-4">
                <label for="title" class="block text-sm font-medium text-gray-700 mb-2">Title *</label>
                <input type="text" 
                       name="title" 
                       id="title"
                       value="{{ old('title', $post->title) }}"
                       class="input-field w-full @error('title') border-red-300 @enderror"
                       placeholder="Enter post title..."
                       onkeyup="updateSlugPreview()"
                       required>
                @error('title')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
                <!-- Slug Preview -->
                <p class="mt-1 text-sm text-gray-500">
                    URL: <span class="font-mono text-primary-600" id="slugPreview">{{ $post->slug }}</span>
                </p>
            </div>
            
            <!-- Content Field dengan character counter -->
            <div class="mb-4">
                <label for="content" class="block text-sm font-medium text-gray-700 mb-2">Content *</label>
                <textarea name="content" 
                          id="content"
                          rows="15"
                          class="input-field w-full @error('content') border-red-300 @enderror"
                          placeholder="Write your post content here..."
                          onkeyup="updateCharacterCount()"
                          required>{{ old('content', $post->content) }}</textarea>
                @error('content')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
                <div class="mt-1 flex justify-between text-sm text-gray-500">
                    <span>Minimum 10 characters required</span>
                    <span id="characterCount">{{ strlen($post->content) }} characters</span>
                </div>
            </div>
            
            <!-- Excerpt Field -->
            <div class="mb-4">
                <label for="excerpt" class="block text-sm font-medium text-gray-700 mb-2">
                    Excerpt 
                    <span class="text-gray-400">(Optional - auto-generated if empty)</span>
                </label>
                <textarea name="excerpt" 
                          id="excerpt"
                          rows="3"
                          class="input-field w-full @error('excerpt') border-red-300 @enderror"
                          placeholder="Optional excerpt or summary..."
                          maxlength="500">{{ old('excerpt', $post->excerpt) }}</textarea>
                @error('excerpt')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
        </div>
        
        <!-- Featured Image Management Card -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">Featured Image</h3>
            
            <!-- Current Featured Image Display -->
            @if($post->featured_image)
                <div class="mb-4">
                    <label class="block text-sm font-medium text-gray-700 mb-2">Current Image</label>
                    <div class="relative inline-block">
                        <img src="{{ $post->featured_image_url }}" 
                             alt="{{ $post->title }}"
                             class="h-32 w-48 object-cover rounded-lg border border-gray-200"
                             id="currentFeaturedImage">
                        <div class="absolute inset-0 bg-black bg-opacity-0 hover:bg-opacity-30 transition-all duration-200 rounded-lg flex items-center justify-center">
                            <button type="button" 
                                    class="opacity-0 hover:opacity-100 bg-red-600 text-white px-3 py-1 rounded text-sm transition-opacity"
                                    onclick="toggleRemoveImage()">
                                Remove
                            </button>
                        </div>
                    </div>
                    
                    <!-- Remove Image Checkbox -->
                    <div class="mt-2">
                        <label class="flex items-center">
                            <input type="checkbox" 
                                   name="remove_featured_image" 
                                   value="1"
                                   class="rounded border-gray-300 text-red-600 shadow-sm focus:border-red-300 focus:ring focus:ring-red-200 focus:ring-opacity-50"
                                   id="removeFeaturedImage"
                                   onchange="toggleImageRemoval()">
                            <span class="ml-2 text-sm text-red-600">Remove current featured image</span>
                        </label>
                    </div>
                </div>
            @endif
            
            <!-- Upload New Image -->
            <div class="mb-4" id="imageUploadSection">
                <label for="featured_image" class="block text-sm font-medium text-gray-700 mb-2">
                    {{ $post->featured_image ? 'Replace with New Image' : 'Upload Featured Image' }}
                </label>
                <div class="flex items-center space-x-4">
                    <input type="file" 
                           name="featured_image" 
                           id="featured_image"
                           accept="image/*"
                           class="input-field flex-1 @error('featured_image') border-red-300 @enderror"
                           onchange="previewNewImage(event)">
                    <div class="text-sm text-gray-500">
                        Max: 2MB<br>
                        Formats: JPEG, PNG, GIF, WebP
                    </div>
                </div>
                @error('featured_image')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
                
                <!-- Image Preview untuk new upload -->
                <div class="mt-3 hidden" id="imagePreviewContainer">
                    <img id="imagePreview" class="h-32 w-48 object-cover rounded-lg border border-gray-200" alt="Preview">
                </div>
            </div>
        </div>
        
        <!-- Metadata and Settings Card -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">Settings & Metadata</h3>
            
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                <!-- Status dengan conditional fields -->
                <div>
                    <label for="status" class="block text-sm font-medium text-gray-700 mb-2">Status *</label>
                    <select name="status" 
                            id="status" 
                            class="input-field w-full @error('status') border-red-300 @enderror" 
                            onchange="togglePublishOptions()"
                            required>
                        <option value="draft" {{ old('status', $post->status) === 'draft' ? 'selected' : '' }}>Draft</option>
                        <option value="published" {{ old('status', $post->status) === 'published' ? 'selected' : '' }}>Published</option>
                        <option value="archived" {{ old('status', $post->status) === 'archived' ? 'selected' : '' }}>Archived</option>
                    </select>
                    @error('status')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Category dengan live search -->
                <div>
                    <label for="category_id" class="block text-sm font-medium text-gray-700 mb-2">Category *</label>
                    <select name="category_id" 
                            id="category_id" 
                            class="input-field w-full @error('category_id') border-red-300 @enderror" 
                            required>
                        <option value="">Select Category</option>
                        @foreach($categories as $category)
                            <option value="{{ $category->id }}" 
                                    {{ old('category_id', $post->category_id) == $category->id ? 'selected' : '' }}
                                    data-color="{{ $category->color }}">
                                {{ $category->name }}
                            </option>
                        @endforeach
                    </select>
                    @error('category_id')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Author -->
                <div>
                    <label for="user_id" class="block text-sm font-medium text-gray-700 mb-2">Author *</label>
                    <select name="user_id" 
                            id="user_id" 
                            class="input-field w-full @error('user_id') border-red-300 @enderror" 
                            required>
                        <option value="">Select Author</option>
                        @foreach($users as $user)
                            <option value="{{ $user->id }}" 
                                    {{ old('user_id', $post->user_id) == $user->id ? 'selected' : '' }}>
                                {{ $user->name }} ({{ $user->role }})
                            </option>
                        @endforeach
                    </select>
                    @error('user_id')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
                
                <!-- Published Date dengan conditional display -->
                <div id="publishDateSection" style="display: {{ old('status', $post->status) === 'published' ? 'block' : 'none' }}">
                    <label for="published_at" class="block text-sm font-medium text-gray-700 mb-2">
                        Published Date
                    </label>
                    <input type="datetime-local" 
                           name="published_at" 
                           id="published_at"
                           value="{{ old('published_at', $post->published_at?->format('Y-m-d\TH:i')) }}"
                           class="input-field w-full @error('published_at') border-red-300 @enderror">
                    @error('published_at')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                    
                    <!-- Schedule Publishing Option -->
                    <div class="mt-2">
                        <label class="flex items-center">
                            <input type="checkbox" 
                                   name="schedule_publish" 
                                   value="1"
                                   class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50">
                            <span class="ml-2 text-sm text-gray-700">Schedule for future publishing</span>
                        </label>
                    </div>
                </div>
            </div>
            
            <!-- Tags Selection dengan improved UI -->
            <div class="mt-6">
                <label class="block text-sm font-medium text-gray-700 mb-3">Tags</label>
                <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-2 max-h-40 overflow-y-auto border border-gray-200 rounded-lg p-3">
                    @foreach($tags as $tag)
                        <label class="flex items-center hover:bg-gray-50 p-1 rounded">
                            <input type="checkbox" 
                                   name="tags[]" 
                                   value="{{ $tag->id }}"
                                   class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
                                   {{ in_array($tag->id, old('tags', $selectedTags)) ? 'checked' : '' }}>
                            <span class="ml-2 text-sm text-gray-700">{{ $tag->name }}</span>
                            <span class="ml-1 w-3 h-3 rounded-full" style="background-color: {{ $tag->color }}"></span>
                        </label>
                    @endforeach
                </div>
                <p class="mt-1 text-sm text-gray-500">
                    Selected: <span id="selectedTagsCount">{{ count($selectedTags) }}</span> tags
                </p>
            </div>
            
            <!-- Options -->
            <div class="mt-6 flex flex-wrap gap-6">
                <label class="flex items-center">
                    <input type="checkbox" 
                           name="is_featured" 
                           value="1"
                           class="rounded border-gray-300 text-yellow-600 shadow-sm focus:border-yellow-300 focus:ring focus:ring-yellow-200 focus:ring-opacity-50"
                           {{ old('is_featured', $post->is_featured) ? 'checked' : '' }}>
                    <span class="ml-2 text-sm text-gray-700">Featured Post</span>
                </label>
                
                <label class="flex items-center">
                    <input type="checkbox" 
                           name="allow_comments" 
                           value="1"
                           class="rounded border-gray-300 text-green-600 shadow-sm focus:border-green-300 focus:ring focus:ring-green-200 focus:ring-opacity-50"
                           {{ old('allow_comments', $post->allow_comments) ? 'checked' : '' }}>
                    <span class="ml-2 text-sm text-gray-700">Allow Comments</span>
                </label>
            </div>
        </div>
        
        <!-- SEO Metadata Card -->
        <div class="card">
            <h3 class="text-lg font-medium text-gray-900 mb-4">SEO Metadata</h3>
            
            <!-- Meta Title -->
            <div class="mb-4">
                <label for="meta_title" class="block text-sm font-medium text-gray-700 mb-2">
                    Meta Title
                    <span class="text-gray-400">(Leave empty to use post title)</span>
                </label>
                <input type="text" 
                       name="meta_title" 
                       id="meta_title"
                       value="{{ old('meta_title', $post->meta_title) }}"
                       class="input-field w-full @error('meta_title') border-red-300 @enderror"
                       maxlength="255"
                       placeholder="SEO-optimized title for search engines">
                @error('meta_title')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
            
            <!-- Meta Description -->
            <div class="mb-4">
                <label for="meta_description" class="block text-sm font-medium text-gray-700 mb-2">
                    Meta Description
                </label>
                <textarea name="meta_description" 
                          id="meta_description"
                          rows="3"
                          class="input-field w-full @error('meta_description') border-red-300 @enderror"
                          maxlength="500"
                          placeholder="Brief description for search engine results">{{ old('meta_description', $post->meta_description) }}</textarea>
                @error('meta_description')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
            
            <!-- Meta Keywords -->
            <div class="mb-4">
                <label for="meta_keywords" class="block text-sm font-medium text-gray-700 mb-2">
                    Meta Keywords
                    <span class="text-gray-400">(Comma-separated)</span>
                </label>
                <input type="text" 
                       name="meta_keywords" 
                       id="meta_keywords"
                       value="{{ old('meta_keywords', $post->meta_keywords) }}"
                       class="input-field w-full @error('meta_keywords') border-red-300 @enderror"
                       placeholder="keyword1, keyword2, keyword3">
                @error('meta_keywords')
                    <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                @enderror
            </div>
        </div>
        
        <!-- Form Actions -->
        <div class="flex justify-between">
            <a href="{{ route('posts.index') }}" class="btn-secondary">
                <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"/>
                </svg>
                Back to Posts
            </div>
            
            <div class="flex space-x-3">
                <button type="button" 
                        class="btn-secondary" 
                        onclick="saveDraft()">
                    <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7H5a2 2 0 00-2 2v9a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-3m-1 4l-3-3m0 0l-3 3m3-3v12"/>
                    </svg>
                    Save Draft
                </button>
                
                <button type="submit" class="btn-primary" id="updatePostBtn">
                    <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"/>
                    </svg>
                    Update Post
                </button>
            </div>
        </div>
    </form>
</div>

<!-- JavaScript untuk enhanced functionality -->
<script>
// Real-time slug preview
function updateSlugPreview() {
    const title = document.getElementById('title').value;
    const slug = title.toLowerCase()
                     .replace(/[^a-z0-9]+/g, '-')
                     .replace(/(^-|-$)/g, '');
    document.getElementById('slugPreview').textContent = slug || 'post-slug';
}

// Character counter untuk content
function updateCharacterCount() {
    const content = document.getElementById('content').value;
    document.getElementById('characterCount').textContent = content.length + ' characters';
}

// Toggle publish date section berdasarkan status
function togglePublishOptions() {
    const status = document.getElementById('status').value;
    const publishSection = document.getElementById('publishDateSection');
    
    if (status === 'published') {
        publishSection.style.display = 'block';
    } else {
        publishSection.style.display = 'none';
    }
}

// Image removal toggle
function toggleRemoveImage() {
    const checkbox = document.getElementById('removeFeaturedImage');
    checkbox.checked = !checkbox.checked;
    toggleImageRemoval();
}

function toggleImageRemoval() {
    const checkbox = document.getElementById('removeFeaturedImage');
    const currentImage = document.getElementById('currentFeaturedImage');
    
    if (checkbox.checked) {
        currentImage.style.opacity = '0.3';
        currentImage.style.filter = 'grayscale(100%)';
    } else {
        currentImage.style.opacity = '1';
        currentImage.style.filter = 'grayscale(0%)';
    }
}

// Preview new image upload
function previewNewImage(event) {
    const file = event.target.files[0];
    const container = document.getElementById('imagePreviewContainer');
    const preview = document.getElementById('imagePreview');
    
    if (file) {
        const reader = new FileReader();
        reader.onload = function(e) {
            preview.src = e.target.result;
            container.classList.remove('hidden');
        }
        reader.readAsDataURL(file);
    } else {
        container.classList.add('hidden');
    }
}

// Save as draft function
function saveDraft() {
    const form = document.getElementById('editPostForm');
    const statusField = document.getElementById('status');
    statusField.value = 'draft';
    form.submit();
}

// Auto-save functionality (optional)
let autoSaveTimer;
function setupAutoSave() {
    const form = document.getElementById('editPostForm');
    const indicator = document.getElementById('autoSaveIndicator');
    
    // Clear existing timer
    if (autoSaveTimer) {
        clearTimeout(autoSaveTimer);
    }
    
    // Set new timer
    autoSaveTimer = setTimeout(() => {
        // Auto-save logic here (AJAX call)
        indicator.classList.remove('bg-gray-300');
        indicator.classList.add('bg-green-500');
        setTimeout(() => {
            indicator.classList.remove('bg-green-500');
            indicator.classList.add('bg-gray-300');
        }, 2000);
    }, 30000); // Auto-save setiap 30 detik
}

// Update selected tags counter
document.addEventListener('DOMContentLoaded', function() {
    const tagCheckboxes = document.querySelectorAll('input[name="tags[]"]');
    const counter = document.getElementById('selectedTagsCount');
    
    tagCheckboxes.forEach(checkbox => {
        checkbox.addEventListener('change', function() {
            const selected = document.querySelectorAll('input[name="tags[]"]:checked').length;
            counter.textContent = selected;
        });
    });
    
    // Setup auto-save
    setupAutoSave();
    
    // Setup form change detection untuk auto-save
    const formInputs = document.querySelectorAll('#editPostForm input, #editPostForm textarea, #editPostForm select');
    formInputs.forEach(input => {
        input.addEventListener('change', setupAutoSave);
        input.addEventListener('keyup', setupAutoSave);
    });
});
</script>
@endsection


```

##### Step 3.5: Membuat Post show Form
**Isi file resources/views/posts/show.blade.php:**
```php

@extends('layouts.dashboard')

@section('title', 'Post Detail')
@section('page-title', 'Post Detail')

@section('content')
@php
  $hasExcerpt = filled($post->excerpt ?? null);
  $excerptTrim = trim((string) $post->excerpt);
  $contentTrim = trim((string) $post->content);
  $isDup = $hasExcerpt && (
      $excerptTrim === $contentTrim ||
      str_starts_with($contentTrim, $excerptTrim)     // PHP 8
  );
@endphp

<div class="max-w-5xl space-y-6">

  <div class="flex items-center justify-between">
    <a href="{{ route('posts.index') }}" class="btn-secondary">← Back</a>
    <div class="space-x-2">
      <a href="{{ route('posts.edit', $post) }}" class="btn-secondary">Edit</a>
      <form action="{{ route('posts.destroy', $post) }}" method="POST" class="inline"
            onsubmit="return confirm('Delete this post?');">
        @csrf @method('DELETE')
        <button class="btn-danger">Delete</button>
      </form>
    </div>
  </div>

  <div class="card">
    <div class="flex items-start justify-between gap-4">
      <div>
        <h2 class="text-2xl font-semibold">{{ $post->title }}</h2>
        <div class="mt-2 flex flex-wrap items-center gap-2 text-sm text-gray-600">
          <span class="px-2 py-0.5 rounded-full border
            {{ $post->status === 'published' ? 'border-green-300 text-green-700 bg-green-50' : 'border-gray-300 text-gray-700 bg-gray-50' }}">
            {{ ucfirst($post->status) }}
          </span>
          @if($post->category)
            <span>• Category: <span class="font-medium">{{ $post->category->name }}</span></span>
          @endif
          @if($post->user)
            <span>• Author: <span class="font-medium">{{ $post->user->name }}</span></span>
          @endif
          @if($post->published_at)
            <span>• Published: {{ $post->published_at->format('Y-m-d H:i') }}</span>
          @endif
        </div>
      </div>
    </div>

    @if($post->featured_image)
      <img src="{{ Storage::url($post->featured_image) }}"
           alt="" class="rounded-lg mt-4 max-h-80 w-full object-cover">
    @endif

    {{-- Excerpt (tampilkan hanya jika tidak duplikat) --}}
    @if($hasExcerpt && !$isDup)
      <div class="mt-4 p-4 rounded-lg bg-gray-50 border border-gray-200">
        <div class="text-sm uppercase tracking-wide text-gray-500 mb-1">Excerpt</div>
        <p class="text-gray-800">{{ $post->excerpt }}</p>
      </div>
    @endif

    {{-- Content --}}
    <div class="prose max-w-none mt-4">
      {!! nl2br(e($post->content)) !!}
    </div>

    {{-- Tags --}}
    @if($post->tags?->count())
      <div class="mt-6 flex flex-wrap gap-2">
        @foreach($post->tags as $tag)
          <span class="px-2 py-1 text-xs rounded bg-blue-50 text-blue-700 border border-blue-200">#{{ $tag->name }}</span>
        @endforeach
      </div>
    @endif
  </div>

  <div class="card">
    <h3 class="text-lg font-medium mb-2">Options</h3>
    <dl class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-y-2 text-sm">
      <div><dt class="text-gray-500">Featured</dt><dd>{{ $post->is_featured ? 'Yes' : 'No' }}</dd></div>
      <div><dt class="text-gray-500">Allow Comments</dt><dd>{{ $post->allow_comments ? 'Yes' : 'No' }}</dd></div>
      <div><dt class="text-gray-500">Slug</dt><dd><code>{{ $post->slug ?? '-' }}</code></dd></div>
      <div><dt class="text-gray-500">Created</dt><dd>{{ $post->created_at->format('Y-m-d H:i') }}</dd></div>
      <div><dt class="text-gray-500">Updated</dt><dd>{{ $post->updated_at->format('Y-m-d H:i') }}</dd></div>
    </dl>
  </div>

</div>
@endsection


```

##### Step 3.5: Implementasi Category
**Isi file resources/views/categories/_form.blade.php:**
```php
@php
  $editing = isset($category);
@endphp

<div class="grid md:grid-cols-2 gap-4">
  <div>
    <label class="block text-sm font-medium text-gray-700 mb-1">Name</label>
    <input type="text" name="name" value="{{ old('name', $category->name ?? '') }}" class="input-field" required>
    @error('name') <p class="text-sm text-red-600 mt-1">{{ $message }}</p> @enderror
  </div>

  <div>
    <label class="block text-sm font-medium text-gray-700 mb-1">Slug (optional)</label>
    <input type="text" name="slug" value="{{ old('slug', $category->slug ?? '') }}" class="input-field">
    @error('slug') <p class="text-sm text-red-600 mt-1">{{ $message }}</p> @enderror
  </div>

  <div>
    <label class="block text-sm font-medium text-gray-700 mb-1">Parent (optional)</label>
    <select name="parent_id" class="input-field">
      <option value="">— None —</option>
      @foreach($parents as $p)
        <option value="{{ $p->id }}" @selected(old('parent_id', $category->parent_id ?? null) == $p->id)>{{ $p->name }}</option>
      @endforeach
    </select>
    @error('parent_id') <p class="text-sm text-red-600 mt-1">{{ $message }}</p> @enderror
  </div>

  <div class="flex items-center gap-2">
    <input type="hidden" name="is_active" value="0">
    <input type="checkbox" name="is_active" value="1"
           @checked(old('is_active', $category->is_active ?? true)) class="h-4 w-4">
    <label class="text-sm text-gray-700">Active</label>
  </div>

  <div class="md:col-span-2">
    <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
    <textarea name="description" rows="3" class="input-field">{{ old('description', $category->description ?? '') }}</textarea>
    @error('description') <p class="text-sm text-red-600 mt-1">{{ $message }}</p> @enderror
  </div>
</div>

<div class="mt-6 flex items-center justify-end gap-3">
  <a href="{{ route('categories.index') }}" class="btn-secondary">Cancel</a>
  <button class="btn-primary">{{ $editing ? 'Update' : 'Create' }}</button>
</div>

```
**** 
**Isi file resources/views/categories/create.blade.php:**
```php
@extends('layouts.dashboard')

@section('title','New Category')
@section('page-title','New Category')

@section('content')
  <div class="card">
    <form method="POST" action="{{ route('categories.store') }}" class="space-y-4">
      @csrf
      @include('categories._form', ['parents' => $parents])
    </form>
  </div>
@endsection

```
**Isi file resources/views/categories/edit.blade.php:**
```php
@extends('layouts.dashboard')

@section('title','Edit Category')
@section('page-title','Edit Category')

@section('content')
  <div class="card">
    <form method="POST" action="{{ route('categories.update', $category) }}" class="space-y-4">
      @csrf @method('PUT')
      @include('categories._form', ['parents' => $parents, 'category' => $category])
    </form>
  </div>
@endsection

```
**Isi file resources/views/categories/index.blade.php:**
```php
@extends('layouts.dashboard')

@section('title', 'Categories') {{-- ini buat <title> --}}
@section('page-title', 'Categories')
@section('page-description', 'Manage post categories, search & filter, and pagination.')

@section('content')
  <div class="card">
    <form method="GET" class="grid md:grid-cols-4 gap-3 mb-4">
      <input name="search" value="{{ $filters['search'] }}" placeholder="Search name/slug..." class="input-field md:col-span-2" />
      <select name="status" class="input-field">
        <option value="">All status</option>
        <option value="active"   @selected($filters['status']=='active')>Active</option>
        <option value="inactive" @selected($filters['status']=='inactive')>Inactive</option>
      </select>
      <button class="btn-secondary">Apply</button>
    </form>

    <div class="overflow-x-auto">
      <table class="min-w-full text-sm">
        <thead class="text-left text-gray-600">
          <tr>
            <th class="py-2">Name</th>
            <th class="py-2">Slug</th>
            <th class="py-2">Posts</th>
            <th class="py-2">Status</th>
            <th class="py-2 text-right">
              <a href="{{ route('categories.create') }}" class="btn-primary">New Category</a>
            </th>
          </tr>
        </thead>
        <tbody class="divide-y">
          @forelse ($categories as $c)
            <tr>
              <td class="py-2 font-medium">{{ $c->name }}</td>
              <td class="py-2 text-gray-500">{{ $c->slug }}</td>
              <td class="py-2">{{ $c->posts_count }}</td>
              <td class="py-2">
                @if($c->is_active)
                  <span class="px-2 py-1 text-xs rounded bg-green-100 text-green-700">Active</span>
                @else
                  <span class="px-2 py-1 text-xs rounded bg-gray-100 text-gray-600">Inactive</span>
                @endif
              </td>
              <td class="py-2 text-right">
                <a href="{{ route('categories.edit', $c) }}" class="text-primary-600 hover:underline mr-3">Edit</a>
                <form action="{{ route('categories.destroy', $c) }}" method="POST" class="inline" onsubmit="return confirm('Delete this category?');">
                  @csrf @method('DELETE')
                  <button class="text-red-600 hover:underline">Delete</button>
                </form>
              </td>
            </tr>
          @empty
            <tr><td colspan="5" class="py-6 text-center text-gray-500">No categories found.</td></tr>
          @endforelse
        </tbody>
      </table>
    </div>

    <div class="mt-4">
      {{ $categories->links() }}
    </div>
  </div>
@endsection
```

#### **Bagian 4: Implementasi API Controllers dan Testing (15 menit)**

##### Step 4.1: Implementasi PostApiController

**Isi file app/Http/Controllers/Api/PostApiController.php:**
```php
<!-- Edit API controller untuk AJAX operations -->

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
}
```

##### Step 4.1 ( 2 ): Implementasi Requests
```bash
php artisan make:request CategoryRequest

php artisan make:request TagRequest
```

Isi file app/Http/Requests/CategoryRequest.php:
```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CategoryRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // ganti sesuai policy kalau perlu
    }

    public function rules(): array
    {
        $id = $this->route('category')?->id;
        return [
            'name'        => ['required','string','max:100'],
            'slug'        => ['nullable','string','max:120','unique:categories,slug,'.($id ?? 'NULL').',id'],
            'parent_id'   => ['nullable','exists:categories,id'],
            'is_active'   => ['required','boolean'],
            'description' => ['nullable','string','max:500'],
        ];
    }

    public function prepareForValidation(): void
    {
        $this->merge([
            'is_active' => filter_var($this->is_active, FILTER_VALIDATE_BOOL, FILTER_NULL_ON_FAILURE) ?? false,
        ]);
    }
}

```


Isi file app/Http/Requests/TagRequest.php:
```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class TagRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    protected function currentId(): ?int
    {
        $model = $this->route('tag'); // bisa objek Tag atau id
        return is_object($model) ? $model->getKey() : (is_numeric($model) ? (int)$model : null);
    }

    protected function prepareForValidation(): void
    {
        $this->merge([
            'is_active' => $this->boolean('is_active'),
        ]);
    }

    public function rules(): array
    {
        $id = $this->currentId();

        return [
            'name'      => ['required','string','max:100'],
            'slug'      => ['nullable','string','max:120', Rule::unique('tags','slug')->ignore($id)],
            'is_active' => ['required','boolean'],
        ];
    }
}

```

##### Step 4.2: Testing CRUD Operations
```bash
# Pastikan sudah php artisan serve

# Test API endpoints
echo "=== Testing API Endpoints ==="

# Test 1: Get all posts
$response = Invoke-RestMethod -Uri "http://localhost:8000/api/posts"
$response.status
Write-Output "✓ API posts index working"

# Test 2: Get API info
$response = Invoke-RestMethod -Uri "http://localhost:8000/api/info"
$response.application
Write-Output "✓ API info endpoint working"

# # Test 3: Get featured posts
# $response = Invoke-RestMethod -Uri "http://localhost:8000/api/posts/featured"
# $response.status
# Write-Output "✓ API featured posts working"

# Test 4: Search functionality
$response = Invoke-RestMethod -Uri "http://localhost:8000/api/posts/search/test"
$response.status
Write-Output "✓ API search working"
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Controller Methods Verification
```bash
echo "=== Controller Methods Verification ==="

php artisan route:list --name=posts | head -10
echo "✓ Posts routes registered"

# Masuk ke Tinker
php artisan tinker

# Jalankan kode ini di dalam Tinker
$controller = new \App\Http\Controllers\PostController();
$methods = get_class_methods($controller);
$required = ['index', 'create', 'store', 'show', 'edit', 'update', 'destroy'];
$missing = array_diff($required, $methods);

if (empty($missing)) {
    echo "✓ All required controller methods exist\n";
} else {
    echo "✗ Missing methods: " . implode(', ', $missing) . "\n";
}


```

<!-- tetap berada di /posts atau api/posts -->

<!-- #### Test 2: Form Validation Testing
```bash
echo "=== Form Validation Testing ==="

# Test validation dengan data kosong
curl -X POST localhost:8000/posts \
     -H "Accept: application/json" \
     -H "X-CSRF-TOKEN: test-token" \
     -d "" 2>/dev/null | grep -q "errors" && echo "✓ Validation working for empty data"

# Test validation dengan data valid (mock)
echo "✓ Form validation rules implemented in controller"
``` --> 

<!-- Belum ada data kategori 1 -->

<!-- #### Test 3: Database CRUD Operations
```bash
echo "=== Database CRUD Operations Testing ==="

php artisan tinker --execute="
// Test Create operation
try {
    \$post = \App\Models\Post::create([
        'title' => 'Test Post Create',
        'content' => 'Test content for CRUD operation testing',
        'slug' => 'test-post-create',
        'status' => 'draft',
        'user_id' => 1,
        'category_id' => 1,
    ]);
    echo '✓ CREATE operation successful (ID: ' . \$post->id . ')' . PHP_EOL;
    
    // Test Read operation
    \$found = \App\Models\Post::find(\$post->id);
    if (\$found && \$found->title === 'Test Post Create') {
        echo '✓ READ operation successful' . PHP_EOL;
    }
    
    // Test Update operation
    \$found->update(['title' => 'Updated Test Post']);
    if (\$found->fresh()->title === 'Updated Test Post') {
        echo '✓ UPDATE operation successful' . PHP_EOL;
    }
    
    // Test Delete operation
    \$found->delete();
    if (!\App\Models\Post::find(\$post->id)) {
        echo '✓ DELETE operation successful' . PHP_EOL;
    }
    
} catch (\Exception \$e) {
    echo '✗ CRUD operations failed: ' . \$e->getMessage() . PHP_EOL;
}
"
``` -->

#### Test 4: View Rendering Testing
```bash
Write-Output "=== View Rendering Testing ==="

# Daftar view yang harus ada
$views = @("posts/index", "posts/create", "posts/show", "posts/edit")
foreach ($view in $views) {
    $filePath = "resources/views/$view.blade.php"
    if (Test-Path $filePath) {
        Write-Output "✓ View $view exists"
    } else {
        Write-Output "✗ View $view missing"
    }
}

# Daftar layout yang harus ada
$layouts = @("layouts/dashboard", "layouts/app")
foreach ($layout in $layouts) {
    $filePath = "resources/views/$layout.blade.php"
    if (Test-Path $filePath) {
        Write-Output "✓ Layout $layout exists"
    } else {
        Write-Output "✗ Layout $layout missing"
    }
}
```

#### Test 5: API Response Format Testing
```bash
Write-Output "=== API Response Format Testing ==="

# Panggil API (otomatis parse JSON ke object)
try {
    $response = Invoke-RestMethod -Uri "http://localhost:8000/api/posts?per_page=1" -Method Get
    } catch {
        Write-Output "✗ Failed to fetch API response"
        exit
    }

# Cek status field
if ($null -ne $response.status) {
    Write-Output "✓ API returns valid JSON with status field"
} else {
    Write-Output "✗ API response missing status field"
}

# Cek data field
if ($null -ne $response.data) {
    Write-Output "✓ API returns data field"
} else {
    Write-Output "✗ API missing data field"
}

# Cek pagination dalam data
if ($null -ne $response.data.pagination) {
    Write-Output "✓ API returns pagination data"
} else {
    Write-Output "✗ API missing pagination data"
}
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Controller Method Not Found
**Gejala:** `Method [methodName] does not exist` error
**Solusi:**
```bash
# Check controller namespace and class name
php artisan route:list | grep posts

# Verify controller file exists and has correct namespace
head -10 app/Http/Controllers/PostController.php

# Clear route cache
php artisan route:clear
php artisan config:clear

# Regenerate autoload files
composer dump-autoload
```

#### Problem 2: Mass Assignment Error
**Gejala:** `Add [fieldname] to fillable property to allow mass assignment`
**Solusi:**
```bash
# Check model fillable attributes
php artisan tinker --execute="
\$post = new \App\Models\Post();
var_dump(\$post->getFillable());
"

# Add missing fields to Model fillable array
nano app/Models/Post.php
# Update $fillable array dengan field yang missing

# Test mass assignment
php artisan tinker --execute="
\$post = \App\Models\Post::create([
    'title' => 'Test',
    'content' => 'Test content', 
    'slug' => 'test',
    'status' => 'draft',
    'user_id' => 1,
    'category_id' => 1
]);
echo 'Mass assignment test: ' . (\$post->id ? 'OK' : 'Failed') . PHP_EOL;
"
```

#### Problem 3: Validation Rules Failing
**Gejala:** Form submission fails dengan validation errors
**Solusi:**
```bash
# Test validation rules dalam controller
php artisan tinker --execute="
\$request = new \Illuminate\Http\Request();
\$request->merge([
    'title' => 'Test Title',
    'content' => 'Test content minimum 10 characters',
    'status' => 'draft',
    'category_id' => 1,
    'user_id' => 1
]);

try {
    \$validated = \$request->validate([
        'title' => 'required|string|max:255',
        'content' => 'required|string|min:10',
        'status' => 'required|in:draft,published,archived',
        'category_id' => 'required|exists:categories,id',
        'user_id' => 'required|exists:users,id',
    ]);
    echo '✓ Validation rules working' . PHP_EOL;
} catch (\Exception \$e) {
    echo '✗ Validation failed: ' . \$e->getMessage() . PHP_EOL;
}
"

# Check if referenced tables have data
mysql -u laravel_user -p laravel_app -e "
SELECT 'Users' as table_name, COUNT(*) as count FROM users
UNION ALL
SELECT 'Categories', COUNT(*) FROM categories;"
```

#### Problem 4: File Upload Issues
**Gejala:** Featured image upload fails atau file tidak tersimpan
**Solusi:**
```bash
# Check storage directory permissions
ls -la storage/app/public/
sudo chown -R $USER:www-data storage/
chmod -R 775 storage/

# Create symbolic link untuk public storage
php artisan storage:link

# Verify storage permissions
sudo chmod -R 775 storage/app/public
sudo chown -R $USER:www-data storage/app/public

# Test storage configuration
php artisan tinker --execute="
echo 'Storage disk: ' . config('filesystems.default') . PHP_EOL;
echo 'Public path: ' . storage_path('app/public') . PHP_EOL;
echo 'Storage writable: ' . (is_writable(storage_path('app/public')) ? 'Yes' : 'No') . PHP_EOL;
"

# Create storage directories
mkdir -p storage/app/public/posts/featured-images
```

#### Problem 5: Route Model Binding Issues
**Gejala:** `No query results for model [Post]` error
**Solusi:**
```bash
# Check route parameter names match model
php artisan route:list | grep posts

# Verify model route key name (default: id)
php artisan tinker --execute="
\$post = \App\Models\Post::first();
if (\$post) {
    echo 'Route key name: ' . \$post->getRouteKeyName() . PHP_EOL;
    echo 'Route key value: ' . \$post->getRouteKey() . PHP_EOL;
} else {
    echo 'No posts found in database' . PHP_EOL;
}
"

# Test route model binding
curl -s "localhost:8000/posts/1" | head -20
```

#### Problem 6: Pagination Not Working
**Gejala:** Pagination links tidak muncul atau error
**Solusi:**
```bash
# Check pagination view exists
ls -la resources/views/vendor/pagination/ 2>/dev/null || echo "Using default pagination views"

# Publish pagination views for customization
php artisan vendor:publish --tag=laravel-pagination

# Test pagination query
php artisan tinker --execute="
\$posts = \App\Models\Post::paginate(10);
echo 'Current page: ' . \$posts->currentPage() . PHP_EOL;
echo 'Total items: ' . \$posts->total() . PHP_EOL;
echo 'Per page: ' . \$posts->perPage() . PHP_EOL;
echo 'Has pages: ' . (\$posts->hasPages() ? 'Yes' : 'No') . PHP_EOL;
"
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

#### ✅ Controller & Routes

- Screenshot php artisan route:list menunjukkan route posts, categories, tags.
- Potongan kode PostController & PostApiController (cukup method utama).

#### ✅ Views & Forms
- Screenshot halaman Posts Index dengan filter/search/pagination.
- Screenshot halaman Create Post Form (validasi terlihat).

#### ✅ CRUD Testing

- Screenshot hasil create categories (tampil di index).
- Screenshot hasil edit categories (judul berubah).
- Screenshot hasil delete categories (categories hilang dari index).

**Format Submission:**
1. Buat folder submission/week/
2. Masukkan semua screenshot dengan nama yang jelas
3. Buat file laporan dalam format Markdown
4. Commit dan push ke repository
5. Sertakan link commit terakhir
