# WEEK 8: Production Deployment & Optimization
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 8
- **Durasi**: 100 menit  
- **Topik**: Production Deployment, Performance Optimization & Security Hardening
- **Target**: Mahasiswa Semester 6
- **Platform**: Google Cloud Shell → Google Cloud Platform
- **Repository**: github.com/aidilsaputrakirsan/praktikum-cc

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Mempersiapkan aplikasi Laravel untuk production deployment di Google Cloud
2. Mengimplementasikan database optimization dan caching strategies
3. Menerapkan security hardening dan monitoring untuk production environment
4. Mengkonfigurasi CI/CD pipeline dengan GitHub Actions untuk automated deployment
5. Mengoptimalkan performance aplikasi dengan lazy loading dan query optimization
6. Menerapkan backup strategies dan disaster recovery planning
7. Memahami cost optimization dan resource management di cloud environment

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1-7 telah completed dengan full application functionality
- Laravel application dengan authentication, authorization, dan CRUD operations
- Google Cloud account dengan billing enabled
- Understanding basic cloud concepts dan deployment strategies

**Environment Verification:**
```bash
# Pastikan berada di direktori project Laravel
cd ~/praktikum-cc/week2/laravel-app

# Verifikasi application status
php artisan about
php artisan route:list | wc -l
php artisan config:show app.env

# Check application structure dan size
du -sh . 
find . -name "*.php" | wc -l
find . -name "*.blade.php" | wc -l

# Verify database dengan sample data
php artisan tinker --execute="
echo 'Application Status:' . PHP_EOL;
echo '  Users: ' . \App\Models\User::count() . PHP_EOL;
echo '  Posts: ' . \App\Models\Post::count() . PHP_EOL;
echo '  Categories: ' . \App\Models\Category::count() . PHP_EOL;
echo '  API Tokens: ' . \DB::table('personal_access_tokens')->count() . PHP_EOL;
echo '  Database Size: ' . \DB::select('SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size FROM information_schema.tables WHERE table_schema = DATABASE()')[0]->size . ' MB' . PHP_EOL;
"

# Check for any errors atau missing configurations
php artisan config:cache
php artisan route:cache
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Production Environment Preparation (25 menit)**

##### Step 1.1: Environment Configuration Optimization
```bash
# Create production environment configuration
cp .env .env.production

# Edit production environment file
nano .env.production
```

**Update .env.production dengan production settings:**
```bash
# === Application Configuration ===
APP_NAME="Praktikum Cloud Computing ITK"
APP_ENV=production
APP_KEY=base64:your_production_key_here
APP_DEBUG=false
APP_URL=https://your-domain.com

# === Database Configuration (Cloud SQL) ===
DB_CONNECTION=mysql
DB_HOST=your-cloud-sql-ip
DB_PORT=3306
DB_DATABASE=laravel_production
DB_USERNAME=laravel_user
DB_PASSWORD=secure_production_password

# === Cache Configuration ===
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120

# === Redis Configuration ===
REDIS_HOST=your-redis-instance
REDIS_PASSWORD=redis_password
REDIS_PORT=6379

# === Mail Configuration (Production SMTP) ===
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your_production_email@gmail.com
MAIL_PASSWORD=your_app_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@your-domain.com
MAIL_FROM_NAME="${APP_NAME}"

# === File Storage Configuration ===
FILESYSTEM_DISK=gcs
GCS_PROJECT_ID=your-gcp-project-id
GCS_KEY_FILE=path/to/service-account.json
GCS_BUCKET=your-storage-bucket

# === Security Configuration ===
SANCTUM_STATEFUL_DOMAINS=your-domain.com,www.your-domain.com
SANCTUM_TOKEN_EXPIRATION=1440
SESSION_SECURE_COOKIE=true
SESSION_SAME_SITE=strict

# === Performance Configuration ===
OCTANE_SERVER=swoole
HORIZON_BALANCE_STRATEGY=auto
TELESCOPE_ENABLED=false

# === Monitoring Configuration ===
LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error

# === Third-party Services ===
GOOGLE_ANALYTICS_ID=G-XXXXXXXXXX
SENTRY_LARAVEL_DSN=https://your-sentry-dsn
```

##### Step 1.2: Application Optimization untuk Production
```bash
# Create optimization service provider
php artisan make:provider OptimizationServiceProvider

# Edit optimization provider
nano app/Providers/OptimizationServiceProvider.php
```

**Isi file app/Providers/OptimizationServiceProvider.php:**
```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\View;
use Illuminate\Database\Events\QueryExecuted;

class OptimizationServiceProvider extends ServiceProvider
{
    /**
     * Register services untuk optimization
     */
    public function register(): void
    {
        // Register performance monitoring jika dalam development
        if (app()->environment('local')) {
            $this->registerQueryMonitoring();
        }
        
        // Register production optimizations
        if (app()->environment('production')) {
            $this->registerProductionOptimizations();
        }
    }

    /**
     * Bootstrap services untuk optimization
     */
    public function boot(): void
    {
        // Setup view caching untuk production
        if (app()->environment('production')) {
            $this->setupViewOptimizations();
        }
        
        // Setup database query optimizations
        $this->setupDatabaseOptimizations();
        
        // Setup cache optimizations
        $this->setupCacheOptimizations();
    }

    /**
     * Register query monitoring untuk development
     */
    private function registerQueryMonitoring(): void
    {
        DB::listen(function (QueryExecuted $query) {
            if ($query->time > 100) { // Queries longer than 100ms
                \Log::warning('Slow Query Detected', [
                    'sql' => $query->sql,
                    'bindings' => $query->bindings,
                    'time' => $query->time . 'ms',
                    'connection' => $query->connectionName,
                ]);
            }
        });
    }

    /**
     * Register production optimizations
     */
    private function registerProductionOptimizations(): void
    {
        // Optimize configuration loading
        $this->app->configurationIsCached() ?: $this->app->make('config')->set([
            'app.debug' => false,
            'debugbar.enabled' => false,
            'telescope.enabled' => false,
        ]);
        
        // Setup production-specific services
        $this->app->singleton('production.metrics', function ($app) {
            return new \App\Services\ProductionMetricsService();
        });
    }

    /**
     * Setup view optimizations
     */
    private function setupViewOptimizations(): void
    {
        // Cache common view data
        View::composer(['layouts.app', 'layouts.dashboard'], function ($view) {
            $cacheKey = 'view.common.data';
            $commonData = Cache::remember($cacheKey, 3600, function () {
                return [
                    'app_name' => config('app.name'),
                    'app_version' => config('app.version', '1.0.0'),
                    'current_year' => date('Y'),
                ];
            });
            
            $view->with($commonData);
        });
        
        // Pre-compile frequently used views
        if (app()->runningInConsole()) {
            $this->precompileViews();
        }
    }

    /**
     * Setup database optimizations
     */
    private function setupDatabaseOptimizations(): void
    {
        // Set default string length for MySQL compatibility
        \Schema::defaultStringLength(191);
        
        // Configure connection pool untuk production
        if (app()->environment('production')) {
            config([
                'database.connections.mysql.options' => [
                    \PDO::ATTR_PERSISTENT => true,
                    \PDO::ATTR_TIMEOUT => 30,
                    \PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => false,
                ],
            ]);
        }
    }

    /**
     * Setup cache optimizations
     */
    private function setupCacheOptimizations(): void
    {
        // Configure Redis untuk session dan cache
        if (app()->environment('production')) {
            config([
                'cache.stores.redis.serializer' => 'igbinary',
                'cache.stores.redis.compress' => true,
                'session.encrypt' => true,
            ]);
        }
        
        // Setup cache tags untuk selective invalidation
        Cache::macro('taggedRemember', function ($tags, $key, $ttl, $callback) {
            return Cache::tags($tags)->remember($key, $ttl, $callback);
        });
    }

    /**
     * Pre-compile frequently used views
     */
    private function precompileViews(): void
    {
        $views = [
            'layouts.app',
            'layouts.dashboard', 
            'layouts.guest',
            'dashboard',
            'posts.index',
            'auth.login',
        ];
        
        foreach ($views as $view) {
            try {
                view($view)->render();
            } catch (\Exception $e) {
                // Skip if view doesn't exist or has errors
                continue;
            }
        }
    }
}
```

##### Step 1.3: Database Migration Optimization
```bash
# Create production database migration
php artisan make:migration optimize_database_for_production

# Edit migration untuk add indexes dan optimizations
nano database/migrations/$(ls database/migrations/ | grep optimize_database_for_production)
```

**Isi file migration optimize_database_for_production:**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run migrations untuk production optimization
     */
    public function up(): void
    {
        // Optimize users table
        Schema::table('users', function (Blueprint $table) {
            // Add composite index untuk login queries
            $table->index(['email', 'is_active'], 'users_email_active_idx');
            
            // Add index untuk role-based queries
            $table->index(['role', 'is_active'], 'users_role_active_idx');
            
            // Add index untuk created_at sorting
            $table->index('created_at', 'users_created_at_idx');
        });

        // Optimize posts table
        Schema::table('posts', function (Blueprint $table) {
            // Add composite index untuk published posts
            $table->index(['status', 'published_at'], 'posts_status_published_idx');
            
            // Add index untuk category filtering
            $table->index(['category_id', 'status'], 'posts_category_status_idx');
            
            // Add index untuk user posts
            $table->index(['user_id', 'status'], 'posts_user_status_idx');
            
            // Add index untuk featured posts
            $table->index(['is_featured', 'status'], 'posts_featured_status_idx');
            
            // Add index untuk views count (popular posts)
            $table->index('views_count', 'posts_views_count_idx');
        });

        // Optimize comments table
        Schema::table('comments', function (Blueprint $table) {
            // Add composite index untuk post comments
            $table->index(['post_id', 'is_approved'], 'comments_post_approved_idx');
            
            // Add index untuk user comments
            $table->index(['user_id', 'created_at'], 'comments_user_created_idx');
            
            // Add index untuk parent comments (nested)
            $table->index('parent_id', 'comments_parent_idx');
        });

        // Optimize personal_access_tokens table
        Schema::table('personal_access_tokens', function (Blueprint $table) {
            // Add index untuk token cleanup
            $table->index('expires_at', 'tokens_expires_at_idx');
            
            // Add index untuk user tokens
            $table->index(['tokenable_type', 'tokenable_id'], 'tokens_tokenable_idx');
            
            // Add index untuk last used
            $table->index('last_used_at', 'tokens_last_used_idx');
        });

        // Optimize categories table
        Schema::table('categories', function (Blueprint $table) {
            // Add index untuk slug lookups
            $table->index('slug', 'categories_slug_idx');
            
            // Add index untuk active categories
            $table->index(['is_active', 'sort_order'], 'categories_active_sort_idx');
            
            // Add index untuk hierarchical queries
            $table->index('parent_id', 'categories_parent_idx');
        });

        // Optimize tags table
        Schema::table('tags', function (Blueprint $table) {
            // Add index untuk slug lookups
            $table->index('slug', 'tags_slug_idx');
        });

        // Add database-level optimizations
        if (DB::getDriverName() === 'mysql') {
            // Set MySQL optimizations
            DB::statement('SET GLOBAL innodb_buffer_pool_size = 128M');
            DB::statement('SET GLOBAL query_cache_size = 16M');
            DB::statement('SET GLOBAL key_buffer_size = 16M');
        }
    }

    /**
     * Reverse migrations
     */
    public function down(): void
    {
        // Drop indexes in reverse order
        Schema::table('tags', function (Blueprint $table) {
            $table->dropIndex('tags_slug_idx');
        });

        Schema::table('categories', function (Blueprint $table) {
            $table->dropIndex('categories_slug_idx');
            $table->dropIndex('categories_active_sort_idx');
            $table->dropIndex('categories_parent_idx');
        });

        Schema::table('personal_access_tokens', function (Blueprint $table) {
            $table->dropIndex('tokens_expires_at_idx');
            $table->dropIndex('tokens_tokenable_idx');
            $table->dropIndex('tokens_last_used_idx');
        });

        Schema::table('comments', function (Blueprint $table) {
            $table->dropIndex('comments_post_approved_idx');
            $table->dropIndex('comments_user_created_idx');
            $table->dropIndex('comments_parent_idx');
        });

        Schema::table('posts', function (Blueprint $table) {
            $table->dropIndex('posts_status_published_idx');
            $table->dropIndex('posts_category_status_idx');
            $table->dropIndex('posts_user_status_idx');
            $table->dropIndex('posts_featured_status_idx');
            $table->dropIndex('posts_views_count_idx');
        });

        Schema::table('users', function (Blueprint $table) {
            $table->dropIndex('users_email_active_idx');
            $table->dropIndex('users_role_active_idx');
            $table->dropIndex('users_created_at_idx');
        });
    }
};
```

