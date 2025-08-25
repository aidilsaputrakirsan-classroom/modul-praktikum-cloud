# WEEK 5: CRUD Operations - Update & Delete
## Praktikum Cloud Computing - Institut Teknologi Kalimantan

### 📋 INFORMASI SESI
- **Week**: 5
- **Durasi**: 100 menit  
- **Topik**: Advanced CRUD Operations - Update & Delete dengan Relationship Handling
- **Target**: Mahasiswa Semester 6
- **Platform**: Laptop/PC (Windows, macOS, atau Linux)

### 🎯 TUJUAN PEMBELAJARAN
Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:
1. Mengimplementasikan Update operations dengan file replacement dan validation
2. Menangani Delete operations dengan soft delete dan cascade relationships
3. Membangun bulk operations untuk multiple record management
4. Mengoptimalkan form handling dengan AJAX dan real-time validation
5. Implementasi advanced relationship updates (many-to-many, nested resources)
6. Menerapkan transaction handling untuk data integrity
7. Membangun user interface yang responsive untuk update/delete operations

### 📚 PERSIAPAN
**Prerequisites yang harus dipenuhi:**
- Week 1-4 telah completed dengan semua functionality working
- CRUD Create & Read operations sudah functional
- Database relationships sudah proper dengan foreign keys
- Authentication system basic sudah setup (minimal user management)

**Environment Verification:**
```bash
# Pastikan berada di direktori project Laravel
cd laravel-app

# Masuk ke tinker
php artisan tinker

# Verifikasi database connection dan existing data
echo 'Database Connection: ' . (\DB::connection()->getPdo() ? 'OK' : 'Failed') . PHP_EOL;
echo 'Posts count: ' . \App\Models\Post::count() . PHP_EOL;
echo 'Categories count: ' . \App\Models\Category::count() . PHP_EOL;
echo 'Tags count: ' . \App\Models\Tag::count() . PHP_EOL;
echo 'Users count: ' . \App\Models\User::count() . PHP_EOL;

# Verifikasi routes dan controllers dari week 4
php artisan route:list | grep -E "(posts|categories|tags)" | head -5
```

### 🛠️ LANGKAH PRAKTIKUM

#### **Bagian 1: Enhanced Update Operations (25 menit)**

##### Step 1.1: Upgrade Post Update Controller Method

**Update method `update` dalam PostController.php:**
```php
/**
 * Update existing post dengan advanced file handling dan relationship updates
 */
public function update(Request $request, Post $post)
{
    // Enhanced validation rules dengan conditional validation
    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'content' => 'required|string|min:10',
        'excerpt' => 'nullable|string|max:500',
        'featured_image' => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
        'remove_featured_image' => 'boolean', // Checkbox untuk remove existing image
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
        'published_at' => 'nullable|date',
        'schedule_publish' => 'boolean', // Checkbox untuk scheduled publishing
    ]);

    // Start database transaction untuk data integrity
    \DB::beginTransaction();
    
    try {
        // Store original data untuk rollback jika diperlukan
        $originalData = $post->toArray();
        
        // Update slug jika title berubah dan ensure uniqueness
        if ($post->title !== $validated['title']) {
            $validated['slug'] = $this->generateUniqueSlug($validated['title'], $post->id);
        }

        // Handle featured image operations
        if ($request->boolean('remove_featured_image')) {
            // Remove existing featured image
            if ($post->featured_image) {
                \Storage::disk('public')->delete($post->featured_image);
                $validated['featured_image'] = null;
            }
        } elseif ($request->hasFile('featured_image')) {
            // Replace existing featured image dengan new upload
            if ($post->featured_image) {
                \Storage::disk('public')->delete($post->featured_image);
            }
            
            // Store new featured image dengan organized directory structure
            $file = $request->file('featured_image');
            $filename = time() . '_' . \Str::slug($validated['title']) . '.' . $file->getClientOriginalExtension();
            $validated['featured_image'] = $file->storeAs('posts/featured-images', $filename, 'public');
        }

        // Handle publishing logic dengan scheduled publishing
        if ($request->boolean('schedule_publish') && $request->filled('published_at')) {
            // Scheduled publishing - set status dan published_at
            $validated['published_at'] = \Carbon\Carbon::parse($validated['published_at']);
            if ($validated['published_at']->isFuture()) {
                $validated['status'] = 'scheduled'; // Custom status untuk scheduled posts
            }
        } elseif ($validated['status'] === 'published' && $post->status !== 'published') {
            // Immediate publishing
            $validated['published_at'] = now();
        } elseif ($validated['status'] !== 'published') {
            // If changing from published to draft/archived, keep original published_at
            unset($validated['published_at']);
        }

        // Update post record
        $post->update($validated);

        // Handle many-to-many relationship updates (tags)
        if (array_key_exists('tags', $validated)) {
            if (!empty($validated['tags'])) {
                // Sync tags dengan timestamp tracking
                $post->tags()->sync($validated['tags'], true);
            } else {
                // Remove all tags jika empty array
                $post->tags()->detach();
            }
        }

        // Log activity untuk audit trail
        \Log::info('Post updated', [
            'post_id' => $post->id,
            'user_id' => auth()->id() ?? $validated['user_id'],
            'changes' => $post->getChanges(),
            'timestamp' => now()
        ]);

        // Commit transaction
        \DB::commit();

        // Flash success message dengan detailed information
        $message = "Post '{$post->title}' successfully updated";
        if ($validated['status'] === 'published' && $originalData['status'] !== 'published') {
            $message .= " and published";
        }

        return redirect()->route('posts.index')
                        ->with('success', $message);

    } catch (\Exception $e) {
        // Rollback transaction on error
        \DB::rollback();
        
        \Log::error('Post update failed', [
            'post_id' => $post->id,
            'error' => $e->getMessage(),
            'timestamp' => now()
        ]);

        return redirect()->back()
                        ->withInput()
                        ->with('error', 'Failed to update post: ' . $e->getMessage());
    }
}
```

