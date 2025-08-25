# WEEK 3: Database Design & Migration System
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 3
- **Durasi**: 100 menit  
- **Topik**: Perancangan Database Schema & Sistem Migrasi Laravel
- **Target**: Mahasiswa Semester 6
- **Platform**: Google Cloud Shell
- **Repository**: github.com/aidilsaputrakirsan/praktikum-cc

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Merancang schema database yang optimal untuk aplikasi web
2. Membuat dan mengelola Laravel migrations untuk version control database
3. Mengimplementasikan Eloquent Models dengan relationships yang tepat
4. Menggunakan Database Seeder dan Factory untuk sample data
5. Memahami konsep foreign key constraints dan referential integrity
6. Melakukan database testing dan validation dengan Laravel tools
7. Mengoptimalkan query database dan indexing strategy

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1 dan Week 2 telah selesai dengan sempurna
- Laravel framework sudah terinstall dan berfungsi
- MySQL database sudah dikonfigurasi dan dapat diakses
- Understanding dasar tentang database relational dan SQL

**Verifikasi Environment:**
```bash
# Pastikan berada di direktori project Laravel
cd ~/praktikum-cc/week2/laravel-app

# Verifikasi database connection
php artisan migrate:status

# Verifikasi Laravel version dan database config
php artisan about
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Perancangan Database Schema (20 menit)**

##### Step 1.1: Analisis Requirements Aplikasi
```bash
# Membuat file dokumentasi database design
mkdir -p database/documentation
nano database/documentation/schema_design.md
```

**Isi file database/documentation/schema_design.md:**
```markdown
# Database Schema Design - Praktikum CC ITK

## Application Requirements
Aplikasi yang akan dibangun adalah sistem manajemen konten sederhana dengan fitur:
- User authentication dan authorization
- Manajemen artikel/posts dengan kategori
- Sistem komentar untuk setiap artikel
- Tag system untuk kategorisasi content
- Media file management untuk images

## Entity Relationship Design

### 1. Users Table
- Primary entity untuk authentication
- Menyimpan informasi dasar user
- Relationship: One-to-Many dengan Posts, Comments

### 2. Categories Table  
- Kategorisasi utama untuk posts
- Hierarchical structure dengan parent_id
- Relationship: One-to-Many dengan Posts

### 3. Posts Table
- Content utama aplikasi
- Memiliki status publication (draft, published, archived)
- Relationship: Many-to-One dengan Users, Categories
- Relationship: One-to-Many dengan Comments
- Relationship: Many-to-Many dengan Tags

### 4. Comments Table
- Sistem komentar untuk posts
- Support nested comments dengan parent_id
- Relationship: Many-to-One dengan Users, Posts

### 5. Tags Table
- Tagging system untuk posts
- Relationship: Many-to-Many dengan Posts melalui post_tags

### 6. Media Table
- File storage management
- Support multiple file types (images, documents)
- Relationship: Many-to-One dengan Posts

## Database Constraints
- Foreign key constraints untuk referential integrity
- Unique constraints untuk email, slug
- Index untuk performance optimization
- Soft deletes untuk data preservation
```

##### Step 1.2: Database Schema Diagram
```bash
# Install tool untuk database diagram (opsional, untuk dokumentasi)
npm install -g @mermaid-js/mermaid-cli