##### Step 1.4: Create Performance Monitoring Service
```bash
# Create performance monitoring service
mkdir -p app/Services
nano app/Services/ProductionMetricsService.php
```

**Isi file app/Services/ProductionMetricsService.php:**
```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ProductionMetricsService
{
    /**
     * Collect application performance metrics
     */
    public function collectMetrics(): array
    {
        $startTime = microtime(true);
        
        $metrics = [
            'timestamp' => now()->toISOString(),
            'memory' => $this->getMemoryMetrics(),
            'database' => $this->getDatabaseMetrics(),
            'cache' => $this->getCacheMetrics(),
            'application' => $this->getApplicationMetrics(),
            'system' => $this->getSystemMetrics(),
        ];
        
        $metrics['collection_time'] = round((microtime(true) - $startTime) * 1000, 2);
        
        return $metrics;
    }

    /**
     * Get memory usage metrics
     */
    private function getMemoryMetrics(): array
    {
        return [
            'current_usage' => round(memory_get_usage(true) / 1024 / 1024, 2), // MB
            'peak_usage' => round(memory_get_peak_usage(true) / 1024 / 1024, 2), // MB
            'limit' => ini_get('memory_limit'),
            'usage_percentage' => round((memory_get_usage(true) / $this->parseMemoryLimit()) * 100, 2),
        ];
    }

    /**
     * Get database performance metrics
     */
    private function getDatabaseMetrics(): array
    {
        $startTime = microtime(true);
        
        // Test database connectivity
        try {
            DB::connection()->getPdo();
            $connectionTime = round((microtime(true) - $startTime) * 1000, 2);
            $connected = true;
        } catch (\Exception $e) {
            $connectionTime = null;
            $connected = false;
        }
        
        $metrics = [
            'connected' => $connected,
            'connection_time_ms' => $connectionTime,
        ];
        
        if ($connected) {
            // Get database statistics
            $metrics = array_merge($metrics, [
                'total_queries' => DB::getQueryLog() ? count(DB::getQueryLog()) : 0,
                'slow_queries' => $this->getSlowQueriesCount(),
                'active_connections' => $this->getActiveConnectionsCount(),
                'database_size' => $this->getDatabaseSize(),
            ]);
        }
        
        return $metrics;
    }

    /**
     * Get cache performance metrics
     */
    private function getCacheMetrics(): array
    {
        $startTime = microtime(true);
        
        try {
            // Test cache connectivity
            Cache::put('metrics_test', 'test_value', 5);
            $testValue = Cache::get('metrics_test');
            Cache::forget('metrics_test');
            
            $cacheTime = round((microtime(true) - $startTime) * 1000, 2);
            $working = $testValue === 'test_value';
        } catch (\Exception $e) {
            $cacheTime = null;
            $working = false;
        }
        
        return [
            'working' => $working,
            'response_time_ms' => $cacheTime,
            'driver' => config('cache.default'),
            'hit_rate' => $this->getCacheHitRate(),
        ];
    }

    /**
     * Get application-specific metrics
     */
    private function getApplicationMetrics(): array
    {
        return [
            'environment' => app()->environment(),
            'debug_mode' => config('app.debug'),
            'timezone' => config('app.timezone'),
            'locale' => app()->getLocale(),
            'uptime' => $this->getApplicationUptime(),
            'routes_cached' => app()->routesAreCached(),
            'config_cached' => app()->configurationIsCached(),
            'events_cached' => app()->eventsAreCached(),
        ];
    }

    /**
     * Get system-level metrics
     */
    private function getSystemMetrics(): array
    {
        return [
            'php_version' => PHP_VERSION,
            'laravel_version' => app()->version(),
            'server_software' => $_SERVER['SERVER_SOFTWARE'] ?? 'Unknown',
            'load_average' => $this->getLoadAverage(),
            'disk_usage' => $this->getDiskUsage(),
        ];
    }

    /**
     * Helper methods untuk metrics collection
     */
    private function parseMemoryLimit(): int
    {
        $limit = ini_get('memory_limit');
        if ($limit === '-1') {
            return PHP_INT_MAX;
        }
        
        $value = (int) $limit;
        $unit = strtolower(substr($limit, -1));
        
        switch ($unit) {
            case 'g': $value *= 1024;
            case 'm': $value *= 1024;
            case 'k': $value *= 1024;
        }
        
        return $value;
    }

    private function getSlowQueriesCount(): int
    {
        try {
            $result = DB::select("SHOW STATUS LIKE 'Slow_queries'");
            return $result[0]->Value ?? 0;
        } catch (\Exception $e) {
            return 0;
        }
    }

    private function getActiveConnectionsCount(): int
    {
        try {
            $result = DB::select("SHOW STATUS LIKE 'Threads_connected'");
            return $result[0]->Value ?? 0;
        } catch (\Exception $e) {
            return 0;
        }
    }

    private function getDatabaseSize(): float
    {
        try {
            $result = DB::select("
                SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size 
                FROM information_schema.tables 
                WHERE table_schema = DATABASE()
            ");
            return $result[0]->size ?? 0;
        } catch (\Exception $e) {
            return 0;
        }
    }

    private function getCacheHitRate(): float
    {
        // This would require cache-specific implementation
        // For now, return a placeholder
        return 85.5;
    }

    private function getApplicationUptime(): string
    {
        // Calculate uptime based on Laravel start time
        $startTime = defined('LARAVEL_START') ? LARAVEL_START : microtime(true);
        $uptime = microtime(true) - $startTime;
        
        return round($uptime, 2) . ' seconds';
    }

    private function getLoadAverage(): array
    {
        if (function_exists('sys_getloadavg')) {
            $load = sys_getloadavg();
            return [
                '1min' => round($load[0], 2),
                '5min' => round($load[1], 2),
                '15min' => round($load[2], 2),
            ];
        }
        
        return ['1min' => 0, '5min' => 0, '15min' => 0];
    }

    private function getDiskUsage(): array
    {
        $bytes = disk_total_space('.');
        $free = disk_free_space('.');
        $used = $bytes - $free;
        
        return [
            'total_gb' => round($bytes / 1024 / 1024 / 1024, 2),
            'used_gb' => round($used / 1024 / 1024 / 1024, 2),
            'free_gb' => round($free / 1024 / 1024 / 1024, 2),
            'usage_percentage' => round(($used / $bytes) * 100, 2),
        ];
    }

    /**
     * Log performance metrics
     */
    public function logMetrics(): void
    {
        $metrics = $this->collectMetrics();
        
        Log::channel('performance')->info('Application Metrics', $metrics);
        
        // Store metrics dalam cache untuk dashboard
        Cache::put('app.metrics.latest', $metrics, 300); // 5 minutes
    }

    /**
     * Check if application performance is healthy
     */
    public function isHealthy(): bool
    {
        $metrics = $this->collectMetrics();
        
        // Define health criteria
        $healthChecks = [
            $metrics['memory']['usage_percentage'] < 90,
            $metrics['database']['connected'] === true,
            $metrics['cache']['working'] === true,
            $metrics['database']['connection_time_ms'] < 1000,
            $metrics['cache']['response_time_ms'] < 100,
        ];
        
        return !in_array(false, $healthChecks, true);
    }
}
```

#### **Bagian 2: Google Cloud Platform Deployment Setup (30 menit)**

##### Step 2.1: Create Google Cloud Project dan Enable APIs
```bash
# Set up Google Cloud CLI (jika belum)
gcloud auth login
gcloud config set project your-project-id

# Enable required APIs untuk Laravel deployment
gcloud services enable \
    cloudsql.googleapis.com \
    container.googleapis.com \
    storage-api.googleapis.com \
    redis.googleapis.com \
    cloudbuild.googleapis.com \
    run.googleapis.com \
    secretmanager.googleapis.com \
    monitoring.googleapis.com

# Verify APIs enabled
gcloud services list --enabled | grep -E "(sql|container|storage|redis|build|run)"
```

##### Step 2.2: Setup Cloud SQL Database
```bash
# Create Cloud SQL MySQL instance
gcloud sql instances create laravel-production \
    --database-version=MYSQL_8_0 \
    --tier=db-f1-micro \
    --region=asia-southeast2 \
    --storage-type=SSD \
    --storage-size=10GB \
    --backup-start-time=02:00 \
    --enable-bin-log \
    --maintenance-window-day=SUN \
    --maintenance-window-hour=03

# Set root password
gcloud sql users set-password root \
    --host=% \
    --instance=laravel-production \
    --password=your_secure_root_password

# Create database user untuk Laravel
gcloud sql users create laravel_user \
    --instance=laravel-production \
    --password=your_secure_user_password

# Create database
gcloud sql databases create laravel_production \
    --instance=laravel-production

# Grant permissions
gcloud sql users create laravel_user \
    --instance=laravel-production \
    --password=your_secure_password

# Get connection details
gcloud sql instances describe laravel-production \
    --format="value(connectionName,ipAddresses[0].ipAddress)"
```

##### Step 2.3: Setup Cloud Storage dan Redis
```bash
# Create Cloud Storage bucket untuk file storage
gsutil mb -p your-project-id \
    -c STANDARD \
    -l asia-southeast2 \
    gs://your-project-laravel-storage

# Set bucket permissions
gsutil iam ch allUsers:objectViewer gs://your-project-laravel-storage

# Create Redis instance untuk caching
gcloud redis instances create laravel-cache \
    --size=1 \
    --region=asia-southeast2 \
    --redis-version=redis_6_x \
    --tier=basic

# Get Redis connection details
gcloud redis instances describe laravel-cache \
    --region=asia-southeast2 \
    --format="value(host,port)"

# Create service account untuk application
gcloud iam service-accounts create laravel-production \
    --display-name="Laravel Production Service Account"

# Grant necessary permissions
gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:laravel-production@your-project-id.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding your-project-id \
    --member="serviceAccount:laravel-production@your-project-id.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

# Create dan download service account key
gcloud iam service-accounts keys create laravel-sa-key.json \
    --iam-account=laravel-production@your-project-id.iam.gserviceaccount.com
```

##### Step 2.4: Create Dockerfile untuk Laravel Application
```bash
# Create optimized Dockerfile untuk production
nano Dockerfile
```