##### Step 1.2: Enhanced Edit Form dengan Advanced Features

**Isi file resources/views/posts/edit.blade.php:**
```html
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

#### **Bagian 2: Advanced Delete Operations (20 menit)**

##### Step 2.1: Enhanced Delete Method dengan Cascade Handling

**Update method `destroy` dalam PostController.php:**
```php
/**
 * Delete post dengan advanced cascade handling dan cleanup
 */
public function destroy(Post $post)
{
    // Authorization check (dalam praktik nyata)
    // $this->authorize('delete', $post);
    
    // Start database transaction
    \DB::beginTransaction();
    
    try {
        $title = $post->title;
        $postId = $post->id;
        
        // Store data untuk logging
        $deletionData = [
            'post_id' => $postId,
            'title' => $title,
            'user_id' => $post->user_id,
            'category_id' => $post->category_id,
            'featured_image' => $post->featured_image,
            'tags_count' => $post->tags()->count(),
            'comments_count' => $post->comments()->count(),
            'deleted_by' => auth()->id() ?? 'system',
            'deleted_at' => now()
        ];

        // Handle cascade relationships dan cleanup
        
        // 1. Handle Comments (soft delete to preserve data integrity)
        if ($post->comments()->exists()) {
            $post->comments()->delete(); // Soft delete if using SoftDeletes
        }
        
        // 2. Detach Many-to-Many relationships (tags)
        $post->tags()->detach();
        
        // 3. Handle Media files yang terkait dengan post
        $post->media()->each(function ($media) {
            // Delete physical files
            if (\Storage::disk('public')->exists($media->path)) {
                \Storage::disk('public')->delete($media->path);
            }
            // Delete media record
            $media->delete();
        });
        
        // 4. Delete featured image file
        if ($post->featured_image) {
            \Storage::disk('public')->delete($post->featured_image);
        }
        
        // 5. Log deletion untuk audit trail sebelum delete
        \Log::info('Post deletion initiated', $deletionData);
        
        // 6. Finally delete the post (soft delete)
        $post->delete();
        
        // Commit transaction
        \DB::commit();
        
        // Flash success message
        return redirect()->route('posts.index')
                        ->with('success', "Post '{$title}' and all related data successfully deleted");

    } catch (\Exception $e) {
        // Rollback transaction
        \DB::rollback();
        
        \Log::error('Post deletion failed', [
            'post_id' => $post->id,
            'error' => $e->getMessage(),
            'stack_trace' => $e->getTraceAsString(),
            'timestamp' => now()
        ]);

        return redirect()->route('posts.index')
                        ->with('error', 'Failed to delete post: ' . $e->getMessage());
    }
}
```

##### Step 2.2: Bulk Delete Operations

**Tambahkan methods baru di PostController:**
```php
/**
 * Bulk delete multiple posts
 */
public function bulkDelete(Request $request)
{
    $validated = $request->validate([
        'post_ids' => 'required|array|min:1',
        'post_ids.*' => 'exists:posts,id',
        'confirm_deletion' => 'required|accepted'
    ]);

    \DB::beginTransaction();
    
    try {
        $posts = Post::whereIn('id', $validated['post_ids'])->get();
        $deletedCount = 0;
        $errors = [];

        foreach ($posts as $post) {
            try {
                // Use existing destroy logic
                $this->performPostDeletion($post);
                $deletedCount++;
            } catch (\Exception $e) {
                $errors[] = "Failed to delete '{$post->title}': " . $e->getMessage();
            }
        }

        \DB::commit();

        if ($deletedCount > 0) {
            $message = "Successfully deleted {$deletedCount} post(s)";
            if (!empty($errors)) {
                $message .= ". Some deletions failed: " . implode(', ', $errors);
            }
            return response()->json(['success' => true, 'message' => $message]);
        } else {
            return response()->json(['success' => false, 'message' => 'No posts were deleted']);
        }

    } catch (\Exception $e) {
        \DB::rollback();
        return response()->json(['success' => false, 'message' => 'Bulk deletion failed: ' . $e->getMessage()]);
    }
}

/**
 * Bulk update status untuk multiple posts
 */
public function bulkUpdateStatus(Request $request)
{
    $validated = $request->validate([
        'post_ids' => 'required|array|min:1',
        'post_ids.*' => 'exists:posts,id',
        'status' => 'required|in:draft,published,archived'
    ]);

    try {
        $updated = Post::whereIn('id', $validated['post_ids'])
                      ->update([
                          'status' => $validated['status'],
                          'updated_at' => now(),
                          // Set published_at jika status published
                          'published_at' => $validated['status'] === 'published' ? now() : null
                      ]);

        \Log::info('Bulk status update', [
            'post_ids' => $validated['post_ids'],
            'new_status' => $validated['status'],
            'updated_count' => $updated,
            'performed_by' => auth()->id() ?? 'system'
        ]);

        return response()->json([
            'success' => true,
            'message' => "Successfully updated {$updated} post(s) to {$validated['status']} status"
        ]);

    } catch (\Exception $e) {
        return response()->json([
            'success' => false,
            'message' => 'Bulk update failed: ' . $e->getMessage()
        ]);
    }
}

/**
 * Helper method untuk post deletion logic
 */