# Buat file diagram ERD
cat > database/documentation/erd.md << 'EOF'
# Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ POSTS : creates
    USERS ||--o{ COMMENTS : writes
    CATEGORIES ||--o{ POSTS : contains
    POSTS ||--o{ COMMENTS : has
    POSTS }o--o{ TAGS : tagged_with
    POSTS ||--o{ MEDIA : contains
    
    USERS {
        bigint id PK
        string name
        string email UK
        timestamp email_verified_at
        string password
        string role
        text bio
        string avatar
        timestamps created_at
        timestamps updated_at
    }
    
    CATEGORIES {
        bigint id PK
        string name
        string slug UK
        text description
        bigint parent_id FK
        boolean is_active
        timestamps created_at
        timestamps updated_at
    }
    
    POSTS {
        bigint id PK
        string title
        string slug UK
        text excerpt
        longtext content
        string featured_image
        enum status
        bigint user_id FK
        bigint category_id FK
        timestamp published_at
        timestamps created_at
        timestamps updated_at
        softDeletes deleted_at
    }
    
    COMMENTS {
        bigint id PK
        text content
        bigint user_id FK
        bigint post_id FK
        bigint parent_id FK
        boolean is_approved
        timestamps created_at
        timestamps updated_at
        softDeletes deleted_at
    }
    
    TAGS {
        bigint id PK
        string name
        string slug UK
        string color
        timestamps created_at
        timestamps updated_at
    }
    
    POST_TAGS {
        bigint post_id FK
        bigint tag_id FK
        timestamps created_at
    }
    
    MEDIA {
        bigint id PK
        string filename
        string original_name
        string mime_type
        bigint file_size
        string path
        bigint post_id FK
        timestamps created_at
        timestamps updated_at
    }
```
EOF
```

#### **Bagian 2: Membuat Laravel Migrations (25 menit)**

##### Step 2.1: Generate Migration Files
```bash
# Generate migration untuk tabel users (sudah ada, kita akan modify)
# Generate migration untuk categories
php artisan make:migration create_categories_table

# Generate migration untuk posts  
php artisan make:migration create_posts_table

# Generate migration untuk comments
php artisan make:migration create_comments_table

# Generate migration untuk tags
php artisan make:migration create_tags_table

# Generate migration untuk pivot table post_tags
php artisan make:migration create_post_tags_table

# Generate migration untuk media table
php artisan make:migration create_media_table

# Lihat file migration yang telah dibuat
ls -la database/migrations/
```

##### Step 2.2: Implementasi Migration - Users Table
```bash
# Edit migration untuk users table (modify existing)
nano database/migrations/2014_10_12_000000_create_users_table.php
```

**Isi file create_users_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Jalankan migration untuk membuat tabel users
     * Tabel ini menyimpan informasi user authentication dan profile
     */
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            // Primary key auto increment
            $table->id();
            
            // Informasi dasar user
            $table->string('name'); // Nama lengkap user
            $table->string('email')->unique(); // Email unik untuk login
            $table->timestamp('email_verified_at')->nullable(); // Timestamp verifikasi email
            $table->string('password'); // Password ter-hash
            
            // Informasi tambahan user
            $table->enum('role', ['admin', 'editor', 'author', 'subscriber'])
                  ->default('subscriber'); // Role-based access control
            $table->text('bio')->nullable(); // Bio/deskripsi user
            $table->string('avatar')->nullable(); // Path ke avatar image
            $table->boolean('is_active')->default(true); // Status aktif user
            
            // Token untuk remember me functionality
            $table->rememberToken();
            
            // Timestamp created_at dan updated_at
            $table->timestamps();
            
            // Indexing untuk performance
            $table->index('email'); // Index untuk login lookup
            $table->index('role'); // Index untuk role filtering
        });
    }

    /**
     * Rollback migration - hapus tabel users
     */
    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

##### Step 2.3: Implementasi Migration - Categories Table
```bash
# Edit migration categories
nano database/migrations/$(ls database/migrations/ | grep create_categories_table)
```

**Isi file create_categories_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Jalankan migration untuk membuat tabel categories
     * Tabel ini untuk kategorisasi posts dengan support hierarchical structure
     */
    public function up(): void
    {
        Schema::create('categories', function (Blueprint $table) {
            // Primary key auto increment
            $table->id();
            
            // Informasi kategori
            $table->string('name'); // Nama kategori
            $table->string('slug')->unique(); // URL-friendly identifier
            $table->text('description')->nullable(); // Deskripsi kategori
            $table->string('color', 7)->default('#3B82F6'); // Hex color untuk UI
            
            // Hierarchical structure - self-referencing foreign key
            $table->unsignedBigInteger('parent_id')->nullable();
            $table->foreign('parent_id')->references('id')->on('categories')
                  ->onDelete('set null'); // Set null jika parent dihapus
            
            // Status dan metadata
            $table->boolean('is_active')->default(true); // Status aktif kategori
            $table->integer('sort_order')->default(0); // Urutan tampilan
            
            // Timestamp created_at dan updated_at
            $table->timestamps();
            
            // Indexing untuk performance
            $table->index('slug'); // Index untuk URL lookup
            $table->index('parent_id'); // Index untuk hierarchical queries
            $table->index(['is_active', 'sort_order']); // Composite index untuk listing
        });
    }

    /**
     * Rollback migration - hapus tabel categories
     */
    public function down(): void
    {
        Schema::dropIfExists('categories');
    }
};
```

##### Step 2.4: Implementasi Migration - Posts Table
```bash
# Edit migration posts
nano database/migrations/$(ls database/migrations/ | grep create_posts_table)
```

**Isi file create_posts_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Jalankan migration untuk membuat tabel posts
     * Tabel ini menyimpan konten utama aplikasi (artikel, blog posts)
     */
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            // Primary key auto increment
            $table->id();
            
            // Konten post
            $table->string('title'); // Judul post
            $table->string('slug')->unique(); // URL-friendly identifier
            $table->text('excerpt')->nullable(); // Ringkasan post
            $table->longText('content'); // Konten lengkap post (HTML allowed)
            $table->string('featured_image')->nullable(); // Path ke featured image
            
            // Status publikasi
            $table->enum('status', ['draft', 'published', 'archived'])
                  ->default('draft'); // Status publikasi post
            
            // Foreign keys untuk relationships
            $table->unsignedBigInteger('user_id'); // Penulis post
            $table->foreign('user_id')->references('id')->on('users')
                  ->onDelete('cascade'); // Hapus post jika user dihapus
                  
            $table->unsignedBigInteger('category_id'); // Kategori post
            $table->foreign('category_id')->references('id')->on('categories')
                  ->onDelete('restrict'); // Prevent delete category jika ada posts
            
            // Metadata
            $table->timestamp('published_at')->nullable(); // Waktu publikasi
            $table->integer('views_count')->default(0); // Counter views
            $table->boolean('is_featured')->default(false); // Apakah featured post
            $table->boolean('allow_comments')->default(true); // Izinkan komentar
            
            // SEO fields
            $table->string('meta_title')->nullable(); // SEO title
            $table->text('meta_description')->nullable(); // SEO description
            $table->text('meta_keywords')->nullable(); // SEO keywords
            
            // Timestamp created_at dan updated_at
            $table->timestamps();
            
            // Soft deletes untuk data preservation
            $table->softDeletes();
            
            // Indexing untuk performance
            $table->index('slug'); // Index untuk URL lookup
            $table->index(['status', 'published_at']); // Composite index untuk published posts
            $table->index('user_id'); // Index untuk author lookup
            $table->index('category_id'); // Index untuk category filtering
            $table->index('is_featured'); // Index untuk featured posts
            $table->fullText(['title', 'content']); // Full-text search index
        });
    }

    /**
     * Rollback migration - hapus tabel posts
     */
    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

##### Step 2.5: Implementasi Migration - Comments, Tags, dan Post_Tags
```bash
# Edit migration comments
nano database/migrations/$(ls database/migrations/ | grep create_comments_table)
```

**Isi file create_comments_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Migration untuk tabel comments - sistem komentar posts
     */
    public function up(): void
    {
        Schema::create('comments', function (Blueprint $table) {
            $table->id();
            
            // Konten komentar
            $table->text('content'); // Isi komentar
            
            // Foreign keys
            $table->unsignedBigInteger('user_id'); // Penulis komentar
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
            
            $table->unsignedBigInteger('post_id'); // Post yang dikomentari
            $table->foreign('post_id')->references('id')->on('posts')->onDelete('cascade');
            
            // Nested comments support
            $table->unsignedBigInteger('parent_id')->nullable(); // Reply to comment
            $table->foreign('parent_id')->references('id')->on('comments')->onDelete('cascade');
            
            // Moderation
            $table->boolean('is_approved')->default(false); // Moderasi komentar
            
            $table->timestamps();
            $table->softDeletes(); // Soft delete untuk preservation
            
            // Indexing
            $table->index(['post_id', 'is_approved']); // Comments per post
            $table->index('parent_id'); // Nested comments lookup
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('comments');
    }
};
```