**Isi file Dockerfile:**
```dockerfile
# Multi-stage build untuk optimized production image
FROM node:18-alpine AS node-builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY vite.config.js ./

# Install dependencies dan build assets
RUN npm ci --only=production
COPY resources/ resources/
RUN npm run build

# PHP Production Stage
FROM php:8.2-fpm-alpine AS php-base

# Install system dependencies
RUN apk add --no-cache \
    nginx \
    mysql-client \
    redis \
    supervisor \
    curl \
    zip \
    unzip \
    git \
    oniguruma-dev \
    libxml2-dev \
    freetype-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    libzip-dev

# Install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        pdo \
        pdo_mysql \
        mysqli \
        mbstring \
        xml \
        zip \
        gd \
        opcache \
        bcmath

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy composer files first untuk better caching
COPY composer.json composer.lock ./

# Install PHP dependencies
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-suggest \
    --optimize-autoloader \
    --prefer-dist

# Copy application code
COPY . .
COPY --from=node-builder /app/public/build ./public/build

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Copy configuration files
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

# Run Laravel optimizations
RUN php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache

# Expose port
EXPOSE 80

# Start supervisor
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1
```

##### Step 2.5: Create Docker Configuration Files
```bash
# Create docker configuration directory
mkdir -p docker

# Create nginx configuration
nano docker/nginx.conf
```

**Isi file docker/nginx.conf:**
```nginx
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging configuration
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # Performance settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 50M;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
    
    server {
        listen 80;
        server_name _;
        root /var/www/html/public;
        index index.php index.html;
        
        # Security headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
        add_header Referrer-Policy "strict-origin-when-cross-origin";
        add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:;";
        
        # Laravel routes
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
        
        # API rate limiting
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            try_files $uri $uri/ /index.php?$query_string;
        }
        
        # Login rate limiting
        location = /login {
            limit_req zone=login burst=5 nodelay;
            try_files $uri /index.php?$query_string;
        }
        
        # PHP-FPM configuration
        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            
            # Security
            fastcgi_hide_header X-Powered-By;
            fastcgi_read_timeout 300;
        }
        
        # Static files caching
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }
        
        # Deny access to sensitive files
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
        
        location ~ /storage/.*\.php$ {
            deny all;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            add_header Content-Type text/plain;
            return 200 "OK";
        }
        
        # Metrics endpoint (for monitoring)
        location /metrics {
            allow 10.0.0.0/8;
            allow 172.16.0.0/12;
            allow 192.168.0.0/16;
            deny all;
            
            try_files $uri /index.php?$query_string;
        }
    }
}
```

```bash
# Create PHP configuration
nano docker/php.ini
```

**Isi file docker/php.ini:**
```ini
; PHP Configuration untuk Production

; Performance settings
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
post_max_size = 50M
upload_max_filesize = 50M
max_file_uploads = 20

; Session settings
session.gc_maxlifetime = 1440
session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1

; Security settings
expose_php = Off
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php_errors.log

; Date settings
date.timezone = Asia/Jakarta

; Realpath cache
realpath_cache_size = 4096K
realpath_cache_ttl = 600
```

```bash
# Create OPcache configuration
nano docker/opcache.ini
```

**Isi file docker/opcache.ini:**
```ini
; OPcache configuration untuk Production Performance

; Enable OPcache
opcache.enable = 1
opcache.enable_cli = 1

; Memory settings
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000

; Performance settings
opcache.validate_timestamps = 0
opcache.revalidate_freq = 0
opcache.save_comments = 0
opcache.enable_file_override = 1

; JIT compilation (PHP 8.0+)
opcache.jit_buffer_size = 100M
opcache.jit = tracing

; Optimization settings
opcache.optimization_level = 0xffffffff
opcache.max_wasted_percentage = 10
opcache.consistency_checks = 0
```

```bash
# Create supervisor configuration
nano docker/supervisord.conf
```

**Isi file docker/supervisord.conf:**
```ini
[supervisord]
nodaemon=true
user=root
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

[program:php-fpm]
command=php-fpm --nodaemonize
stdout_logfile=/var/log/supervisor/php-fpm.log
stderr_logfile=/var/log/supervisor/php-fpm.log
autorestart=true
priority=1

[program:nginx]
command=nginx -g "daemon off;"
stdout_logfile=/var/log/supervisor/nginx.log
stderr_logfile=/var/log/supervisor/nginx.log
autorestart=true
priority=2

[program:laravel-queue]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-queue.log
stopwaitsecs=3600

[program:laravel-schedule]
command=/bin/bash -c "while true; do php /var/www/html/artisan schedule:run --verbose --no-interaction; sleep 60; done"
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/supervisor/laravel-schedule.log
```

#### **Bagian 3: CI/CD Pipeline dengan GitHub Actions (25 menit)**

##### Step 3.1: Create GitHub Actions Workflow
```bash
# Create GitHub Actions directory
mkdir -p .github/workflows

# Create deployment workflow
nano .github/workflows/deploy-production.yml
```

**Isi file .github/workflows/deploy-production.yml:**
```yaml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  SERVICE_NAME: laravel-app
  REGION: asia-southeast2

jobs:
  test:
    runs-on: ubuntu-latest
    name: Test Application
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: 8.2
        extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
        coverage: none

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: 18
        cache: 'npm'

    - name: Install Composer dependencies
      run: composer install --prefer-dist --no-interaction --no-progress --optimize-autoloader

    - name: Install NPM dependencies
      run: npm ci

    - name: Build assets
      run: npm run build

    - name: Create testing environment file
      run: |
        cp .env.example .env.testing
        echo "DB_CONNECTION=mysql" >> .env.testing
        echo "DB_HOST=127.0.0.1" >> .env.testing
        echo "DB_PORT=3306" >> .env.testing
        echo "DB_DATABASE=testing" >> .env.testing
        echo "DB_USERNAME=root" >> .env.testing
        echo "DB_PASSWORD=password" >> .env.testing

    - name: Generate application key
      run: php artisan key:generate --env=testing

    - name: Run database migrations
      run: php artisan migrate --env=testing --force

    - name: Seed database
      run: php artisan db:seed --env=testing --force

    - name: Run PHPUnit tests
      run: php artisan test --env=testing

    - name: Run security analysis
      run: |
        composer require --dev enlightn/security-checker
        vendor/bin/security-checker security:check composer.lock

  security-scan:
    runs-on: ubuntu-latest
    name: Security Scan
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Snyk security scan
      uses: snyk/actions/php@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    name: Deploy to Production
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Google Cloud CLI
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ secrets.GCP_PROJECT_ID }}
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        export_default_credentials: true

    - name: Configure Docker to use gcloud as credential helper
      run: gcloud auth configure-docker

    - name: Build Docker image
      run: |
        docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA .
        docker tag gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA gcr.io/$PROJECT_ID/$SERVICE_NAME:latest

    - name: Push Docker image
      run: |
        docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA
        docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:latest

    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy $SERVICE_NAME \
          --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
          --platform managed \
          --region $REGION \
          --allow-unauthenticated \
          --memory 1Gi \
          --cpu 2 \
          --min-instances 1 \
          --max-instances 10 \
          --timeout 300 \
          --concurrency 100 \
          --set-env-vars "APP_ENV=production" \
          --set-env-vars "APP_DEBUG=false" \
          --set-env-vars "LOG_CHANNEL=stderr" \
          --add-cloudsql-instances ${{ secrets.CLOUD_SQL_CONNECTION_NAME }} \
          --service-account laravel-production@$PROJECT_ID.iam.gserviceaccount.com

    - name: Run database migrations
      run: |
        gcloud run jobs create laravel-migrate \
          --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
          --region $REGION \
          --add-cloudsql-instances ${{ secrets.CLOUD_SQL_CONNECTION_NAME }} \
          --service-account laravel-production@$PROJECT_ID.iam.gserviceaccount.com \
          --set-env-vars "APP_ENV=production" \
          --command "php" \
          --args "artisan,migrate,--force" \
          --max-retries 3 \
          --replace || true

        gcloud run jobs execute laravel-migrate --region $REGION --wait

    - name: Clear application cache
      run: |
        gcloud run jobs create laravel-cache-clear \
          --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
          --region $REGION \
          --service-account laravel-production@$PROJECT_ID.iam.gserviceaccount.com \
          --set-env-vars "APP_ENV=production" \
          --command "php" \
          --args "artisan,cache:clear" \
          --max-retries 3 \
          --replace || true

        gcloud run jobs execute laravel-cache-clear --region $REGION --wait

    - name: Notify deployment status
      if: always()
      run: |
        if [ ${{ job.status }} == 'success' ]; then
          echo "✅ Deployment successful!"
          echo "🌐 Application URL: $(gcloud run services describe $SERVICE_NAME --region $REGION --format 'value(status.url)')"
        else
          echo "❌ Deployment failed!"
        fi

  performance-test:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: deploy
    runs-on: ubuntu-latest
    name: Performance Testing
    
    steps:
    - name: Get service URL
      id: service-url
      run: |
        URL=$(gcloud run services describe $SERVICE_NAME --region $REGION --format 'value(status.url)')
        echo "SERVICE_URL=$URL" >> $GITHUB_OUTPUT

    - name: Run load testing
      run: |
        # Install Apache Bench
        sudo apt-get update
        sudo apt-get install -y apache2-utils
        
        # Run basic load test
        ab -n 100 -c 10 ${{ steps.service-url.outputs.SERVICE_URL }}/
        
        # Test API endpoints
        ab -n 50 -c 5 ${{ steps.service-url.outputs.SERVICE_URL }}/api/info

    - name: Check application health
      run: |
        # Wait for application to be ready
        sleep 30
        
        # Health check
        curl -f ${{ steps.service-url.outputs.SERVICE_URL }}/health || exit 1
        
        # API check
        curl -f ${{ steps.service-url.outputs.SERVICE_URL }}/api/info || exit 1
        
        echo "✅ Application health checks passed!"
```

##### Step 3.2: Create Application Health Check Endpoint
```bash
# Create health check controller
php artisan make:controller HealthController

# Edit health controller
nano app/Http/Controllers/HealthController.php
```

**Isi file app/Http/Controllers/HealthController.php:**
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use App\Services\ProductionMetricsService;

class HealthController extends Controller
{
    /**
     * Basic health check endpoint
     */
    public function index()
    {
        return response()->json([
            'status' => 'OK',
            'timestamp' => now()->toISOString(),
            'service' => 'Laravel Application',
            'version' => config('app.version', '1.0.0'),
        ]);
    }

    /**
     * Detailed health check dengan service dependencies
     */
    public function detailed()
    {
        $health = [
            'status' => 'OK',
            'timestamp' => now()->toISOString(),
            'checks' => [],
        ];

        // Database health check
        try {
            DB::connection()->getPdo();
            $dbTime = microtime(true);
            DB::select('SELECT 1');
            $dbTime = round((microtime(true) - $dbTime) * 1000, 2);
            
            $health['checks']['database'] = [
                'status' => 'OK',
                'response_time_ms' => $dbTime,
                'connection' => DB::connection()->getName(),
            ];
        } catch (\Exception $e) {
            $health['status'] = 'ERROR';
            $health['checks']['database'] = [
                'status' => 'ERROR',
                'error' => $e->getMessage(),
            ];
        }

        // Cache health check
        try {
            $cacheTime = microtime(true);
            Cache::put('health_check', 'test', 5);
            $value = Cache::get('health_check');
            Cache::forget('health_check');
            $cacheTime = round((microtime(true) - $cacheTime) * 1000, 2);
            
            $health['checks']['cache'] = [
                'status' => $value === 'test' ? 'OK' : 'ERROR',
                'response_time_ms' => $cacheTime,
                'driver' => config('cache.default'),
            ];
        } catch (\Exception $e) {
            $health['status'] = 'ERROR';
            $health['checks']['cache'] = [
                'status' => 'ERROR',
                'error' => $e->getMessage(),
            ];
        }

        // Storage health check
        try {
            $storageTime = microtime(true);
            \Storage::put('health_check.txt', 'test');
            $content = \Storage::get('health_check.txt');
            \Storage::delete('health_check.txt');
            $storageTime = round((microtime(true) - $storageTime) * 1000, 2);
            
            $health['checks']['storage'] = [
                'status' => $content === 'test' ? 'OK' : 'ERROR',
                'response_time_ms' => $storageTime,
                'driver' => config('filesystems.default'),
            ];
        } catch (\Exception $e) {
            $health['checks']['storage'] = [
                'status' => 'WARNING',
                'error' => $e->getMessage(),
            ];
        }

        // Queue health check
        try {
            $queueTime = microtime(true);
            // Simple queue connectivity test
            $connection = \Queue::connection();
            $queueTime = round((microtime(true) - $queueTime) * 1000, 2);
            
            $health['checks']['queue'] = [
                'status' => 'OK',
                'response_time_ms' => $queueTime,
                'driver' => config('queue.default'),
            ];
        } catch (\Exception $e) {
            $health['checks']['queue'] = [
                'status' => 'WARNING',
                'error' => $e->getMessage(),
            ];
        }

        $httpStatus = $health['status'] === 'OK' ? 200 : 503;
        
        return response()->json($health, $httpStatus);
    }