private function performPostDeletion(Post $post)
{
    // Handle comments
    $post->comments()->delete();
    
    // Detach tags
    $post->tags()->detach();
    
    // Delete media files
    $post->media()->each(function ($media) {
        if (\Storage::disk('public')->exists($media->path)) {
            \Storage::disk('public')->delete($media->path);
        }
        $media->delete();
    });
    
    // Delete featured image
    if ($post->featured_image) {
        \Storage::disk('public')->delete($post->featured_image);
    }
    
    // Delete post
    $post->delete();
}
```

##### Step 2.3: Enhanced Posts Index dengan Bulk Operations

**Tambahkan bulk operations UI ke posts/index.blade.php:**
```html
<!-- Tambahkan setelah Statistics Cards, sebelum Filters and Actions -->

<!-- Bulk Actions Bar (hidden by default) -->
<div class="card hidden" id="bulkActionsBar">
    <div class="flex items-center justify-between">
        <div class="flex items-center space-x-4">
            <span class="text-sm font-medium text-gray-700">
                <span id="selectedCount">0</span> post(s) selected
            </span>
            
            <div class="flex items-center space-x-2">
                <select id="bulkAction" class="input-field text-sm">
                    <option value="">Choose Action</option>
                    <option value="publish">Publish Selected</option>
                    <option value="draft">Move to Draft</option>
                    <option value="archive">Archive Selected</option>
                    <option value="delete">Delete Selected</option>
                </select>
                
                <button type="button" 
                        class="btn-primary text-sm"
                        onclick="performBulkAction()">
                    Apply
                </button>
            </div>
        </div>
        
        <button type="button" 
                class="text-gray-400 hover:text-gray-600"
                onclick="clearSelection()">
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
        </button>
    </div>
</div>

<!-- Update Posts Table dengan bulk selection -->
<!-- Dalam thead, tambahkan checkbox column di awal -->
<th class="px-6 py-3 text-left">
    <input type="checkbox" 
           id="selectAll"
           class="rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
           onchange="toggleSelectAll()">
</th>

<!-- Dalam tbody, tambahkan checkbox untuk setiap row -->
@forelse($posts as $post)
    <tr class="hover:bg-gray-50" id="post-row-{{ $post->id }}">
        <td class="px-6 py-4">
            <input type="checkbox" 
                   name="selected_posts[]"
                   value="{{ $post->id }}"
                   class="post-checkbox rounded border-gray-300 text-primary-600 shadow-sm focus:border-primary-300 focus:ring focus:ring-primary-200 focus:ring-opacity-50"
                   onchange="updateBulkSelection()">
        </td>
        <!-- Rest of existing table columns -->
        <!-- ... -->
    </tr>
@empty
    <!-- ... existing empty state -->
@endforelse

<!-- Tambahkan JavaScript untuk bulk operations di akhir file -->
<script>
// Bulk operations JavaScript
let selectedPosts = [];

function toggleSelectAll() {
    const selectAllCheckbox = document.getElementById('selectAll');
    const postCheckboxes = document.querySelectorAll('.post-checkbox');
    
    postCheckboxes.forEach(checkbox => {
        checkbox.checked = selectAllCheckbox.checked;
    });
    
    updateBulkSelection();
}

function updateBulkSelection() {
    const checkboxes = document.querySelectorAll('.post-checkbox:checked');
    const bulkBar = document.getElementById('bulkActionsBar');
    const selectedCount = document.getElementById('selectedCount');
    
    selectedPosts = Array.from(checkboxes).map(cb => cb.value);
    
    if (selectedPosts.length > 0) {
        bulkBar.classList.remove('hidden');
        selectedCount.textContent = selectedPosts.length;
    } else {
        bulkBar.classList.add('hidden');
    }
    
    // Update select all checkbox state
    const allCheckboxes = document.querySelectorAll('.post-checkbox');
    const selectAllCheckbox = document.getElementById('selectAll');
    
    if (selectedPosts.length === allCheckboxes.length) {
        selectAllCheckbox.checked = true;
        selectAllCheckbox.indeterminate = false;
    } else if (selectedPosts.length > 0) {
        selectAllCheckbox.checked = false;
        selectAllCheckbox.indeterminate = true;
    } else {
        selectAllCheckbox.checked = false;
        selectAllCheckbox.indeterminate = false;
    }
}

function clearSelection() {
    document.querySelectorAll('.post-checkbox').forEach(cb => cb.checked = false);
    document.getElementById('selectAll').checked = false;
    updateBulkSelection();
}

function performBulkAction() {
    const action = document.getElementById('bulkAction').value;
    
    if (!action) {
        alert('Please select an action');
        return;
    }
    
    if (selectedPosts.length === 0) {
        alert('Please select posts to perform action on');
        return;
    }
    
    const actionText = document.getElementById('bulkAction').selectedOptions[0].text;
    
    if (action === 'delete') {
        if (!confirm(`Are you sure you want to delete ${selectedPosts.length} selected post(s)? This action cannot be undone.`)) {
            return;
        }
        
        performBulkDelete();
    } else {
        if (!confirm(`Are you sure you want to ${actionText.toLowerCase()} ${selectedPosts.length} selected post(s)?`)) {
            return;
        }
        
        performBulkStatusUpdate(action);
    }
}

function performBulkDelete() {
    fetch('/posts/bulk-delete', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').getAttribute('content'),
            'Accept': 'application/json',
        },
        body: JSON.stringify({
            post_ids: selectedPosts,
            confirm_deletion: true
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            showNotification(data.message, 'success');
            // Remove deleted rows from table
            selectedPosts.forEach(postId => {
                const row = document.getElementById(`post-row-${postId}`);
                if (row) row.remove();
            });
            clearSelection();
        } else {
            showNotification(data.message, 'error');
        }
    })
    .catch(error => {
        console.error('Error:', error);
        showNotification('An error occurred during bulk deletion', 'error');
    });
}