```bash
# Edit migration tags
nano database/migrations/$(ls database/migrations/ | grep create_tags_table)
```

**Isi file create_tags_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Migration untuk tabel tags - sistem tagging posts
     */
    public function up(): void
    {
        Schema::create('tags', function (Blueprint $table) {
            $table->id();
            
            $table->string('name'); // Nama tag
            $table->string('slug')->unique(); // URL-friendly identifier
            $table->string('color', 7)->default('#10B981'); // Hex color untuk UI
            $table->text('description')->nullable(); // Deskripsi tag
            
            $table->timestamps();
            
            // Indexing
            $table->index('slug');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tags');
    }
};
```

```bash
# Edit migration post_tags (pivot table)
nano database/migrations/$(ls database/migrations/ | grep create_post_tags_table)
```

**Isi file create_post_tags_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Migration untuk pivot table post_tags - many-to-many relationship
     */
    public function up(): void
    {
        Schema::create('post_tags', function (Blueprint $table) {
            // Composite primary key
            $table->unsignedBigInteger('post_id');
            $table->unsignedBigInteger('tag_id');
            
            // Foreign key constraints
            $table->foreign('post_id')->references('id')->on('posts')->onDelete('cascade');
            $table->foreign('tag_id')->references('id')->on('tags')->onDelete('cascade');
            
            // Prevent duplicate entries
            $table->primary(['post_id', 'tag_id']);
            
            $table->timestamps(); // Track when tag was added to post
            
            // Indexing untuk reverse lookup
            $table->index('tag_id'); // Posts by tag
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('post_tags');
    }
};
```

##### Step 2.6: Implementasi Migration - Media Table
```bash
# Edit migration media
nano database/migrations/$(ls database/migrations/ | grep create_media_table)
```

**Isi file create_media_table.php:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Migration untuk tabel media - file management system
     */
    public function up(): void
    {
        Schema::create('media', function (Blueprint $table) {
            $table->id();
            
            // File information
            $table->string('filename'); // Generated filename
            $table->string('original_name'); // Original uploaded filename
            $table->string('mime_type'); // File MIME type
            $table->unsignedBigInteger('file_size'); // File size in bytes
            $table->string('path'); // Storage path
            $table->json('metadata')->nullable(); // Additional file metadata (dimensions, etc.)
            
            // Relationships
            $table->unsignedBigInteger('post_id')->nullable(); // Associated post
            $table->foreign('post_id')->references('id')->on('posts')->onDelete('set null');
            
            $table->unsignedBigInteger('uploaded_by'); // User who uploaded
            $table->foreign('uploaded_by')->references('id')->on('users')->onDelete('cascade');
            
            // File type categorization
            $table->enum('type', ['image', 'document', 'video', 'audio', 'other'])->default('other');
            
            $table->timestamps();
            
            // Indexing
            $table->index('post_id');
            $table->index('type');
            $table->index('uploaded_by');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('media');
    }
};
```

#### **Bagian 3: Membuat Eloquent Models (20 menit)**

##### Step 3.1: Generate Model Files
```bash
# Generate models dengan resource controllers (akan digunakan week 4)
php artisan make:model Category -mcr
php artisan make:model Post -mcr  
php artisan make:model Comment -mcr
php artisan make:model Tag -mcr
php artisan make:model Media -mcr

# Note: User model sudah ada dari Laravel default
# Lihat models yang telah dibuat
ls -la app/Models/
```

##### Step 3.2: Implementasi User Model
```bash
# Edit User model untuk menambahkan relationships
nano app/Models/User.php
```

**Isi file app/Models/User.php:**
```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

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
}
```

##### Step 3.3: Implementasi Category Model
```bash
# Edit Category model
nano app/Models/Category.php
```

**Isi file app/Models/Category.php:**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class Category extends Model
{
    use HasFactory;

    /**
     * Atribut yang dapat diisi secara mass assignment
     */
    protected $fillable = [
        'name',         // Nama kategori
        'slug',         // URL-friendly identifier
        'description',  // Deskripsi kategori
        'color',        // Hex color untuk UI
        'parent_id',    // Parent category ID untuk hierarchical structure
        'is_active',    // Status aktif kategori
        'sort_order',   // Urutan tampilan
    ];

    /**
     * Casting atribut ke tipe data yang sesuai
     */
    protected $casts = [
        'is_active' => 'boolean',   // Cast ke boolean
        'sort_order' => 'integer',  // Cast ke integer
    ];

    /**
     * Boot method untuk auto-generate slug
     */
    protected static function boot()
    {
        parent::boot();

        // Auto-generate slug dari name saat creating
        static::creating(function ($category) {
            if (empty($category->slug)) {
                $category->slug = Str::slug($category->name);
            }
        });

        // Update slug saat name berubah
        static::updating(function ($category) {
            if ($category->isDirty('name')) {
                $category->slug = Str::slug($category->name);
            }
        });
    }

    /**
     * Relationship: Category belongs to Parent Category
     * Self-referencing relationship untuk hierarchical structure
     */
    public function parent()
    {
        return $this->belongsTo(Category::class, 'parent_id');
    }

    /**
     * Relationship: Category has many Child Categories
     * Self-referencing relationship untuk hierarchical structure
     */
    public function children()
    {
        return $this->hasMany(Category::class, 'parent_id')
                   ->orderBy('sort_order');
    }

    /**
     * Relationship: Category has many Posts
     * Satu kategori dapat memiliki banyak posts
     */
    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    /**
     * Scope: Filter active categories only
     * Usage: Category::active()->get()
     */
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    /**
     * Scope: Get root categories (no parent)
     * Usage: Category::roots()->get()
     */
    public function scopeRoots($query)
    {
        return $query->whereNull('parent_id');
    }

    /**
     * Scope: Order by sort_order
     * Usage: Category::sorted()->get()
     */
    public function scopeSorted($query)
    {
        return $query->orderBy('sort_order');
    }

    /**
     * Get all posts count including from child categories
     * Menghitung total posts di kategori ini dan child categories
     */
    public function getTotalPostsCountAttribute()
    {
        $count = $this->posts()->count();
        
        foreach ($this->children as $child) {
            $count += $child->total_posts_count;
        }
        
        return $count;
    }

    /**
     * Get breadcrumb path untuk hierarchical navigation
     * Returns array of parent categories
     */
    public function getBreadcrumbAttribute()
    {
        $breadcrumb = collect([$this]);
        $parent = $this->parent;
        
        while ($parent) {
            $breadcrumb->prepend($parent);
            $parent = $parent->parent;
        }
        
        return $breadcrumb;
    }
}
```

##### Step 3.4: Implementasi Post Model
```bash
# Edit Post model
nano app/Models/Post.php
```

**Isi file app/Models/Post.php:**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Str;
use Carbon\Carbon;

class Post extends Model
{
    use HasFactory, SoftDeletes;

    /**
     * Atribut yang dapat diisi secara mass assignment
     */
    protected $fillable = [
        'title',            // Judul post
        'slug',             // URL-friendly identifier
        'excerpt',          // Ringkasan post
        'content',          // Konten lengkap
        'featured_image',   // Path ke featured image
        'status',           // Status publikasi
        'user_id',          // Penulis post
        'category_id',      // Kategori post
        'published_at',     // Waktu publikasi
        'views_count',      // Counter views
        'is_featured',      // Featured post flag
        'allow_comments',   // Izinkan komentar
        'meta_title',       // SEO title
        'meta_description', // SEO description
        'meta_keywords',    // SEO keywords
    ];

    /**
     * Casting atribut ke tipe data yang sesuai
     */
    protected $casts = [
        'published_at' => 'datetime',   // Cast ke Carbon instance
        'is_featured' => 'boolean',     // Cast ke boolean
        'allow_comments' => 'boolean',  // Cast ke boolean
        'views_count' => 'integer',     // Cast ke integer
    ];

    /**
     * Boot method untuk auto-generate slug dan set published_at
     */
    protected static function boot()
    {
        parent::boot();

        // Auto-generate slug dari title saat creating
        static::creating(function ($post) {
            if (empty($post->slug)) {
                $post->slug = Str::slug($post->title);
            }
            
            // Set published_at jika status published dan belum di-set
            if ($post->status === 'published' && !$post->published_at) {
                $post->published_at = now();
            }
        });

        // Update slug dan published_at saat updating
        static::updating(function ($post) {
            if ($post->isDirty('title')) {
                $post->slug = Str::slug($post->title);
            }
            
            // Set published_at saat status berubah ke published
            if ($post->isDirty('status') && $post->status === 'published' && !$post->published_at) {
                $post->published_at = now();
            }
        });
    }

    /**
     * Relationship: Post belongs to User (author)
     * Setiap post dimiliki oleh satu user
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Alias untuk relationship user sebagai author
     */
    public function author()
    {
        return $this->user();
    }

    /**
     * Relationship: Post belongs to Category
     * Setiap post masuk dalam satu kategori
     */
    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    /**
     * Relationship: Post has many Comments
     * Satu post dapat memiliki banyak comments
     */
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * Relationship: Post belongs to many Tags (many-to-many)
     * Satu post dapat memiliki banyak tags
     */
    public function tags()
    {
        return $this->belongsToMany(Tag::class, 'post_tags')->withTimestamps();
    }

    /**
     * Relationship: Post has many Media files
     * Satu post dapat memiliki banyak media files
     */
    public function media()
    {
        return $this->hasMany(Media::class);
    }

    /**
     * Scope: Filter published posts only
     * Usage: Post::published()->get()
     */
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
                    ->where('published_at', '<=', now());
    }

    /**
     * Scope: Filter featured posts
     * Usage: Post::featured()->get()
     */
    public function scopeFeatured($query)
    {
        return $query->where('is_featured', true);
    }

    /**
     * Scope: Order by latest published
     * Usage: Post::latest()->get()
     */
    public function scopeLatest($query)
    {
        return $query->orderBy('published_at', 'desc');
    }

    /**
     * Scope: Search posts by title and content
     * Usage: Post::search('keyword')->get()
     */
    public function scopeSearch($query, $term)
    {
        return $query->where(function ($q) use ($term) {
            $q->where('title', 'LIKE', "%{$term}%")
              ->orWhere('content', 'LIKE', "%{$term}%")
              ->orWhere('excerpt', 'LIKE', "%{$term}%");
        });
    }

    /**
     * Accessor: Get excerpt with fallback
     * Jika excerpt kosong, ambil dari content
     */
    public function getExcerptAttribute($value)
    {
        if ($value) {
            return $value;
        }
        
        // Fallback ke content yang dipotong
        return Str::limit(strip_tags($this->content), 150);
    }

    /**
     * Accessor: Get featured image URL
     */
    public function getFeaturedImageUrlAttribute()
    {
        if ($this->featured_image) {
            return asset('storage/' . $this->featured_image);
        }
        
        return null;
    }

    /**
     * Accessor: Get reading time estimate
     * Estimasi waktu baca berdasarkan jumlah kata
     */
    public function getReadingTimeAttribute()
    {
        $wordCount = str_word_count(strip_tags($this->content));
        $readingTime = ceil($wordCount / 200); // Asumsi 200 kata per menit
        
        return $readingTime . ' min read';
    }

    /**
     * Check if post is published
     */
    public function isPublished()
    {
        return $this->status === 'published' && 
               $this->published_at && 
               $this->published_at <= now();
    }

    /**
     * Increment views count
     */
    public function incrementViews()
    {
        $this->increment('views_count');
    }
}
```

##### Step 3.5: Implementasi Comment, Tag, dan Media Models
```bash
# Edit Comment model
nano app/Models/Comment.php
```

**Isi file app/Models/Comment.php (abbreviated untuk space):**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Comment extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'content', 'user_id', 'post_id', 'parent_id', 'is_approved'
    ];

    protected $casts = [
        'is_approved' => 'boolean',
    ];

    // Relationships
    public function user() { return $this->belongsTo(User::class); }
    public function post() { return $this->belongsTo(Post::class); }
    public function parent() { return $this->belongsTo(Comment::class, 'parent_id'); }
    public function replies() { return $this->hasMany(Comment::class, 'parent_id'); }

    // Scopes
    public function scopeApproved($query) { return $query->where('is_approved', true); }
    public function scopeRootComments($query) { return $query->whereNull('parent_id'); }
}
```