    /**
     * Application metrics endpoint
     */
    public function metrics(ProductionMetricsService $metricsService)
    {
        // Check if request is from allowed sources
        $allowedIPs = ['127.0.0.1', '::1'];
        $clientIP = request()->ip();
        
        // Allow internal network ranges
        if (!in_array($clientIP, $allowedIPs) && 
            !$this->isInternalIP($clientIP)) {
            return response()->json(['error' => 'Access denied'], 403);
        }

        $metrics = $metricsService->collectMetrics();
        
        return response()->json($metrics);
    }

    /**
     * Readiness probe untuk Kubernetes/Cloud Run
     */
    public function ready()
    {
        // Check critical services untuk readiness
        $ready = true;
        $checks = [];

        // Database readiness
        try {
            DB::select('SELECT 1');
            $checks['database'] = 'ready';
        } catch (\Exception $e) {
            $ready = false;
            $checks['database'] = 'not_ready';
        }

        // Cache readiness
        try {
            Cache::get('test');
            $checks['cache'] = 'ready';
        } catch (\Exception $e) {
            $ready = false;
            $checks['cache'] = 'not_ready';
        }

        $status = $ready ? 'ready' : 'not_ready';
        $httpStatus = $ready ? 200 : 503;

        return response()->json([
            'status' => $status,
            'checks' => $checks,
            'timestamp' => now()->toISOString(),
        ], $httpStatus);
    }

    /**
     * Liveness probe untuk Kubernetes/Cloud Run
     */
    public function live()
    {
        // Basic liveness check - application is running
        return response()->json([
            'status' => 'alive',
            'timestamp' => now()->toISOString(),
            'uptime' => $this->getUptime(),
        ]);
    }

    /**
     * Helper methods
     */
    private function isInternalIP($ip): bool
    {
        $internalRanges = [
            '10.0.0.0/8',
            '172.16.0.0/12',
            '192.168.0.0/16',
        ];

        foreach ($internalRanges as $range) {
            if ($this->ipInRange($ip, $range)) {
                return true;
            }
        }

        return false;
    }

    private function ipInRange($ip, $range): bool
    {
        list($subnet, $bits) = explode('/', $range);
        $ip = ip2long($ip);
        $subnet = ip2long($subnet);
        $mask = -1 << (32 - $bits);
        $subnet &= $mask;
        
        return ($ip & $mask) == $subnet;
    }

    private function getUptime(): string
    {
        if (defined('LARAVEL_START')) {
            $uptime = microtime(true) - LARAVEL_START;
            return round($uptime, 2) . ' seconds';
        }
        
        return 'unknown';
    }
}
```

##### Step 3.3: Add Health Check Routes
```bash
# Add health check routes
nano routes/web.php
```

**Tambahkan health check routes ke web.php:**
```php
// Add after existing routes

/*
|--------------------------------------------------------------------------
| Health Check Routes
|--------------------------------------------------------------------------
*/

Route::get('/health', [App\Http\Controllers\HealthController::class, 'index'])
     ->name('health');

Route::get('/health/detailed', [App\Http\Controllers\HealthController::class, 'detailed'])
     ->name('health.detailed');

Route::get('/ready', [App\Http\Controllers\HealthController::class, 'ready'])
     ->name('health.ready');

Route::get('/live', [App\Http\Controllers\HealthController::class, 'live'])
     ->name('health.live');

Route::get('/metrics', [App\Http\Controllers\HealthController::class, 'metrics'])
     ->name('health.metrics')
     ->middleware('throttle:60,1');
```

#### **Bagian 4: Security Hardening dan Monitoring (20 menit)**

##### Step 4.1: Implement Security Headers Middleware
```bash
# Create security headers middleware
php artisan make:middleware SecurityHeaders

# Edit security headers middleware
nano app/Http/Middleware/SecurityHeaders.php
```

**Isi file app/Http/Middleware/SecurityHeaders.php:**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class SecurityHeaders
{
    /**
     * Handle incoming request dengan security headers
     */
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        // Content Security Policy
        $csp = [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.tailwindcss.com https://fonts.bunny.net",
            "style-src 'self' 'unsafe-inline' https://fonts.bunny.net https://cdn.tailwindcss.com",
            "img-src 'self' data: https: blob:",
            "font-src 'self' data: https://fonts.bunny.net",
            "connect-src 'self' https:",
            "media-src 'self'",
            "object-src 'none'",
            "frame-src 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "frame-ancestors 'none'",
        ];

        // Security headers
        $response->headers->set('Content-Security-Policy', implode('; ', $csp));
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        
        // HSTS (HTTP Strict Transport Security) untuk HTTPS
        if ($request->isSecure()) {
            $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
        }
        
        // Remove server information disclosure
        $response->headers->remove('Server');
        $response->headers->set('Server', 'Praktikum CC ITK');
        
        return $response;
    }
}
```

##### Step 4.2: Register Security Middleware
```bash
# Register security middleware dalam Kernel
nano app/Http/Kernel.php
```

**Update Kernel.php untuk add security middleware:**
```php
// In app/Http/Kernel.php, tambahkan di $middleware array:

protected $middleware = [
    // ... existing middleware
    \App\Http\Middleware\SecurityHeaders::class,
];

// Atau di $middlewareGroups untuk specific groups:
protected $middlewareGroups = [
    'web' => [
        // ... existing web middleware
        \App\Http\Middleware\SecurityHeaders::class,
    ],
    
    'api' => [
        // ... existing api middleware  
        \App\Http\Middleware\SecurityHeaders::class,
    ],
];
```

##### Step 4.3: Create Backup and Recovery Commands
```bash
# Create backup command
php artisan make:command BackupDatabase

# Edit backup command
nano app/Console/Commands/BackupDatabase.php
```

**Isi file app/Console/Commands/BackupDatabase.php:**
```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\DB;

class BackupDatabase extends Command
{
    /**
     * The name and signature of the console command.
     */
    protected $signature = 'backup:database {--compress} {--upload-cloud}';

    /**
     * The console command description.
     */
    protected $description = 'Create database backup dengan optional compression dan cloud upload';

    /**
     * Execute the console command.
     */
    public function handle()
    {
        $this->info('Starting database backup...');
        
        $timestamp = now()->format('Y-m-d_H-i-s');
        $filename = "backup_database_{$timestamp}.sql";
        $tempPath = storage_path("app/backups/{$filename}");
        
        // Ensure backup directory exists
        Storage::makeDirectory('backups');
        
        // Get database configuration
        $connection = config('database.default');
        $database = config("database.connections.{$connection}");
        
        // Create mysqldump command
        $command = sprintf(
            'mysqldump -h%s -P%s -u%s -p%s %s > %s',
            $database['host'],
            $database['port'],
            $database['username'],
            $database['password'],
            $database['database'],
            $tempPath
        );
        
        // Execute backup
        $result = null;
        $output = [];
        exec($command, $output, $result);
        
        if ($result !== 0) {
            $this->error('Database backup failed!');
            return 1;
        }
        
        $this->info("Database backup created: {$filename}");
        
        // Compress if requested
        if ($this->option('compress')) {
            $this->info('Compressing backup...');
            $compressedFilename = $filename . '.gz';
            $compressedPath = storage_path("app/backups/{$compressedFilename}");
            
            exec("gzip -c {$tempPath} > {$compressedPath}");
            unlink($tempPath);
            
            $filename = $compressedFilename;
            $tempPath = $compressedPath;
            $this->info("Backup compressed: {$filename}");
        }
        
        // Upload to cloud storage if requested
        if ($this->option('upload-cloud')) {
            $this->info('Uploading to cloud storage...');
            
            try {
                $cloudPath = "backups/database/{$filename}";
                Storage::disk('gcs')->put($cloudPath, file_get_contents($tempPath));
                $this->info("Backup uploaded to cloud: {$cloudPath}");
                
                // Remove local file after successful upload
                unlink($tempPath);
                $this->info('Local backup file removed.');
                
            } catch (\Exception $e) {
                $this->error("Failed to upload backup to cloud: {$e->getMessage()}");
                $this->info("Local backup retained at: {$tempPath}");
                return 1;
            }
        }
        
        $this->info('Database backup completed successfully!');
        return 0;
    }
}
```

##### Step 4.4: Create Performance Monitoring Command
```bash
# Create performance monitoring command  
php artisan make:command MonitorPerformance

# Edit monitoring command
nano app/Console/Commands/MonitorPerformance.php
```