function performBulkStatusUpdate(status) {
    fetch('/posts/bulk-update-status', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').getAttribute('content'),
            'Accept': 'application/json',
        },
        body: JSON.stringify({
            post_ids: selectedPosts,
            status: status
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            showNotification(data.message, 'success');
            // Refresh page to show updated statuses
            location.reload();
        } else {
            showNotification(data.message, 'error');
        }
    })
    .catch(error => {
        console.error('Error:', error);
        showNotification('An error occurred during bulk update', 'error');
    });
}

function showNotification(message, type) {
    // Create notification element
    const notification = document.createElement('div');
    notification.className = `fixed top-4 right-4 p-4 rounded-lg shadow-lg z-50 ${
        type === 'success' ? 'bg-green-50 border border-green-200 text-green-800' : 'bg-red-50 border border-red-200 text-red-800'
    }`;
    notification.innerHTML = `
        <div class="flex items-center">
            <span>${message}</span>
            <button onclick="this.parentElement.parentElement.remove()" class="ml-4 text-sm underline">Close</button>
        </div>
    `;
    
    document.body.appendChild(notification);
    
    // Auto remove after 5 seconds
    setTimeout(() => {
        if (notification.parentElement) {
            notification.remove();
        }
    }, 5000);
}

// Enhanced delete single post function
function deletePost(postId) {
    if (confirm('Are you sure you want to delete this post? This action cannot be undone.')) {
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
                showNotification(data.message || 'Post deleted successfully', 'success');
                // Remove row from table
                const row = document.getElementById(`post-row-${postId}`);
                if (row) row.remove();
            } else {
                showNotification(data.message || 'Error deleting post', 'error');
            }
        })
        .catch(error => {
            console.error('Error:', error);
            showNotification('Error deleting post', 'error');
        });
    }
}
</script>
```

#### **Bagian 3: Advanced Routes dan API Enhancements (15 menit)**

##### Step 3.1: Add Bulk Operation Routes

**Tambahkan routes untuk bulk operations di routes/web.php:**
```php
// Tambahkan setelah existing posts routes

/*
|--------------------------------------------------------------------------
| Bulk Operations Routes
|--------------------------------------------------------------------------
*/

// Bulk operations untuk posts
Route::prefix('posts')->name('posts.')->group(function () {
    Route::post('bulk-delete', [PostController::class, 'bulkDelete'])->name('bulk-delete');
    Route::post('bulk-update-status', [PostController::class, 'bulkUpdateStatus'])->name('bulk-update-status');
    Route::post('bulk-update-category', [PostController::class, 'bulkUpdateCategory'])->name('bulk-update-category');
});

// Restore deleted posts (soft delete recovery)
Route::prefix('admin')->name('admin.')->group(function () {
    Route::get('deleted-posts', [PostController::class, 'deletedPosts'])->name('deleted-posts');
    Route::post('posts/{id}/restore', [PostController::class, 'restore'])->name('posts.restore');
    Route::delete('posts/{id}/force-delete', [PostController::class, 'forceDelete'])->name('posts.force-delete');
});
```

##### Step 3.2: Enhanced API Routes dengan Advanced Features

**Tambahkan advanced API routes di routes/api.php:**
```php
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
    Route::get('{post}/analytics', [PostApiController::class, 'analytics'])->name('analytics');
    
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
```

#### **Bagian 4: Transaction Handling dan Error Recovery (20 menit)**

##### Step 4.1: Advanced Transaction Service Class
```bash
# Buat service class untuk transaction handling
mkdir -p app/Services
```

**Isi file app/Services/PostService.php:**
```php
<?php

namespace App\Services;

use App\Models\Post;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;

class PostService
{
    /**
     * Create post dengan transaction handling
     */
    public function createPost(array $data): array
    {
        DB::beginTransaction();
        
        try {
            // Generate slug
            $data['slug'] = $this->generateUniqueSlug($data['title']);
            
            // Handle file upload jika ada
            if (isset($data['featured_image']) && $data['featured_image']) {
                $data['featured_image'] = $this->handleFileUpload($data['featured_image'], $data['title']);
            }
            
            // Set published_at jika status published
            if ($data['status'] === 'published') {
                $data['published_at'] = now();
            }
            
            // Create post
            $post = Post::create($data);
            
            // Handle tags relationship
            if (!empty($data['tags'])) {
                $post->tags()->attach($data['tags']);
            }
            
            // Log activity
            Log::info('Post created successfully', [
                'post_id' => $post->id,
                'title' => $post->title,
                'user_id' => $data['user_id'] ?? null,
                'timestamp' => now()
            ]);
            
            DB::commit();
            
            return [
                'success' => true,
                'post' => $post,
                'message' => "Post '{$post->title}' created successfully"
            ];
            
        } catch (\Exception $e) {
            DB::rollback();
            
            // Cleanup uploaded file jika ada error
            if (isset($data['featured_image']) && is_string($data['featured_image'])) {
                Storage::disk('public')->delete($data['featured_image']);
            }
            
            Log::error('Post creation failed', [
                'error' => $e->getMessage(),
                'data' => $data,
                'timestamp' => now()
            ]);
            
            return [
                'success' => false,
                'message' => 'Failed to create post: ' . $e->getMessage()
            ];
        }
    }
    
    /**
     * Update post dengan transaction handling
     */
    public function updatePost(Post $post, array $data): array
    {
        DB::beginTransaction();
        
        try {
            $originalData = $post->toArray();
            
            // Update slug jika title berubah
            if ($post->title !== $data['title']) {
                $data['slug'] = $this->generateUniqueSlug($data['title'], $post->id);
            }
            
            // Handle featured image operations
            $data = $this->handleImageUpdate($post, $data);
            
            // Handle publishing logic
            $data = $this->handlePublishingLogic($post, $data);
            
            // Update post
            $post->update($data);
            
            // Handle tags relationship
            if (array_key_exists('tags', $data)) {
                if (!empty($data['tags'])) {
                    $post->tags()->sync($data['tags'], true);
                } else {
                    $post->tags()->detach();
                }
            }
            
            // Log changes
            Log::info('Post updated successfully', [
                'post_id' => $post->id,
                'changes' => $post->getChanges(),
                'original_data' => $originalData,
                'timestamp' => now()
            ]);
            
            DB::commit();
            
            return [
                'success' => true,
                'post' => $post->fresh(),
                'message' => "Post '{$post->title}' updated successfully"
            ];
            
        } catch (\Exception $e) {
            DB::rollback();
            
            Log::error('Post update failed', [
                'post_id' => $post->id,
                'error' => $e->getMessage(),
                'data' => $data,
                'timestamp' => now()
            ]);
            
            return [
                'success' => false,
                'message' => 'Failed to update post: ' . $e->getMessage()
            ];
        }
    }
    