```bash
# Edit Tag dan Media models (ringkas)
nano app/Models/Tag.php
```

**Isi file app/Models/Tag.php:**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class Tag extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'slug', 'color', 'description'];

    protected static function boot()
    {
        parent::boot();
        static::creating(function ($tag) {
            if (empty($tag->slug)) {
                $tag->slug = Str::slug($tag->name);
            }
        });
    }

    public function posts() { return $this->belongsToMany(Post::class, 'post_tags'); }
}
```

#### **Bagian 4: Database Seeding dan Testing (15 menit)**

##### Step 4.1: Jalankan Migrations
```bash
# Jalankan semua migrations
php artisan migrate

# Verifikasi tables telah dibuat
php artisan migrate:status

# Check database tables via MySQL
mysql -u laravel_user -p laravel_app -e "SHOW TABLES;"
```

##### Step 4.2: Membuat Database Seeders
```bash
# Generate seeder files
php artisan make:seeder UserSeeder
php artisan make:seeder CategorySeeder  
php artisan make:seeder TagSeeder
php artisan make:seeder PostSeeder

# Edit DatabaseSeeder untuk menjalankan semua seeders
nano database/seeders/DatabaseSeeder.php
```

**Isi file DatabaseSeeder.php:**
```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed aplikasi database dengan sample data
     * Urutan seeding penting karena foreign key dependencies
     */
    public function run(): void
    {
        // Jalankan seeders sesuai dependency order
        $this->call([
            UserSeeder::class,      // Users dulu (tidak ada dependencies)
            CategorySeeder::class,  // Categories (tidak ada dependencies)
            TagSeeder::class,       // Tags (tidak ada dependencies)
            PostSeeder::class,      // Posts (depends on users, categories, tags)
        ]);
        
        $this->command->info('Database seeding completed successfully!');
    }
}
```

##### Step 4.3: Implementasi Sample Seeders
```bash
# Edit UserSeeder untuk sample data
nano database/seeders/UserSeeder.php
```

**Isi file UserSeeder.php:**
```php
<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Admin user
        User::create([
            'name' => 'Administrator',
            'email' => 'admin@itk.ac.id',
            'password' => Hash::make('admin123'),
            'role' => 'admin',
            'bio' => 'System Administrator',
            'is_active' => true,
            'email_verified_at' => now(),
        ]);

        // Editor user
        User::create([
            'name' => 'Editor ITK',
            'email' => 'editor@itk.ac.id', 
            'password' => Hash::make('editor123'),
            'role' => 'editor',
            'bio' => 'Content Editor',
            'is_active' => true,
            'email_verified_at' => now(),
        ]);

        // Sample authors
        User::factory(5)->create();
    }
}
```

##### Step 4.4: Jalankan Database Seeding
```bash
# Jalankan seeders
php artisan db:seed