**Isi file app/Console/Commands/MonitorPerformance.php:**
```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Services\ProductionMetricsService;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;

class MonitorPerformance extends Command
{
    /**
     * The name and signature of the console command.
     */
    protected $signature = 'monitor:performance {--alert} {--store-metrics}';

    /**
     * The console command description.
     */
    protected $description = 'Monitor application performance dan send alerts jika necessary';

    /**
     * Execute the console command.
     */
    public function handle(ProductionMetricsService $metricsService)
    {
        $this->info('Collecting performance metrics...');
        
        $metrics = $metricsService->collectMetrics();
        $isHealthy = $metricsService->isHealthy();
        
        // Display metrics summary
        $this->displayMetricsSummary($metrics, $isHealthy);
        
        // Store metrics if requested
        if ($this->option('store-metrics')) {
            $this->storeMetrics($metrics);
        }
        
        // Send alerts if necessary
        if ($this->option('alert') && !$isHealthy) {
            $this->sendHealthAlert($metrics);
        }
        
        return $isHealthy ? 0 : 1;
    }
    
    /**
     * Display metrics summary
     */
    private function displayMetricsSummary(array $metrics, bool $isHealthy): void
    {
        $this->line('=== Performance Metrics Summary ===');
        
        // Memory metrics
        $memory = $metrics['memory'];
        $this->line("Memory Usage: {$memory['current_usage']}MB / {$memory['peak_usage']}MB peak ({$memory['usage_percentage']}%)");
        
        // Database metrics
        $database = $metrics['database'];
        if ($database['connected']) {
            $this->line("Database: Connected ({$database['connection_time_ms']}ms)");
            $this->line("Database Size: {$database['database_size']}MB");
        } else {
            $this->error('Database: Disconnected');
        }
        
        // Cache metrics
        $cache = $metrics['cache'];
        if ($cache['working']) {
            $this->line("Cache: Working ({$cache['response_time_ms']}ms, {$cache['hit_rate']}% hit rate)");
        } else {
            $this->error('Cache: Not working');
        }
        
        // System metrics
        $system = $metrics['system'];
        $load = $system['load_average'];
        $this->line("Load Average: {$load['1min']} / {$load['5min']} / {$load['15min']}");
        
        $disk = $system['disk_usage'];
        $this->line("Disk Usage: {$disk['used_gb']}GB / {$disk['total_gb']}GB ({$disk['usage_percentage']}%)");
        
        // Overall health
        if ($isHealthy) {
            $this->info('✅ Application health: HEALTHY');
        } else {
            $this->error('❌ Application health: UNHEALTHY');
        }
    }
    
    /**
     * Store metrics untuk historical analysis
     */
    private function storeMetrics(array $metrics): void
    {
        $this->info('Storing metrics...');
        
        // Store in cache dengan timestamp
        $cacheKey = 'metrics.history.' . now()->format('Y-m-d-H-i');
        Cache::put($cacheKey, $metrics, 86400); // 24 hours
        
        // Store latest metrics
        Cache::put('metrics.latest', $metrics, 300); // 5 minutes
        
        // Log metrics untuk external monitoring systems
        Log::channel('performance')->info('Performance Metrics', $metrics);
        
        $this->info('Metrics stored successfully.');
    }
    
    /**
     * Send health alert
     */
    private function sendHealthAlert(array $metrics): void
    {
        $this->error('Application is unhealthy! Sending alerts...');
        
        $alertData = [
            'timestamp' => $metrics['timestamp'],
            'memory_usage' => $metrics['memory']['usage_percentage'],
            'database_connected' => $metrics['database']['connected'],
            'cache_working' => $metrics['cache']['working'],
            'disk_usage' => $metrics['system']['disk_usage']['usage_percentage'],
        ];
        
        // Log critical alert
        Log::critical('Application Health Alert', $alertData);
        
        // Send notification (example implementation)
        try {
            // You can implement email, Slack, or other notification methods here
            $this->sendNotification($alertData);
            $this->info('Health alert sent successfully.');
        } catch (\Exception $e) {
            $this->error("Failed to send alert: {$e->getMessage()}");
        }
    }
    
    /**
     * Send notification (placeholder implementation)
     */
    private function sendNotification(array $alertData): void
    {
        // Example: Send to a webhook endpoint
        // $webhook = config('monitoring.webhook_url');
        // Http::post($webhook, $alertData);
        
        // Example: Send email notification
        // Mail::to(config('monitoring.alert_email'))->send(new HealthAlert($alertData));
        
        // For now, just log the alert
        Log::info('Health alert notification would be sent here', $alertData);
    }
}
```

##### Step 4.5: Schedule Automated Tasks
```bash
# Update the scheduler untuk automated tasks
nano app/Console/Kernel.php
```

**Update app/Console/Kernel.php:**
```php
<?php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    /**
     * Define the application's command schedule.
     */
    protected function schedule(Schedule $schedule): void
    {
        // Database backup - daily at 2 AM
        $schedule->command('backup:database --compress --upload-cloud')
                 ->dailyAt('02:00')
                 ->environments(['production'])
                 ->runInBackground()
                 ->emailOutputOnFailure('admin@praktikum-cc.itk.ac.id');
        
        // Performance monitoring - every 5 minutes
        $schedule->command('monitor:performance --alert --store-metrics')
                 ->everyFiveMinutes()
                 ->environments(['production'])
                 ->runInBackground()
                 ->withoutOverlapping();
        
        // Clear expired tokens - daily at 3 AM
        $schedule->command('sanctum:prune-expired --hours=24')
                 ->dailyAt('03:00')
                 ->environments(['production']);
        
        // Clear application cache - weekly
        $schedule->command('cache:clear')
                 ->weekly()
                 ->sundays()
                 ->at('04:00')
                 ->environments(['production']);
        
        // Clear old log files - weekly
        $schedule->command('log:clear')
                 ->weekly()
                 ->sundays()
                 ->at('05:00')
                 ->environments(['production']);
        
        // Application health check - every minute
        $schedule->call(function () {
            $metricsService = app(\App\Services\ProductionMetricsService::class);
            $isHealthy = $metricsService->isHealthy();
            
            if (!$isHealthy) {
                \Log::warning('Application health check failed');
            }
            
            // Store health status
            \Cache::put('app.health.status', $isHealthy, 120); // 2 minutes
        })->everyMinute()->environments(['production']);
        
        // Update sitemap - daily at 6 AM
        $schedule->call(function () {
            // Generate sitemap untuk SEO
            \Artisan::call('sitemap:generate');
        })->dailyAt('06:00')->environments(['production']);
        
        // Database maintenance - weekly
        $schedule->call(function () {
            // Optimize database tables
            \DB::statement('OPTIMIZE TABLE users, posts, categories, comments');
        })->weekly()->sundays()->at('03:30')->environments(['production']);
    }

    /**
     * Register the commands for the application.
     */
    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Production Environment Setup Testing
```bash
echo "=== Production Environment Setup Testing ==="

# Test 1.1: Verify production environment file exists
if [ -f ".env.production" ]; then
    echo "✓ Production environment file exists"
else
    echo "✗ Production environment file missing"
fi

# Test 1.2: Check optimization service provider
if [ -f "app/Providers/OptimizationServiceProvider.php" ]; then
    echo "✓ Optimization service provider exists"
    
    # Check if registered in config
    if grep -q "OptimizationServiceProvider" config/app.php; then
        echo "✓ Optimization service provider registered"
    else
        echo "⚠ Optimization service provider not registered in config/app.php"
    fi
else
    echo "✗ Optimization service provider missing"
fi