    /**
     * Delete post dengan cleanup transaction
     */
    public function deletePost(Post $post): array
    {
        DB::beginTransaction();
        
        try {
            $title = $post->title;
            $postId = $post->id;
            
            // Prepare deletion data untuk logging
            $deletionData = [
                'post_id' => $postId,
                'title' => $title,
                'featured_image' => $post->featured_image,
                'tags_count' => $post->tags()->count(),
                'comments_count' => $post->comments()->count(),
                'deleted_at' => now()
            ];
            
            // Handle cascade deletions
            $this->handleCascadeDeletion($post);
            
            // Delete the post
            $post->delete();
            
            Log::info('Post deleted successfully', $deletionData);
            
            DB::commit();
            
            return [
                'success' => true,
                'message' => "Post '{$title}' deleted successfully"
            ];
            
        } catch (\Exception $e) {
            DB::rollback();
            
            Log::error('Post deletion failed', [
                'post_id' => $post->id,
                'error' => $e->getMessage(),
                'timestamp' => now()
            ]);
            
            return [
                'success' => false,
                'message' => 'Failed to delete post: ' . $e->getMessage()
            ];
        }
    }
    
    /**
     * Bulk operations dengan transaction
     */
    public function bulkUpdateStatus(array $postIds, string $status): array
    {
        DB::beginTransaction();
        
        try {
            $posts = Post::whereIn('id', $postIds)->get();
            $updateData = [
                'status' => $status,
                'updated_at' => now()
            ];
            
            // Set published_at jika status published
            if ($status === 'published') {
                $updateData['published_at'] = now();
            }
            
            $updated = Post::whereIn('id', $postIds)->update($updateData);
            
            Log::info('Bulk status update completed', [
                'post_ids' => $postIds,
                'new_status' => $status,
                'updated_count' => $updated,
                'timestamp' => now()
            ]);
            
            DB::commit();
            
            return [
                'success' => true,
                'message' => "Successfully updated {$updated} post(s) to {$status} status",
                'updated_count' => $updated
            ];
            
        } catch (\Exception $e) {
            DB::rollback();
            
            Log::error('Bulk status update failed', [
                'post_ids' => $postIds,
                'status' => $status,
                'error' => $e->getMessage(),
                'timestamp' => now()
            ]);
            
            return [
                'success' => false,
                'message' => 'Failed to update posts: ' . $e->getMessage()
            ];
        }
    }
    
    /*
    |--------------------------------------------------------------------------
    | Helper Methods
    |--------------------------------------------------------------------------
    */
    
    private function generateUniqueSlug(string $title, ?int $excludeId = null): string
    {
        $slug = \Str::slug($title);
        $originalSlug = $slug;
        $counter = 1;
        
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
    
    private function handleFileUpload($file, string $title): string
    {
        $filename = time() . '_' . \Str::slug($title) . '.' . $file->getClientOriginalExtension();
        return $file->storeAs('posts/featured-images', $filename, 'public');
    }
    
    private function handleImageUpdate(Post $post, array $data): array
    {
        if (isset($data['remove_featured_image']) && $data['remove_featured_image']) {
            // Remove existing image
            if ($post->featured_image) {
                Storage::disk('public')->delete($post->featured_image);
                $data['featured_image'] = null;
            }
        } elseif (isset($data['featured_image']) && $data['featured_image']) {
            // Replace existing image
            if ($post->featured_image) {
                Storage::disk('public')->delete($post->featured_image);
            }
            $data['featured_image'] = $this->handleFileUpload($data['featured_image'], $data['title']);
        }
        
        return $data;
    }
    
    private function handlePublishingLogic(Post $post, array $data): array
    {
        if (isset($data['schedule_publish']) && $data['schedule_publish'] && isset($data['published_at'])) {
            $publishDate = \Carbon\Carbon::parse($data['published_at']);
            if ($publishDate->isFuture()) {
                $data['status'] = 'scheduled';
            }
        } elseif ($data['status'] === 'published' && $post->status !== 'published') {
            $data['published_at'] = now();
        }
        
        return $data;
    }
    
    private function handleCascadeDeletion(Post $post): void
    {
        // Soft delete comments
        $post->comments()->delete();
        
        // Detach tags
        $post->tags()->detach();
        
        // Delete media files
        $post->media()->each(function ($media) {
            if (Storage::disk('public')->exists($media->path)) {
                Storage::disk('public')->delete($media->path);
            }
            $media->delete();
        });
        
        // Delete featured image
        if ($post->featured_image) {
            Storage::disk('public')->delete($post->featured_image);
        }
    }
}
```

##### Step 4.2: Update Controller untuk menggunakan Service

**Update PostController methods untuk menggunakan service:**
```php
// Tambahkan di bagian atas class PostController

use App\Services\PostService;

class PostController extends Controller
{
    protected $postService;

    public function __construct(PostService $postService)
    {
        $this->postService = $postService;
    }