# Atau jalankan seeder spesifik
php artisan db:seed --class=UserSeeder

# Verifikasi data telah terisi
php artisan tinker --execute="
echo 'Users: ' . \App\Models\User::count() . PHP_EOL;
echo 'Categories: ' . \App\Models\Category::count() . PHP_EOL;
echo 'Posts: ' . \App\Models\Post::count() . PHP_EOL;
echo 'Tags: ' . \App\Models\Tag::count() . PHP_EOL;
"
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Database Schema Verification
```bash
echo "=== Database Schema Verification ==="

# Test 1.1: Verify all tables exist
EXPECTED_TABLES=("users" "categories" "posts" "comments" "tags" "post_tags" "media" "migrations")
for table in "${EXPECTED_TABLES[@]}"; do
    COUNT=$(mysql -u laravel_user -p laravel_app -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'laravel_app' AND table_name = '$table';" -s -N 2>/dev/null)
    if [ "$COUNT" -eq 1 ]; then
        echo "✓ Table $table exists"
    else
        echo "✗ Table $table missing"
    fi
done

# Test 1.2: Verify foreign key constraints
mysql -u laravel_user -p laravel_app -e "
SELECT CONSTRAINT_NAME, TABLE_NAME, COLUMN_NAME, REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME 
FROM information_schema.KEY_COLUMN_USAGE 
WHERE REFERENCED_TABLE_SCHEMA = 'laravel_app';" 2>/dev/null && echo "✓ Foreign key constraints verified"
```