# Test 1.3: Verify database migrations for optimization
if ls database/migrations/*optimize_database_for_production* 1> /dev/null 2>&1; then
    echo "✓ Database optimization migration exists"
else
    echo "✗ Database optimization migration missing"
fi

# Test 1.4: Test production metrics service
php artisan tinker --execute="
try {
    \$service = new \App\Services\ProductionMetricsService();
    \$metrics = \$service->collectMetrics();
    echo '✓ Production metrics service working' . PHP_EOL;
    echo '  Memory usage: ' . \$metrics['memory']['current_usage'] . 'MB' . PHP_EOL;
    echo '  Database connected: ' . (\$metrics['database']['connected'] ? 'Yes' : 'No') . PHP_EOL;
    echo '  Cache working: ' . (\$metrics['cache']['working'] ? 'Yes' : 'No') . PHP_EOL;
} catch (\Exception \$e) {
    echo '✗ Production metrics service error: ' . \$e->getMessage() . PHP_EOL;
}
"
```

#### Test 2: Docker Configuration Testing
```bash
echo "=== Docker Configuration Testing ==="

# Test 2.1: Verify Dockerfile exists
if [ -f "Dockerfile" ]; then
    echo "✓ Dockerfile exists"
    
    # Check if multi-stage build
    if grep -q "FROM.*AS" Dockerfile; then
        echo "✓ Multi-stage build configured"
    else
        echo "⚠ Single-stage build (consider multi-stage for optimization)"
    fi
else
    echo "✗ Dockerfile missing"
fi

# Test 2.2: Check docker configuration files
DOCKER_CONFIGS=("nginx.conf" "php.ini" "opcache.ini" "supervisord.conf")
for config in "${DOCKER_CONFIGS[@]}"; do
    if [ -f "docker/$config" ]; then
        echo "✓ Docker config $config exists"
    else
        echo "✗ Docker config $config missing"
    fi
done

# Test 2.3: Validate nginx configuration syntax
if [ -f "docker/nginx.conf" ]; then
    # Basic syntax check (if nginx is available)
    if command -v nginx >/dev/null 2>&1; then
        if nginx -t -c $(pwd)/docker/nginx.conf 2>/dev/null; then
            echo "✓ Nginx configuration syntax valid"
        else
            echo "⚠ Nginx configuration syntax may have issues"
        fi
    else
        echo "⚠ Nginx not available for syntax checking"
    fi
fi

# Test 2.4: Check PHP configuration
if [ -f "docker/php.ini" ]; then
    # Check for production settings
    if grep -q "display_errors = Off" docker/php.ini; then
        echo "✓ Production PHP settings configured"
    else
        echo "⚠ Development PHP settings detected"
    fi
fi
```

#### Test 3: CI/CD Pipeline Testing
```bash
echo "=== CI/CD Pipeline Testing ==="

# Test 3.1: Verify GitHub Actions workflow exists
if [ -f ".github/workflows/deploy-production.yml" ]; then
    echo "✓ GitHub Actions workflow exists"
    
    # Check for required jobs
    REQUIRED_JOBS=("test" "security-scan" "deploy" "performance-test")
    for job in "${REQUIRED_JOBS[@]}"; do
        if grep -q "name:.*$job" .github/workflows/deploy-production.yml; then
            echo "✓ Job $job configured"
        else
            echo "✗ Job $job missing"
        fi
    done
else
    echo "✗ GitHub Actions workflow missing"
fi

# Test 3.2: Check health check endpoints
if [ -f "app/Http/Controllers/HealthController.php" ]; then
    echo "✓ Health check controller exists"
    
    # Test if methods exist
    HEALTH_METHODS=("index" "detailed" "ready" "live" "metrics")
    for method in "${HEALTH_METHODS[@]}"; do
        if grep -q "function $method" app/Http/Controllers/HealthController.php; then
            echo "✓ Health check method $method exists"
        else
            echo "✗ Health check method $method missing"
        fi
    done
else
    echo "✗ Health check controller missing"
fi

# Test 3.3: Verify health check routes
if grep -q "health" routes/web.php; then
    echo "✓ Health check routes registered"
else
    echo "✗ Health check routes missing"
fi

# Test 3.4: Test health endpoints locally
if php artisan serve --host=127.0.0.1 --port=8001 &>/dev/null & then
    SERVER_PID=$!
    sleep 3
    
    # Test basic health endpoint
    if curl -s -f http://127.0.0.1:8001/health >/dev/null; then
        echo "✓ Basic health endpoint working"
    else
        echo "✗ Basic health endpoint not working"
    fi
    
    # Test detailed health endpoint
    if curl -s -f http://127.0.0.1:8001/health/detailed >/dev/null; then
        echo "✓ Detailed health endpoint working"
    else
        echo "✗ Detailed health endpoint not working"
    fi
    
    # Clean up
    kill $SERVER_PID 2>/dev/null
else
    echo "⚠ Could not start test server"
fi
```

#### Test 4: Security Hardening Testing
```bash
echo "=== Security Hardening Testing ==="

# Test 4.1: Verify security headers middleware
if [ -f "app/Http/Middleware/SecurityHeaders.php" ]; then
    echo "✓ Security headers middleware exists"
    
    # Check for required headers
    SECURITY_HEADERS=("Content-Security-Policy" "X-Content-Type-Options" "X-Frame-Options" "X-XSS-Protection")
    for header in "${SECURITY_HEADERS[@]}"; do
        if grep -q "$header" app/Http/Middleware/SecurityHeaders.php; then
            echo "✓ Security header $header configured"
        else
            echo "✗ Security header $header missing"
        fi
    done
else
    echo "✗ Security headers middleware missing"
fi

# Test 4.2: Check if security middleware is registered
if grep -q "SecurityHeaders" app/Http/Kernel.php; then
    echo "✓ Security headers middleware registered"
else
    echo "✗ Security headers middleware not registered"
fi

# Test 4.3: Verify backup and monitoring commands
if [ -f "app/Console/Commands/BackupDatabase.php" ]; then
    echo "✓ Database backup command exists"
else
    echo "✗ Database backup command missing"
fi

if [ -f "app/Console/Commands/MonitorPerformance.php" ]; then
    echo "✓ Performance monitoring command exists"
else
    echo "✗ Performance monitoring command missing"
fi

# Test 4.4: Check scheduled tasks
if grep -q "backup:database" app/Console/Kernel.php; then
    echo "✓ Database backup scheduled"
else
    echo "✗ Database backup not scheduled"
fi

if grep -q "monitor:performance" app/Console/Kernel.php; then
    echo "✓ Performance monitoring scheduled"
else
    echo "✗ Performance monitoring not scheduled"
fi

# Test 4.5: Test security headers locally
if php artisan serve --host=127.0.0.1 --port=8002 &>/dev/null & then
    SERVER_PID=$!
    sleep 3
    
    # Test security headers
    HEADERS_RESPONSE=$(curl -s -I http://127.0.0.1:8002/)
    
    if echo "$HEADERS_RESPONSE" | grep -q "X-Content-Type-Options"; then
        echo "✓ Security headers working"
    else
        echo "✗ Security headers not working"
    fi
    
    if echo "$HEADERS_RESPONSE" | grep -q "X-Frame-Options"; then
        echo "✓ X-Frame-Options header present"
    else
        echo "✗ X-Frame-Options header missing"
    fi
    
    # Clean up
    kill $SERVER_PID 2>/dev/null
else
    echo "⚠ Could not start test server for security testing"
fi
```

#### Test 5: Performance Optimization Testing
```bash
echo "=== Performance Optimization Testing ==="

# Test 5.1: Check database indexes from migration
php artisan tinker --execute="
try {
    \$indexes = \DB::select('SHOW INDEX FROM users WHERE Key_name != \"PRIMARY\"');
    if (count(\$indexes) > 0) {
        echo '✓ Database indexes created for users table' . PHP_EOL;
        foreach (\$indexes as \$index) {
            echo '  ' . \$index->Key_name . PHP_EOL;
        }
    } else {
        echo '⚠ No custom indexes found for users table' . PHP_EOL;
    }
} catch (\Exception \$e) {
    echo '⚠ Could not check database indexes: ' . \$e->getMessage() . PHP_EOL;
}
"

# Test 5.2: Check Laravel optimizations
OPTIMIZATIONS=("config" "route" "view")
for opt in "${OPTIMIZATIONS[@]}"; do
    if [ -f "bootstrap/cache/${opt}.php" ]; then
        echo "✓ $opt cache exists"
    else
        echo "⚠ $opt cache missing (run php artisan ${opt}:cache)"
    fi
done

# Test 5.3: Test OPcache configuration
php artisan tinker --execute="
if (function_exists('opcache_get_status')) {
    \$status = opcache_get_status();
    if (\$status && \$status['opcache_enabled']) {
        echo '✓ OPcache enabled' . PHP_EOL;
        echo '  Memory usage: ' . round(\$status['memory_usage']['used_memory'] / 1024 / 1024, 2) . 'MB' . PHP_EOL;
        echo '  Cached files: ' . \$status['opcache_statistics']['num_cached_scripts'] . PHP_EOL;
    } else {
        echo '⚠ OPcache not enabled' . PHP_EOL;
    }
} else {
    echo '⚠ OPcache not available' . PHP_EOL;
}
"

# Test 5.4: Check memory usage
php artisan tinker --execute="
echo 'Current memory usage: ' . round(memory_get_usage(true) / 1024 / 1024, 2) . 'MB' . PHP_EOL;
echo 'Peak memory usage: ' . round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB' . PHP_EOL;
echo 'Memory limit: ' . ini_get('memory_limit') . PHP_EOL;
"

# Test 5.5: Test application performance
TIME_START=$(php -r "echo microtime(true);")
php artisan tinker --execute="
\$users = \App\Models\User::with('posts')->take(10)->get();
echo 'Loaded ' . \$users->count() . ' users with posts' . PHP_EOL;
"
TIME_END=$(php -r "echo microtime(true);")
EXECUTION_TIME=$(php -r "echo round($TIME_END - $TIME_START, 3);")
echo "Query execution time: ${EXECUTION_TIME}s"

if (( $(echo "$EXECUTION_TIME < 1.0" | bc -l) )); then
    echo "✓ Good query performance"
else
    echo "⚠ Slow query performance (consider optimization)"
fi
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Docker Build Failures
**Gejala:** Docker build fails dengan dependency atau permission errors
**Solusi:**
```bash
# Check Docker daemon is running
docker info >/dev/null 2>&1 && echo "✓ Docker daemon running" || echo "✗ Docker daemon not running"

# Clear Docker build cache
docker builder prune -a

# Build dengan verbose output untuk debugging
docker build --no-cache --progress=plain -t laravel-app:debug .

# Check for common issues
echo "Checking common Docker issues..."

# Check file permissions
find . -name "Dockerfile" -exec ls -la {} \;
find docker/ -name "*.conf" -exec ls -la {} \; 2>/dev/null || echo "Docker config directory missing"

# Verify base images are available
docker pull node:18-alpine
docker pull php:8.2-fpm-alpine

# Test multi-stage build separately
docker build --target node-builder -t laravel-app:node .
docker build --target php-base -t laravel-app:php .

# Check for disk space
df -h | grep -E "(/$|/var)"
```

#### Problem 2: Google Cloud Deployment Issues
**Gejala:** Cloud Run deployment fails atau Cloud SQL tidak accessible
**Solusi:**
```bash
# Check Google Cloud CLI authentication
gcloud auth list
gcloud config get-value project

# Verify APIs are enabled
gcloud services list --enabled | grep -E "(cloudsql|run|storage)"

# Check Cloud SQL instance status
gcloud sql instances list
gcloud sql instances describe laravel-production

# Test Cloud SQL connectivity
gcloud sql connect laravel-production --user=laravel_user

# Check Cloud Run service status
gcloud run services list
gcloud run services describe laravel-app --region=asia-southeast2

# Debug Cloud Run logs
gcloud logs read --filter="resource.type=cloud_run_revision AND resource.labels.service_name=laravel-app" --limit=50

# Check service account permissions
gcloud projects get-iam-policy your-project-id --flatten="bindings[].members" --filter="bindings.members:laravel-production@*"

# Test local Docker container with production environment
docker run --env-file .env.production -p 8080:80 laravel-app:latest

# Check network connectivity
gcloud compute networks list
gcloud sql instances describe laravel-production --format="value(ipAddresses[0].ipAddress)"
```

#### Problem 3: Database Migration and Optimization Issues
**Gejala:** Migration fails atau database performance poor setelah optimization
**Solusi:**
```bash
# Check migration status
php artisan migrate:status

# Rollback last migration if needed
php artisan migrate:rollback --step=1

# Test migration pada separate database
cp .env .env.migration-test
# Update database name in .env.migration-test
php artisan migrate --env=migration-test

# Check database indexes
php artisan tinker --execute="
\$tables = ['users', 'posts', 'categories', 'comments'];
foreach (\$tables as \$table) {
    try {
        \$indexes = \DB::select('SHOW INDEX FROM ' . \$table);
        echo 'Indexes for ' . \$table . ':' . PHP_EOL;
        foreach (\$indexes as \$index) {
            echo '  ' . \$index->Key_name . ' (' . \$index->Column_name . ')' . PHP_EOL;
        }
        echo PHP_EOL;
    } catch (\Exception \$e) {
        echo 'Error checking ' . \$table . ': ' . \$e->getMessage() . PHP_EOL;
    }
}
"

# Test query performance with EXPLAIN
php artisan tinker --execute="
\$queries = [
    'SELECT * FROM users WHERE email = \"test@example.com\" AND is_active = 1',
    'SELECT * FROM posts WHERE status = \"published\" ORDER BY published_at DESC LIMIT 10',
    'SELECT * FROM posts WHERE category_id = 1 AND status = \"published\"'
];

foreach (\$queries as \$query) {
    echo 'Query: ' . \$query . PHP_EOL;
    try {
        \$explain = \DB::select('EXPLAIN ' . \$query);
        foreach (\$explain as \$row) {
            echo '  Key: ' . (\$row->key ?? 'None') . ', Type: ' . \$row->type . PHP_EOL;
        }
    } catch (\Exception \$e) {
        echo '  Error: ' . \$e->getMessage() . PHP_EOL;
    }
    echo PHP_EOL;
}
"

# Check database size and optimization
php artisan tinker --execute="
\$size = \DB::select('SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb,
    ROUND(SUM(data_free) / 1024 / 1024, 2) AS free_mb
    FROM information_schema.tables 
    WHERE table_schema = DATABASE()')[0];

echo 'Database size: ' . \$size->size_mb . 'MB' . PHP_EOL;
echo 'Free space: ' . \$size->free_mb . 'MB' . PHP_EOL;

// Check for fragmentation
\$tables = \DB::select('SELECT table_name, data_free FROM information_schema.tables WHERE table_schema = DATABASE() AND data_free > 0');
if (count(\$tables) > 0) {
    echo 'Tables with fragmentation:' . PHP_EOL;
    foreach (\$tables as \$table) {
        echo '  ' . \$table->table_name . ': ' . round(\$table->data_free / 1024 / 1024, 2) . 'MB' . PHP_EOL;
    }
} else {
    echo 'No table fragmentation detected' . PHP_EOL;
}
"
```

#### Problem 4: Security Headers and Middleware Issues
**Gejala:** Security headers tidak terpakai atau middleware conflict
**Solusi:**
```bash
# Check middleware registration order
grep -A 30 "protected \$middleware" app/Http/Kernel.php

# Test middleware execution
php artisan tinker --execute="
\$kernel = app(\Illuminate\Contracts\Http\Kernel::class);
\$middleware = \$kernel->getMiddleware();
echo 'Global middleware:' . PHP_EOL;
foreach (\$middleware as \$mw) {
    echo '  ' . \$mw . PHP_EOL;
}
"

# Test security headers manually
php artisan serve --host=127.0.0.1 --port=8003 &
SERVER_PID=$!
sleep 3

echo "Testing security headers..."
SECURITY_TEST=$(curl -s -I http://127.0.0.1:8003/ | grep -E "(X-Content-Type-Options|X-Frame-Options|X-XSS-Protection|Content-Security-Policy)")

if [ -n "$SECURITY_TEST" ]; then
    echo "✓ Security headers found:"
    echo "$SECURITY_TEST"
else
    echo "✗ Security headers missing"
fi

# Clean up
kill $SERVER_PID 2>/dev/null

# Check CSP syntax
if [ -f "app/Http/Middleware/SecurityHeaders.php" ]; then
    echo "Validating Content Security Policy syntax..."
    CSP_LINE=$(grep -n "Content-Security-Policy" app/Http/Middleware/SecurityHeaders.php | head -1)
    echo "CSP configuration: $CSP_LINE"
fi

# Test middleware priority
php artisan tinker --execute="
\$request = \Illuminate\Http\Request::create('/', 'GET');
\$middlewareGroups = config('app.middleware_groups.web', []);
echo 'Web middleware order:' . PHP_EOL;
foreach (\$middlewareGroups as \$index => \$mw) {
    echo '  ' . (\$index + 1) . '. ' . \$mw . PHP_EOL;
}
"
```

#### Problem 5: Performance Monitoring and Metrics Collection Issues
**Gejala:** Metrics collection fails atau performance monitoring tidak accurate
**Solusi:**
```bash
# Test metrics service directly
php artisan tinker --execute="
try {
    \$service = new \App\Services\ProductionMetricsService();
    \$metrics = \$service->collectMetrics();
    echo '✓ Metrics collection working' . PHP_EOL;
    
    // Test each metric category
    \$categories = ['memory', 'database', 'cache', 'application', 'system'];
    foreach (\$categories as \$category) {
        if (isset(\$metrics[\$category])) {
            echo '✓ ' . ucfirst(\$category) . ' metrics collected' . PHP_EOL;
        } else {
            echo '✗ ' . ucfirst(\$category) . ' metrics missing' . PHP_EOL;
        }
    }
    
    // Test health check
    \$isHealthy = \$service->isHealthy();
    echo 'Health status: ' . (\$isHealthy ? 'Healthy' : 'Unhealthy') . PHP_EOL;
    
} catch (\Exception \$e) {
    echo '✗ Metrics service error: ' . \$e->getMessage() . PHP_EOL;
    echo 'Stack trace: ' . \$e->getTraceAsString() . PHP_EOL;
}
"

# Check system functions availability
php artisan tinker --execute="
\$functions = ['memory_get_usage', 'memory_get_peak_usage', 'sys_getloadavg', 'disk_total_space', 'disk_free_space'];
foreach (\$functions as \$func) {
    if (function_exists(\$func)) {
        echo '✓ Function ' . \$func . ' available' . PHP_EOL;
    } else {
        echo '✗ Function ' . \$func . ' not available' . PHP_EOL;
    }
}
"

# Test database metrics collection
php artisan tinker --execute="
try {
    \$startTime = microtime(true);
    \DB::select('SELECT 1');
    \$queryTime = round((microtime(true) - \$startTime) * 1000, 2);
    echo '✓ Database query time: ' . \$queryTime . 'ms' . PHP_EOL;
    
    // Test MySQL-specific queries
    try {
        \$result = \DB::select('SHOW STATUS LIKE \"Threads_connected\"');
        echo '✓ MySQL status queries working' . PHP_EOL;
    } catch (\Exception \$e) {
        echo '⚠ MySQL status queries not available: ' . \$e->getMessage() . PHP_EOL;
    }
    
} catch (\Exception \$e) {
    echo '✗ Database metrics error: ' . \$e->getMessage() . PHP_EOL;
}
"

# Test cache metrics
php artisan tinker --execute="
try {
    \$cacheTime = microtime(true);
    \Cache::put('test_key', 'test_value', 5);
    \$value = \Cache::get('test_key');
    \Cache::forget('test_key');
    \$responseTime = round((microtime(true) - \$cacheTime) * 1000, 2);
    
    echo '✓ Cache test: ' . (\$value === 'test_value' ? 'Working' : 'Failed') . PHP_EOL;
    echo '  Response time: ' . \$responseTime . 'ms' . PHP_EOL;
    echo '  Driver: ' . config('cache.default') . PHP_EOL;
    
} catch (\Exception \$e) {
    echo '✗ Cache metrics error: ' . \$e->getMessage() . PHP_EOL;
}
"

# Check scheduled tasks
php artisan schedule:list

# Test monitoring command manually
php artisan monitor:performance --store-metrics
```

#### Problem 6: Backup and Recovery System Issues
**Gejala:** Database backup fails atau cloud upload tidak working
**Solusi:**
```bash
# Test database backup command
echo "Testing database backup system..."

# Check mysqldump availability
if command -v mysqldump >/dev/null 2>&1; then
    echo "✓ mysqldump available"
else
    echo "✗ mysqldump not available - install mysql-client"
    # For Ubuntu/Debian: sudo apt-get install mysql-client
    # For CentOS/RHEL: sudo yum install mysql
fi

# Test backup directory creation
mkdir -p storage/app/backups
if [ -d "storage/app/backups" ]; then
    echo "✓ Backup directory exists"
    ls -la storage/app/backups/
else
    echo "✗ Could not create backup directory"
fi

# Test database connectivity for backup
php artisan tinker --execute="
\$connection = config('database.default');
\$database = config('database.connections.' . \$connection);

echo 'Database connection details:' . PHP_EOL;
echo '  Host: ' . \$database['host'] . PHP_EOL;
echo '  Port: ' . \$database['port'] . PHP_EOL;
echo '  Database: ' . \$database['database'] . PHP_EOL;
echo '  Username: ' . \$database['username'] . PHP_EOL;

try {
    \DB::connection()->getPdo();
    echo '✓ Database connection working' . PHP_EOL;
} catch (\Exception \$e) {
    echo '✗ Database connection failed: ' . \$e->getMessage() . PHP_EOL;
}
"

# Test backup command manually (without cloud upload)
php artisan backup:database --compress

# Check if backup was created
LATEST_BACKUP=$(ls -t storage/app/backups/ | head -1)
if [ -n "$LATEST_BACKUP" ]; then
    echo "✓ Backup created: $LATEST_BACKUP"
    ls -lh "storage/app/backups/$LATEST_BACKUP"
else
    echo "✗ No backup found"
fi

# Test cloud storage configuration (if using Google Cloud Storage)
php artisan tinker --execute="
try {
    \$disk = \Storage::disk('gcs');
    echo '✓ Google Cloud Storage disk configured' . PHP_EOL;
    
    // Test basic operations
    \$disk->put('test.txt', 'test content');
    \$content = \$disk->get('test.txt');
    \$disk->delete('test.txt');
    
    if (\$content === 'test content') {
        echo '✓ Google Cloud Storage operations working' . PHP_EOL;
    } else {
        echo '✗ Google Cloud Storage operations failed' . PHP_EOL;
    }
    
} catch (\Exception \$e) {
    echo '⚠ Google Cloud Storage not configured: ' . \$e->getMessage() . PHP_EOL;
}
"

# Check scheduled backup
grep -n "backup:database" app/Console/Kernel.php
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

#### ✅ Production Environment Setup
- [ ] Production environment configuration (.env.production) dengan secure settings
- [ ] OptimizationServiceProvider implemented dengan caching dan view optimizations
- [ ] Database optimization migration dengan proper indexes untuk all tables
- [ ] ProductionMetricsService implemented dengan comprehensive monitoring
- [ ] All Laravel optimizations (config:cache, route:cache, view:cache) applied

#### ✅ Google Cloud Platform Deployment
- [ ] Google Cloud project setup dengan all required APIs enabled
- [ ] Cloud SQL MySQL instance configured dengan proper security settings
- [ ] Cloud Storage bucket setup untuk file storage dengan appropriate permissions
- [ ] Redis instance configured untuk caching dan session management
- [ ] Service account created dengan minimal required permissions

#### ✅ Docker Configuration
- [ ] Multi-stage Dockerfile optimized untuk production deployment
- [ ] Nginx configuration dengan security headers dan performance optimizations
- [ ] PHP-FPM configuration dengan OPcache enabled untuk maximum performance
- [ ] Supervisor configuration untuk process management (PHP-FPM, Nginx, Queue, Scheduler)
- [ ] All configuration files properly organized dalam docker/ directory

#### ✅ CI/CD Pipeline Implementation
- [ ] GitHub Actions workflow dengan comprehensive testing pipeline
- [ ] Automated security scanning dengan vulnerability checks
- [ ] Docker image building dan pushing ke Google Container Registry
- [ ] Cloud Run deployment dengan proper environment variables dan resource limits
- [ ] Database migration automation dan cache clearing pada deployment
- [ ] Performance testing integration dalam deployment pipeline

#### ✅ Health Monitoring System
- [ ] HealthController implemented dengan multiple health check endpoints
- [ ] Basic health check (/health) untuk load balancer health checks
- [ ] Detailed health check (/health/detailed) dengan dependency verification
- [ ] Readiness dan liveness probes untuk container orchestration
- [ ] Metrics endpoint (/metrics) dengan comprehensive application metrics
- [ ] All health check routes properly registered dan tested

#### ✅ Security Hardening
- [ ] SecurityHeaders middleware implemented dengan comprehensive security headers
- [ ] Content Security Policy configured dengan proper directives
- [ ] HSTS, X-Frame-Options, dan other security headers enabled
- [ ] Rate limiting configured untuk API dan login endpoints
- [ ] Middleware properly registered dalam Kernel.php

#### ✅ Automated Backup & Monitoring
- [ ] BackupDatabase command dengan compression dan cloud upload capabilities
- [ ] MonitorPerformance command dengan health alerts dan metrics storage
- [ ] Automated scheduling dalam Console/Kernel.php untuk all maintenance tasks
- [ ] Database maintenance, cache clearing, dan log cleanup scheduled
- [ ] Performance monitoring dengan alerting system implemented

**Format Submission:**
```bash
cd ~/praktikum-cc
mkdir -p submission/week8/{docker,services,controllers,middleware,commands,config,deployment}

# Copy Docker configuration
cp Dockerfile submission/week8/
cp -r docker/ submission/week8/docker/

# Copy services dan controllers
cp app/Services/ProductionMetricsService.php submission/week8/services/
cp app/Http/Controllers/HealthController.php submission/week8/controllers/

# Copy middleware dan providers
cp app/Http/Middleware/SecurityHeaders.php submission/week8/middleware/
cp app/Providers/OptimizationServiceProvider.php submission/week8/services/

# Copy console commands
cp app/Console/Commands/BackupDatabase.php submission/week8/commands/
cp app/Console/Commands/MonitorPerformance.php submission/week8/commands/
cp app/Console/Kernel.php submission/week8/commands/

# Copy configuration files
cp .env.production submission/week8/config/env-production.example
cp -r .github/ submission/week8/deployment/

# Copy database migration
cp database/migrations/*optimize_database_for_production* submission/week8/config/ 2>/dev/null || echo "Migration file not found"

# Copy updated routes
cp routes/web.php submission/week8/config/routes-web.php

# Export application structure untuk documentation
find app/ -name "*.php" | grep -E "(Provider|Service|Controller|Command|Middleware)" > submission/week8/app-structure.txt

# Create comprehensive documentation
cat > submission/week8/README.md << 'EOF'
# Week 8: Production Deployment & Optimization

## Implementation Summary
- [x] Production environment preparation dengan comprehensive optimization
- [x] Google Cloud Platform deployment setup dengan Cloud SQL, Storage, dan Redis
- [x] Docker multi-stage build dengan optimized configuration
- [x] CI/CD pipeline dengan GitHub Actions untuk automated deployment
- [x] Health monitoring system dengan multiple endpoints
- [x] Security hardening dengan comprehensive middleware
- [x] Automated backup dan performance monitoring system

## Production Optimizations
### Environment Configuration
- Production .env dengan secure database credentials dan API keys
- Cache driver configured untuk Redis dengan compression enabled
- Session management dengan secure cookies dan proper expiration
- File storage integration dengan Google Cloud Storage
- Email configuration dengan production SMTP settings

### Performance Optimizations
- **Database Optimization**: Composite indexes untuk all major query patterns
- **OPcache Configuration**: JIT compilation enabled dengan optimized memory settings
- **View Caching**: Pre-compilation frequently used views untuk faster rendering
- **Query Optimization**: Database connection pooling dan slow query monitoring
- **Asset Optimization**: Multi-stage Docker build dengan separate asset compilation

### Application Metrics
- Memory usage monitoring dengan percentage calculation
- Database performance metrics dengan connection time tracking
- Cache performance dengan hit rate calculation
- System-level metrics (load average, disk usage)
- Application uptime dan health status tracking

## Google Cloud Platform Setup
### Infrastructure Components
- **Cloud SQL**: MySQL 8.0 instance dengan automated backups
- **Cloud Storage**: File storage bucket dengan proper IAM permissions  
- **Redis**: Caching instance untuk session dan application cache
- **Cloud Run**: Serverless container deployment dengan auto-scaling
- **Secret Manager**: Secure storage untuk sensitive configuration

### Security Configuration
- Service account dengan minimal required permissions
- IAM roles properly configured untuk Cloud SQL dan Storage access
- Network security dengan private IP untuk database connections
- SSL/TLS termination dengan automatic certificate management

## Docker Deployment
### Multi-Stage Build Process
1. **Node Builder Stage**: Asset compilation dengan Vite build optimization
2. **PHP Production Stage**: Optimized PHP-FPM dengan all required extensions
3. **Final Stage**: Nginx + PHP-FPM + Supervisor dalam single container

### Configuration Management
- **Nginx**: Performance tuning dengan gzip compression dan static file caching
- **PHP-FPM**: Production settings dengan memory limits dan security configurations
- **Supervisor**: Process management untuk web server, queue workers, dan scheduler
- **OPcache**: Maximum performance configuration dengan JIT compilation

## CI/CD Pipeline
### GitHub Actions Workflow
- **Testing Stage**: Automated PHPUnit tests dengan MySQL service container
- **Security Scanning**: Vulnerability assessment dengan Snyk integration
- **Build Stage**: Docker image creation dengan caching optimization
- **Deployment Stage**: Cloud Run deployment dengan database migration
- **Performance Testing**: Load testing dengan Apache Bench integration

### Deployment Features
- Zero-downtime deployment dengan rolling updates
- Automated database migrations dengan rollback capability
- Cache warming dan optimization commands
- Health check verification before traffic routing
- Deployment notifications dengan status reporting

## Health Monitoring
### Health Check Endpoints
- **/health**: Basic liveness check untuk load balancer
- **/health/detailed**: Comprehensive dependency verification
- **/ready**: Readiness probe untuk container orchestration
- **/live**: Liveness probe dengan uptime information
- **/metrics**: Application metrics untuk monitoring systems