    /**
     * Store new post menggunakan service
     */
    public function store(Request $request)
    {
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

        $result = $this->postService->createPost($validated);

        if ($result['success']) {
            return redirect()->route('posts.index')
                            ->with('success', $result['message']);
        } else {
            return redirect()->back()
                            ->withInput()
                            ->with('error', $result['message']);
        }
    }

    /**
     * Update post menggunakan service
     */
    public function update(Request $request, Post $post)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'excerpt' => 'nullable|string|max:500',
            'featured_image' => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
            'remove_featured_image' => 'boolean',
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
            'published_at' => 'nullable|date',
            'schedule_publish' => 'boolean',
        ]);

        $result = $this->postService->updatePost($post, $validated);

        if ($result['success']) {
            return redirect()->route('posts.index')
                            ->with('success', $result['message']);
        } else {
            return redirect()->back()
                            ->withInput()
                            ->with('error', $result['message']);
        }
    }

    /**
     * Delete post menggunakan service
     */
    public function destroy(Post $post)
    {
        $result = $this->postService->deletePost($post);

        if ($result['success']) {
            return redirect()->route('posts.index')
                            ->with('success', $result['message']);
        } else {
            return redirect()->route('posts.index')
                            ->with('error', $result['message']);
        }
    }

    /**
     * Bulk update status menggunakan service
     */
    public function bulkUpdateStatus(Request $request)
    {
        $validated = $request->validate([
            'post_ids' => 'required|array|min:1',
            'post_ids.*' => 'exists:posts,id',
            'status' => 'required|in:draft,published,archived'
        ]);

        $result = $this->postService->bulkUpdateStatus(
            $validated['post_ids'], 
            $validated['status']
        );

        return response()->json($result);
    }
}
```

### 🧪 TESTING & VERIFIKASI

#### Test 1: Enhanced Update Operations Verification
```bash
echo "=== Enhanced Update Operations Testing ==="

# Test update dengan file replacement
php artisan tinker --execute="
\$post = \App\Models\Post::first();
if (\$post) {
    \$originalTitle = \$post->title;
    \$post->update(['title' => 'Updated Title Test']);
    echo 'Title update: ' . (\$post->title === 'Updated Title Test' ? 'OK' : 'Failed') . PHP_EOL;
    
    // Test slug generation
    echo 'Slug updated: ' . (\$post->slug !== \Str::slug(\$originalTitle) ? 'OK' : 'Failed') . PHP_EOL;
    
    // Restore original title
    \$post->update(['title' => \$originalTitle]);
    echo 'Title restored: OK' . PHP_EOL;
} else {
    echo 'No posts found for testing' . PHP_EOL;
}
"
```

#### Test 2: Delete Operations dengan Cascade Testing
```bash
echo "=== Delete Operations Testing ==="

php artisan tinker --execute="
// Create test post dengan relationships
\$user = \App\Models\User::first();
\$category = \App\Models\Category::first();

if (\$user && \$category) {
    \$testPost = \App\Models\Post::create([
        'title' => 'Test Delete Post',
        'content' => 'This is a test post for deletion testing',
        'slug' => 'test-delete-post-' . time(),
        'status' => 'draft',
        'user_id' => \$user->id,
        'category_id' => \$category->id,
    ]);
    
    // Add tags relationship
    \$tags = \App\Models\Tag::take(2)->pluck('id');
    if (\$tags->isNotEmpty()) {
        \$testPost->tags()->attach(\$tags);
        echo 'Tags attached: ' . \$testPost->tags()->count() . PHP_EOL;
    }
    
    // Test delete operation
    \$postId = \$testPost->id;
    \$testPost->delete();
    
    // Verify soft delete
    \$deletedPost = \App\Models\Post::withTrashed()->find(\$postId);
    echo 'Soft delete working: ' . (\$deletedPost && \$deletedPost->trashed() ? 'OK' : 'Failed') . PHP_EOL;
    
    // Verify tags detached
    echo 'Tags detached: ' . (\$deletedPost->tags()->count() === 0 ? 'OK' : 'Failed') . PHP_EOL;
    
    // Cleanup
    \$deletedPost->forceDelete();
    echo 'Test cleanup: OK' . PHP_EOL;
} else {
    echo 'Missing test data (user or category)' . PHP_EOL;
}
"
```

#### Test 3: Bulk Operations Testing
```bash
echo "=== Bulk Operations Testing ==="

# Test bulk routes registration
php artisan route:list | grep -E "(bulk|restore)" && echo "✓ Bulk routes registered"

# Test bulk update functionality
php artisan tinker --execute="
// Create multiple test posts
\$user = \App\Models\User::first();
\$category = \App\Models\Category::first();

if (\$user && \$category) {
    \$testPosts = [];
    for (\$i = 1; \$i <= 3; \$i++) {
        \$testPosts[] = \App\Models\Post::create([
            'title' => 'Bulk Test Post ' . \$i,
            'content' => 'Content for bulk test post ' . \$i,
            'slug' => 'bulk-test-post-' . \$i . '-' . time(),
            'status' => 'draft',
            'user_id' => \$user->id,
            'category_id' => \$category->id,
        ]);
    }
    
    \$postIds = array_map(fn(\$post) => \$post->id, \$testPosts);
    
    // Test bulk status update
    \$updated = \App\Models\Post::whereIn('id', \$postIds)
                                ->update(['status' => 'published']);
    
    echo 'Bulk update count: ' . \$updated . PHP_EOL;
    
    // Verify update
    \$publishedCount = \App\Models\Post::whereIn('id', \$postIds)
                                      ->where('status', 'published')
                                      ->count();
    
    echo 'Bulk update verification: ' . (\$publishedCount === 3 ? 'OK' : 'Failed') . PHP_EOL;
    
    // Cleanup
    \App\Models\Post::whereIn('id', \$postIds)->forceDelete();
    echo 'Bulk test cleanup: OK' . PHP_EOL;
} else {
    echo 'Missing test data for bulk operations' . PHP_EOL;
}
"
```

#### Test 4: Transaction Handling Testing
```bash
echo "=== Transaction Handling Testing ==="