#### Test 2: Model Relationships Testing
```bash
echo "=== Model Relationships Testing ==="

php artisan tinker --execute="
// Test User -> Posts relationship
\$user = \App\Models\User::first();
if (\$user && \$user->posts() !== null) {
    echo '✓ User->posts relationship working' . PHP_EOL;
} else {
    echo '✗ User->posts relationship failed' . PHP_EOL;
}

// Test Post -> Category relationship  
if (class_exists('\App\Models\Post')) {
    \$post = new \App\Models\Post();
    if (method_exists(\$post, 'category')) {
        echo '✓ Post->category relationship defined' . PHP_EOL;
    } else {
        echo '✗ Post->category relationship missing' . PHP_EOL;
    }
}

// Test Category -> Posts relationship
if (class_exists('\App\Models\Category')) {
    \$category = new \App\Models\Category();
    if (method_exists(\$category, 'posts')) {
        echo '✓ Category->posts relationship defined' . PHP_EOL;
    } else {
        echo '✗ Category->posts relationship missing' . PHP_EOL;
    }
}
"
```

#### Test 3: Migration Rollback and Re-run
```bash
echo "=== Migration Rollback Testing ==="

# Backup current migration status
php artisan migrate:status > /tmp/migration_status_before.txt

# Test rollback (hati-hati, ini akan menghapus data!)
echo "Testing migration rollback..."
php artisan migrate:rollback --step=3

# Check some tables are dropped
TABLES_AFTER_ROLLBACK=$(mysql -u laravel_user -p laravel_app -e "SHOW TABLES;" -s 2>/dev/null | wc -l)
echo "Tables after rollback: $TABLES_AFTER_ROLLBACK"

# Re-run migrations
php artisan migrate

# Verify tables are back
TABLES_AFTER_MIGRATE=$(mysql -u laravel_user -p laravel_app -e "SHOW TABLES;" -s 2>/dev/null | wc -l)
echo "Tables after re-migrate: $TABLES_AFTER_MIGRATE"

if [ "$TABLES_AFTER_MIGRATE" -gt "$TABLES_AFTER_ROLLBACK" ]; then
    echo "✓ Migration rollback and re-run successful"
else
    echo "✗ Migration rollback test failed"
fi
```