### Monitoring Features
- Database connectivity dengan response time measurement
- Cache functionality verification dengan performance metrics
- Storage system health checking
- Queue system status monitoring
- Memory usage dan system resource tracking

## Security Implementation
### Security Headers
- **Content Security Policy**: Comprehensive CSP dengan proper directives
- **HSTS**: HTTP Strict Transport Security untuk secure connections
- **X-Frame-Options**: Clickjacking protection
- **X-Content-Type-Options**: MIME type sniffing prevention
- **X-XSS-Protection**: Cross-site scripting protection

### Additional Security Features
- Rate limiting untuk API endpoints dan login attempts
- Server information disclosure prevention
- Secure session configuration dengan httpOnly cookies
- Input validation dan output encoding
- Permission-based access control dengan role verification

## Automated Operations
### Backup System
- **Database Backups**: Automated daily backups dengan compression
- **Cloud Upload**: Secure backup storage dalam Google Cloud Storage
- **Retention Policy**: Automated cleanup of old backup files
- **Failure Notifications**: Email alerts untuk backup failures

### Performance Monitoring
- **Metrics Collection**: Comprehensive application performance data
- **Health Alerts**: Automated notifications untuk system issues
- **Trend Analysis**: Historical data storage untuk performance tracking
- **Proactive Monitoring**: Early warning system untuk resource constraints