php artisan tinker --execute="
// Test transaction rollback pada error
try {
    \DB::beginTransaction();
    
    // Create valid post
    \$post = \App\Models\Post::create([
        'title' => 'Transaction Test Post',
        'content' => 'Testing transaction handling',
        'slug' => 'transaction-test-' . time(),
        'status' => 'draft',
        'user_id' => 1,
        'category_id' => 1,
    ]);
    
    echo 'Post created with ID: ' . \$post->id . PHP_EOL;
    
    // Simulate error (invalid foreign key)
    \App\Models\Post::create([
        'title' => 'Invalid Post',
        'content' => 'This should fail',
        'slug' => 'invalid-post',
        'status' => 'draft',
        'user_id' => 99999, // Non-existent user
        'category_id' => 99999, // Non-existent category
    ]);
    
    \DB::commit();
    echo 'Transaction committed (unexpected)' . PHP_EOL;
    
} catch (\Exception \$e) {
    \DB::rollback();
    echo 'Transaction rolled back: OK' . PHP_EOL;
    
    // Verify first post was also rolled back
    \$postExists = \App\Models\Post::where('title', 'Transaction Test Post')->exists();
    echo 'Rollback verification: ' . (!\$postExists ? 'OK' : 'Failed') . PHP_EOL;
}
"
```

#### Test 5: Service Class Integration Testing
```bash
echo "=== Service Class Integration Testing ==="

# Test service class exists and methods are available
php artisan tinker --execute="
if (class_exists('\App\Services\PostService')) {
    echo '✓ PostService class exists' . PHP_EOL;
    
    \$service = new \App\Services\PostService();
    \$methods = get_class_methods(\$service);
    \$requiredMethods = ['createPost', 'updatePost', 'deletePost', 'bulkUpdateStatus'];
    
    \$missingMethods = array_diff(\$requiredMethods, \$methods);
    if (empty(\$missingMethods)) {
        echo '✓ All required service methods exist' . PHP_EOL;
    } else {
        echo '✗ Missing service methods: ' . implode(', ', \$missingMethods) . PHP_EOL;
    }
} else {
    echo '✗ PostService class not found' . PHP_EOL;
}
"

# Test service injection dalam controller
php artisan tinker --execute="
try {
    \$controller = app(\App\Http\Controllers\PostController::class);
    echo '✓ PostController dependency injection working' . PHP_EOL;
} catch (\Exception \$e) {
    echo '✗ Controller dependency injection failed: ' . \$e->getMessage() . PHP_EOL;
}
"
```

### 🆘 TROUBLESHOOTING

#### Problem 1: Transaction Deadlock Issues
**Gejala:** `Deadlock found when trying to get lock` error during bulk operations
**Solusi:**
```bash
# Check MySQL deadlock status
mysql -u laravel_user -p laravel_app -e "SHOW ENGINE INNODB STATUS\G" | grep -A 20 "LATEST DETECTED DEADLOCK"

# Optimize transaction timeout
nano config/database.php
# Add to mysql connection config:
# 'options' => [
#     PDO::ATTR_TIMEOUT => 30,
#     PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => false,
# ]

# Reduce transaction scope
php artisan tinker --execute="
// Process bulk operations in smaller batches
\$postIds = [1, 2, 3, 4, 5];
\$chunks = array_chunk(\$postIds, 2); // Process 2 at a time

foreach (\$chunks as \$chunk) {
    \DB::transaction(function () use (\$chunk) {
        \App\Models\Post::whereIn('id', \$chunk)->update(['status' => 'published']);
    });
}
echo 'Chunked bulk operation completed' . PHP_EOL;
"
```

#### Problem 2: File Upload Cleanup Failures
**Gejala:** Old images tidak terhapus saat update atau delete
**Solusi:**
```bash
# Check storage permissions
ls -la storage/app/public/posts/
chmod -R 775 storage/app/public/

# Test file operations
php artisan tinker --execute="
// Test file existence check
\$testFile = 'posts/featured-images/test.jpg';
echo 'Storage disk: ' . config('filesystems.default') . PHP_EOL;
echo 'File exists method available: ' . (method_exists(\Storage::disk('public'), 'exists') ? 'Yes' : 'No') . PHP_EOL;

// Test file deletion
try {
    \Storage::disk('public')->put('test-delete.txt', 'test content');
    echo 'Test file created: OK' . PHP_EOL;
    
    \Storage::disk('public')->delete('test-delete.txt');
    echo 'Test file deleted: OK' . PHP_EOL;
} catch (\Exception \$e) {
    echo 'File operation failed: ' . \$e->getMessage() . PHP_EOL;
}
"

# Create file cleanup command
php artisan make:command CleanupOrphanedFiles
# Implement logic untuk menghapus files yang tidak terpakai
```

#### Problem 3: Bulk Operation Memory Issues
**Gejala:** `Fatal error: Allowed memory size exhausted` saat bulk operations
**Solusi:**
```bash
# Increase PHP memory limit
php -d memory_limit=512M artisan tinker

# Implement chunked processing
php artisan tinker --execute="
// Use chunk() method untuk large datasets
\App\Models\Post::where('status', 'draft')
                ->chunk(100, function (\$posts) {
                    foreach (\$posts as \$post) {
                        // Process individual post
                        \$post->update(['status' => 'published']);
                    }
                    echo 'Processed chunk of ' . \$posts->count() . ' posts' . PHP_EOL;
                });
"