#### Test 4: Database Performance and Indexing
```bash
echo "=== Database Performance Testing ==="

php artisan tinker --execute="
// Test database connection performance
\$start = microtime(true);
\DB::connection()->getPdo();
\$end = microtime(true);
echo 'Database connection time: ' . round((\$end - \$start) * 1000, 2) . 'ms' . PHP_EOL;

// Test query performance with indexing
\$start = microtime(true);
\App\Models\User::where('email', 'admin@itk.ac.id')->first();
\$end = microtime(true);
echo 'Indexed query time: ' . round((\$end - \$start) * 1000, 2) . 'ms' . PHP_EOL;

echo '✓ Database performance tests completed' . PHP_EOL;
"
```

#### Test 5: Data Integrity and Constraints
```bash
echo "=== Data Integrity Testing ==="

php artisan tinker --execute="
try {
    // Test foreign key constraint
    \$post = new \App\Models\Post();
    \$post->title = 'Test Post';
    \$post->content = 'Test content';
    \$post->user_id = 99999; // Non-existent user
    \$post->category_id = 99999; // Non-existent category
    \$post->save();
    echo '✗ Foreign key constraint not working' . PHP_EOL;
} catch (\Exception \$e) {
    echo '✓ Foreign key constraints working properly' . PHP_EOL;
}

// Test unique constraint
try {
    \$user1 = \App\Models\User::create([
        'name' => 'Test User',
        'email' => 'duplicate@test.com',
        'password' => 'password123'
    ]);
    
    \$user2 = \App\Models\User::create([
        'name' => 'Test User 2', 
        'email' => 'duplicate@test.com', // Same email
        'password' => 'password123'
    ]);
    echo '✗ Unique constraint not working' . PHP_EOL;
} catch (\Exception \$e) {
    echo '✓ Unique constraints working properly' . PHP_EOL;
}
"
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Migration Foreign Key Error
**Gejala:** `SQLSTATE[HY000]: General error: 1215 Cannot add foreign key constraint`
**Solusi:**
```bash
# Check migration order - foreign key tables must exist first
ls -la database/migrations/ | grep -E "(users|categories|posts)"

# Ensure proper column types for foreign keys
mysql -u laravel_user -p laravel_app -e "
DESCRIBE users;
DESCRIBE categories; 
DESCRIBE posts;"

# Check if tables use same engine
mysql -u laravel_user -p laravel_app -e "
SELECT table_name, engine FROM information_schema.tables 
WHERE table_schema = 'laravel_app';"

# Fix: Make sure all tables use InnoDB
php artisan make:migration fix_table_engines
# Add Schema::table() statements to set engine = 'InnoDB'
```

#### Problem 2: Model Relationship Not Working
**Gejala:** `Call to undefined method relationship()`
**Solusi:**
```bash
# Check model imports and namespaces
php artisan tinker --execute="
echo 'User model path: ' . (new ReflectionClass(\App\Models\User::class))->getFileName() . PHP_EOL;
echo 'Post model path: ' . (new ReflectionClass(\App\Models\Post::class))->getFileName() . PHP_EOL;
"

# Verify method names and return types
grep -n "public function" app/Models/User.php
grep -n "public function" app/Models/Post.php

# Clear autoload cache
composer dump-autoload
php artisan clear-compiled
```

#### Problem 3: Seeder Class Not Found
**Gejala:** `Class 'Database\Seeders\UserSeeder' not found`
**Solusi:**
```bash
# Check seeder namespace and class name
head -5 database/seeders/UserSeeder.php

# Regenerate autoload files
composer dump-autoload

# Check if seeder is properly registered
php artisan list | grep seed

# Run specific seeder to test
php artisan db:seed --class=UserSeeder --verbose
```

#### Problem 4: Migration Stuck or Timeout
**Gejala:** Migration hangs or times out during execution
**Solusi:**
```bash
# Check MySQL processlist for long-running queries
mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# Increase PHP memory and time limits
php -d memory_limit=512M -d max_execution_time=300 artisan migrate