### Maintenance Tasks
- **Database Optimization**: Weekly table optimization dan index maintenance
- **Cache Management**: Scheduled cache clearing dan warming
- **Log Management**: Automated log rotation dan cleanup
- **Security Updates**: Token cleanup dan expired session removal

## Testing Results
- ✓ All production optimizations implemented dan functional
- ✓ Docker multi-stage build working dengan optimal image size
- ✓ Google Cloud deployment successful dengan all services connected
- ✓ CI/CD pipeline executing with all stages passing
- ✓ Health monitoring endpoints responding correctly
- ✓ Security headers implemented dan verified
- ✓ Automated backup system tested dan working
- ✓ Performance monitoring collecting accurate metrics
- ✓ Load testing showing improved response times
- ✓ Database optimizations providing measurable performance gains

## Performance Metrics
- **Application Load Time**: < 500ms untuk cached pages
- **Database Query Time**: < 100ms untuk optimized queries  
- **Memory Usage**: < 256MB under normal load
- **Cache Hit Rate**: > 85% untuk application cache
- **Docker Image Size**: < 500MB final optimized image

## Configuration Files
- Production environment template dengan all required variables
- Docker configuration files untuk all services
- GitHub Actions workflow dengan comprehensive pipeline
- Database migration dengan performance optimizations
- Health check routes dengan proper middleware protection
- Security middleware dengan production-ready headers

## Next Steps
- Monitor production metrics dan adjust resource allocation
- Implement advanced monitoring dengan external services (New Relic, Datadog)
- Set up CDN untuk static asset delivery
- Implement database read replicas untuk scalability
- Advanced security features (WAF, DDoS protection)
- Cost optimization analysis dan resource right-sizing
EOF

# Run comprehensive production readiness testing
echo "Running comprehensive production readiness tests..."

# Test 1: Production Environment Configuration
echo "=== Testing Production Environment ==="
if [ -f ".env.production" ]; then
    echo "✓ Production environment file exists"
    
    # Check required production settings
    REQUIRED_SETTINGS=("APP_ENV=production" "APP_DEBUG=false" "CACHE_DRIVER" "SESSION_DRIVER" "MAIL_MAILER")
    for setting in "${REQUIRED_SETTINGS[@]}"; do
        if grep -q "$setting" .env.production; then
            echo "✓ $setting configured"
        else
            echo "⚠ $setting missing or not configured"
        fi
    done
else
    echo "✗ Production environment file missing"
fi

# Test 2: Docker Configuration
echo "=== Testing Docker Configuration ==="
if [ -f "Dockerfile" ] && [ -d "docker" ]; then
    echo "✓ Docker configuration complete"
    
    # Test Docker build (if Docker is available)
    if command -v docker >/dev/null 2>&1; then
        echo "Testing Docker build..."
        if docker build -t laravel-test:latest . >/dev/null 2>&1; then
            echo "✓ Docker build successful"
            docker rmi laravel-test:latest >/dev/null 2>&1
        else
            echo "⚠ Docker build failed (check Dockerfile configuration)"
        fi
    else
        echo "⚠ Docker not available for testing"
    fi
else
    echo "✗ Docker configuration incomplete"
fi

# Test 3: Application Performance
echo "=== Testing Application Performance ==="
php artisan config:cache >/dev/null 2>&1
php artisan route:cache >/dev/null 2>&1
php artisan view:cache >/dev/null 2>&1

# Test optimized application performance
START_TIME=$(php -r "echo microtime(true);")
php artisan tinker --execute="
\$users = \App\Models\User::take(5)->get();
\$posts = \App\Models\Post::with('user', 'category')->take(10)->get();
echo 'Performance test completed' . PHP_EOL;
" >/dev/null 2>&1
END_TIME=$(php -r "echo microtime(true);")
EXECUTION_TIME=$(php -r "echo round($END_TIME - $START_TIME, 3);")

echo "Query execution time: ${EXECUTION_TIME}s"
if (( $(echo "$EXECUTION_TIME < 1.0" | bc -l) )); then
    echo "✓ Good application performance"
else
    echo "⚠ Application performance may need optimization"
fi

# Test 4: Security Configuration
echo "=== Testing Security Configuration ==="
if [ -f "app/Http/Middleware/SecurityHeaders.php" ]; then
    echo "✓ Security headers middleware exists"
    
    if grep -q "SecurityHeaders" app/Http/Kernel.php; then
        echo "✓ Security middleware registered"
    else
        echo "⚠ Security middleware not registered"
    fi
else
    echo "✗ Security headers middleware missing"
fi

# Test 5: Health Monitoring
echo "=== Testing Health Monitoring ==="
if [ -f "app/Http/Controllers/HealthController.php" ]; then
    echo "✓ Health controller exists"
    
    # Test health endpoints registration
    if grep -q "/health" routes/web.php; then
        echo "✓ Health routes registered"
    else
        echo "⚠ Health routes not registered"
    fi
else
    echo "✗ Health controller missing"
fi

# Test 6: Automated Operations
echo "=== Testing Automated Operations ==="
COMMANDS=("BackupDatabase" "MonitorPerformance")
for cmd in "${COMMANDS[@]}"; do
    if [ -f "app/Console/Commands/${cmd}.php" ]; then
        echo "✓ $cmd command exists"
    else
        echo "✗ $cmd command missing"
    fi
done

if grep -q "backup:database" app/Console/Kernel.php; then
    echo "✓ Backup scheduling configured"
else
    echo "⚠ Backup scheduling not configured"
fi

# Test 7: CI/CD Pipeline
echo "=== Testing CI/CD Pipeline ==="
if [ -f ".github/workflows/deploy-production.yml" ]; then
    echo "✓ GitHub Actions workflow exists"
    
    # Check for required jobs
    REQUIRED_JOBS=("test" "security-scan" "deploy" "performance-test")
    for job in "${REQUIRED_JOBS[@]}"; do
        if grep -q "$job:" .github/workflows/deploy-production.yml; then
            echo "✓ Job $job configured"
        else
            echo "⚠ Job $job missing"
        fi
    done
else
    echo "✗ GitHub Actions workflow missing"
fi

# Generate testing report
cat > submission/week8/production-readiness-report.txt << EOF
Production Readiness Test Report
Generated: $(date)

Environment Configuration: $([ -f ".env.production" ] && echo "✓ PASS" || echo "✗ FAIL")
Docker Configuration: $([ -f "Dockerfile" ] && [ -d "docker" ] && echo "✓ PASS" || echo "✗ FAIL")
Application Performance: $([ $(echo "$EXECUTION_TIME < 1.0" | bc -l) -eq 1 ] && echo "✓ PASS" || echo "⚠ REVIEW")
Security Configuration: $([ -f "app/Http/Middleware/SecurityHeaders.php" ] && echo "✓ PASS" || echo "✗ FAIL")
Health Monitoring: $([ -f "app/Http/Controllers/HealthController.php" ] && echo "✓ PASS" || echo "✗ FAIL")
Automated Operations: $([ -f "app/Console/Commands/BackupDatabase.php" ] && echo "✓ PASS" || echo "✗ FAIL")
CI/CD Pipeline: $([ -f ".github/workflows/deploy-production.yml" ] && echo "✓ PASS" || echo "✗ FAIL")

Performance Metrics:
- Query Execution Time: ${EXECUTION_TIME}s
- Memory Usage: $(php -r "echo round(memory_get_peak_usage(true) / 1024 / 1024, 2);")MB
- Configuration Cache: $([ -f "bootstrap/cache/config.php" ] && echo "Enabled" || echo "Disabled")
- Route Cache: $([ -f "bootstrap/cache/routes-v7.php" ] && echo "Enabled" || echo "Disabled")
- View Cache: $([ -d "storage/framework/views" ] && [ "$(ls -A storage/framework/views)" ] && echo "Enabled" || echo "Disabled")

Recommendations:
- Monitor application performance dalam production environment
- Set up external monitoring services untuk comprehensive coverage
- Implement CDN untuk static asset delivery
- Configure database read replicas untuk high availability
- Set up automated alerts untuk critical system events
EOF

# Commit and push
git add submission/week8/
git commit -m "Week 8: Production Deployment & Optimization - [Nama]"
git push origin main

echo "Week 8 submission completed successfully!"
echo "Production readiness report: submission/week8/production-readiness-report.txt"
echo "Docker deployment ready: Use 'docker build -t laravel-app .' to build"
echo "GitHub Actions pipeline ready: Push to main branch to trigger deployment"
echo "Health check endpoints: /health, /health/detailed, /ready, /live, /metrics"
```