# Update bulk operations untuk use chunking
nano app/Services/PostService.php
# Modify bulkUpdateStatus untuk process in chunks
```

#### Problem 4: Relationship Sync Errors
**Gejala:** Tags atau relationships tidak ter-update dengan benar
**Solusi:**
```bash
# Check pivot table structure
mysql -u laravel_user -p laravel_app -e "DESCRIBE post_tags;"

# Test relationship methods
php artisan tinker --execute="
\$post = \App\Models\Post::first();
if (\$post) {
    echo 'Post ID: ' . \$post->id . PHP_EOL;
    echo 'Current tags: ' . \$post->tags()->count() . PHP_EOL;
    
    // Test sync method
    \$tagIds = [1, 2, 3];
    \$post->tags()->sync(\$tagIds);
    echo 'After sync: ' . \$post->tags()->count() . PHP_EOL;
    
    // Test detach
    \$post->tags()->detach();
    echo 'After detach: ' . \$post->tags()->count() . PHP_EOL;
} else {
    echo 'No posts found for relationship testing' . PHP_EOL;
}
"

# Check for duplicate entries in pivot table
mysql -u laravel_user -p laravel_app -e "
SELECT post_id, tag_id, COUNT(*) as count 
FROM post_tags 
GROUP BY post_id, tag_id 
HAVING count > 1;"
```

#### Problem 5: Validation Errors pada Conditional Rules
**Gejala:** Form validation fails untuk conditional fields
**Solusi:**
```bash
# Test validation rules
php artisan tinker --execute="
\$request = new \Illuminate\Http\Request();
\$request->merge([
    'title' => 'Test Title',
    'content' => 'Test content with enough characters',
    'status' => 'published',
    'category_id' => 1,
    'user_id' => 1,
    'schedule_publish' => true,
    'published_at' => '2025-12-25 10:00:00'
]);

\$validator = \Validator::make(\$request->all(), [
    'title' => 'required|string|max:255',
    'content' => 'required|string|min:10',
    'status' => 'required|in:draft,published,archived',
    'category_id' => 'required|exists:categories,id',
    'user_id' => 'required|exists:users,id',
    'published_at' => 'nullable|date',
    'schedule_publish' => 'boolean',
]);

if (\$validator->passes()) {
    echo '✓ Validation passed' . PHP_EOL;
} else {
    echo '✗ Validation failed:' . PHP_EOL;
    foreach (\$validator->errors()->all() as \$error) {
        echo '  - ' . \$error . PHP_EOL;
    }
}
"

# Add custom validation rule untuk conditional fields
php artisan make:rule ConditionalPublishDate
```

#### Problem 6: JavaScript Bulk Operations Not Working
**Gejala:** Bulk selection atau AJAX calls tidak berfungsi
**Solusi:**
```bash
# Check CSRF token availability
php artisan tinker --execute="
echo 'CSRF token generation: ' . csrf_token() . PHP_EOL;
echo 'Session driver: ' . config('session.driver') . PHP_EOL;
"

# Test AJAX endpoint manually
curl -X POST http://localhost:8000/posts/bulk-update-status \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{"post_ids":[1,2],"status":"published"}' | jq .

# Check browser console for JavaScript errors
# Add debugging to JavaScript functions
echo 'Add console.log statements to bulk operations JavaScript'

# Verify route definitions
php artisan route:list | grep bulk
```

### 📋 DELIVERABLES

**Checklist yang harus diserahkan pada akhir sesi:**

#### ✅ Enhanced Update Operations
- [ ] Screenshot edit form dengan advanced features (image replacement, scheduling)
- [ ] Video/screenshot testing file upload replacement functionality
- [ ] Screenshot real-time form features (slug preview, character counter)
- [ ] Testing update operations dengan berbagai skenario (title change, status change, file operations)

#### ✅ Advanced Delete Operations
- [ ] Screenshot delete operations dengan cascade cleanup verification
- [ ] Database screenshots showing proper relationship cleanup
- [ ] Log files showing audit trail untuk delete operations
- [ ] Testing soft delete dan restore functionality

#### ✅ Bulk Operations Implementation
- [ ] Screenshot bulk selection UI dalam posts index
- [ ] Video demonstration bulk delete dan bulk status update
- [ ] API testing screenshots untuk bulk endpoints
- [ ] Performance testing results untuk bulk operations

#### ✅ Transaction Handling
- [ ] PostService class implementation dengan comprehensive error handling
- [ ] Transaction testing screenshots showing rollback functionality
- [ ] Error logging verification dan audit trail examples
- [ ] Service integration testing results

#### ✅ Advanced UI Features
- [ ] Enhanced forms dengan JavaScript functionality working
- [ ] Real-time validation dan feedback systems
- [ ] AJAX operations working tanpa page refresh
- [ ] Responsive design testing pada berbagai screen sizes

#### ✅ Code Quality dan Architecture
- [ ] Service class following SOLID principles
- [ ] Proper error handling dan logging implementation
- [ ] Comprehensive validation rules dan security measures
- [ ] Code documentation dan inline comments

**Format Submission:**
```bash
cd ~/praktikum-cc
mkdir -p submission/week5/{controllers,services,views,js,tests,logs}

# Copy enhanced files
cp app/Http/Controllers/PostController.php submission/week5/controllers/
cp app/Services/PostService.php submission/week5/services/
cp resources/views/posts/edit.blade.php submission/week5/views/
cp resources/views/posts/index.blade.php submission/week5/views/

# Copy route files
cp routes/web.php submission/week5/routes-web.php
cp routes/api.php submission/week5/routes-api.php

# Export recent logs for audit trail
tail -50 storage/logs/laravel.log > submission/week5/logs/recent-operations.log

**Format Submission:**
1. Buat folder submission/week5/
2. Masukkan semua screenshot dengan nama yang jelas
3. Buat file laporan dalam format Markdown
4. Commit dan push ke repository
5. Sertakan link commit terakhir