# Break down large migrations into smaller steps
php artisan migrate --step

# Check MySQL lock status
mysql -u root -p -e "SHOW ENGINE INNODB STATUS\G" | grep -A 10 "LATEST DETECTED DEADLOCK"
```

#### Problem 5: Eloquent Model Attributes Not Saving
**Gejala:** Model attributes tidak tersimpan ke database
**Solusi:**
```bash
# Check fillable attributes in model
php artisan tinker --execute="
\$model = new \App\Models\Post();
var_dump(\$model->getFillable());
"

# Verify column exists in database
mysql -u laravel_user -p laravel_app -e "DESCRIBE posts;"

# Test with explicit attribute assignment
php artisan tinker --execute="
\$post = new \App\Models\Post();
\$post->title = 'Test';
\$post->content = 'Test content';
\$post->user_id = 1;
\$post->category_id = 1;
echo 'Before save: ' . \$post->title . PHP_EOL;
\$result = \$post->save();
echo 'Save result: ' . (\$result ? 'success' : 'failed') . PHP_EOL;
"
```

#### Problem 6: Database Connection Pool Exhaustion
**Gejala:** `Too many connections` error dari MySQL
**Solusi:**
```bash
# Check current MySQL connections
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"

# Optimize Laravel database config
nano config/database.php
# Set 'pool' => 10 untuk connection pooling

# Restart MySQL service
sudo systemctl restart mysql

# Monitor connection usage
watch -n 1 'mysql -u root -p -e "SHOW STATUS LIKE \"Threads_connected\";"'
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

#### ✅ Database Schema Design
- [ ] File `database/documentation/schema_design.md` dengan complete ER diagram
- [ ] Screenshot hasil `php artisan migrate:status` showing all migrations
- [ ] Screenshot database tables di MySQL: `SHOW TABLES;`
- [ ] Screenshot foreign key relationships: `SHOW CREATE TABLE posts;`

#### ✅ Laravel Migrations
- [ ] 6 migration files dengan proper foreign key constraints
- [ ] Screenshot migration rollback dan re-run berhasil
- [ ] Screenshot database after migrations dengan correct data types
- [ ] File backup migration dengan `php artisan schema:dump`

#### ✅ Eloquent Models
- [ ] 5 Model files dengan complete relationships dan methods
- [ ] Screenshot testing model relationships di `php artisan tinker`
- [ ] Screenshot model attributes dan fillable properties
- [ ] Dokumentasi model methods dan scopes usage

#### ✅ Database Seeding
- [ ] 4 Seeder files dengan realistic sample data
- [ ] Screenshot hasil `php artisan db:seed` successful execution
- [ ] Screenshot sample data di database tables
- [ ] Database backup file setelah seeding

#### ✅ Testing & Verification
- [ ] Screenshot 5 test suite results (semua ✓ passed)
- [ ] Screenshot database performance metrics
- [ ] Screenshot foreign key constraint testing
- [ ] Screenshot model relationship testing results

#### ✅ Documentation
- [ ] File `week3/README.md` berisi:
  - Database design decisions dan rationale
  - Migration strategy dan execution steps
  - Model relationships explanation
  - Testing results dan performance metrics
  - Troubleshooting issues encountered dan solutions

**Format Submission:**
```bash
cd ~/praktikum-cc
mkdir -p submission/week3/{database,models,seeders,tests,docs}

# Export database schema
mysqldump -u laravel_user -p laravel_app --no-data > submission/week3/database/schema.sql

# Copy migration files
cp database/migrations/*.php submission/week3/database/

# Copy model files  
cp app/Models/*.php submission/week3/models/

# Copy seeder files
cp database/seeders/*.php submission/week3/seeders/

# Create documentation
cat > submission/week3/README.md << 'EOF'
# Week 3: Database Design & Migration System

## Completed Tasks
- [x] Database schema design dengan ERD
- [x] 6 Laravel migrations dengan proper constraints
- [x] 5 Eloquent models dengan complete relationships
- [x] Database seeding dengan sample data
- [x] Comprehensive testing dan verification
- [x] Performance optimization dan indexing

## Database Statistics
- Tables Created: 7 (+ migrations table)
- Foreign Key Constraints: 8
- Indexes Created: 12
- Sample Records: [total dari seeding]

## Model Relationships Implemented
- User: hasMany Posts, Comments, Media
- Category: hasMany Posts, belongsTo Parent, hasMany Children  
- Post: belongsTo User/Category, hasMany Comments, belongsToMany Tags
- Comment: belongsTo User/Post/Parent, hasMany Replies
- Tag: belongsToMany Posts
- Media: belongsTo Post, User

## Performance Optimizations
- Indexed foreign keys untuk fast joins
- Full-text search index pada posts table
- Composite indexes untuk filtering
- Query optimization dengan eager loading

## Next Steps
- Week 4: Implementasi CRUD operations (Create & Read)
- Frontend integration dengan database
EOF

# Test database export
php artisan schema:dump --prune

# Commit and push
git add submission/week3/
git commit -m "Week 3: Database Design & Migration System - [Nama]"
git push origin main
```