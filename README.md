# Tutorial Laravel: Panduan Teknis + Boilerplate (Pemula-Menengah)

Panduan ini berisi alur praktis membangun aplikasi Laravel dari nol dengan konsep inti:

- Pengenalan MVC (Model, View, Controller)
- Routing dan Middleware
- Blade Templating + inheritance
- Database dengan Eloquent ORM dan Migration
- Authentication (starter kit)
- Validasi input dengan Form Request
- Upload file ke storage
- RESTful API dengan Laravel Sanctum
- Langkah deployment ke hosting

---

## 1) Setup Project Laravel

```bash
composer create-project laravel/laravel laravel-tutorial
cd laravel-tutorial
cp .env.example .env
php artisan key:generate
```

Atur koneksi database di `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_tutorial
DB_USERNAME=root
DB_PASSWORD=
```

Jalankan server lokal:

```bash
php artisan serve
```

---

## 2) Pengenalan MVC

### A. Model (`app/Models/Post.php`)

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;

    protected $fillable = [
        'title',
        'content',
        'thumbnail',
        'user_id',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

### B. Controller (`app/Http/Controllers/PostController.php`)

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Storage;
use Illuminate\View\View;

class PostController extends Controller
{
    public function index(): View
    {
        $posts = Post::latest()->paginate(10);

        return view('posts.index', compact('posts'));
    }

    public function create(): View
    {
        return view('posts.create');
    }

    public function store(StorePostRequest $request): RedirectResponse
    {
        $data = $request->validated();

        if ($request->hasFile('thumbnail')) {
            $data['thumbnail'] = $request->file('thumbnail')->store('thumbnails', 'public');
        }

        $request->user()->posts()->create($data);

        return redirect()->route('posts.index')->with('success', 'Post berhasil dibuat.');
    }

    public function destroy(Post $post): RedirectResponse
    {
        if ($post->thumbnail) {
            Storage::disk('public')->delete($post->thumbnail);
        }

        $post->delete();

        return redirect()->route('posts.index')->with('success', 'Post berhasil dihapus.');
    }
}
```

### C. View (Blade)
View bertugas menampilkan data dari controller ke user.

---

## 3) Routing + Middleware

Contoh di `routes/web.php`:

```php
<?php

use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return view('welcome');
});

Route::middleware(['auth'])->group(function () {
    Route::resource('posts', PostController::class)
        ->only(['index', 'create', 'store', 'destroy']);
});
```

Contoh middleware custom (`app/Http/Middleware/EnsureUserIsAdmin.php`):

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserIsAdmin
{
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user() || ! $request->user()->is_admin) {
            abort(403, 'Akses ditolak.');
        }

        return $next($request);
    }
}
```

Daftarkan middleware di `bootstrap/app.php` (Laravel terbaru) pada bagian alias middleware.

---

## 4) Blade Templating + Inheritance

### Layout utama (`resources/views/layouts/app.blade.php`)

```blade
<!doctype html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', 'Tutorial Laravel')</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <main class="container mx-auto p-6">
        @if (session('success'))
            <div class="mb-4 rounded bg-green-100 p-3 text-green-700">
                {{ session('success') }}
            </div>
        @endif

        @yield('content')
    </main>
</body>
</html>
```

### Halaman turunan (`resources/views/posts/index.blade.php`)

```blade
@extends('layouts.app')

@section('title', 'Daftar Post')

@section('content')
    <h1 class="mb-4 text-2xl font-bold">Daftar Post</h1>

    <a href="{{ route('posts.create') }}" class="mb-4 inline-block rounded bg-blue-600 px-4 py-2 text-white">
        + Tambah Post
    </a>

    <ul class="space-y-3">
        @forelse($posts as $post)
            <li class="rounded border p-4">
                <h2 class="font-semibold">{{ $post->title }}</h2>
                <p class="text-gray-600">{{ \Illuminate\Support\Str::limit($post->content, 120) }}</p>
            </li>
        @empty
            <li>Belum ada data post.</li>
        @endforelse
    </ul>

    <div class="mt-4">
        {{ $posts->links() }}
    </div>
@endsection
```

---

## 5) Database: Migration + Eloquent ORM

Buat migration:

```bash
php artisan make:migration create_posts_table
```

Isi migration (`database/migrations/xxxx_xx_xx_create_posts_table.php`):

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->text('content');
            $table->string('thumbnail')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

Jalankan migrasi:

```bash
php artisan migrate
```

Contoh query Eloquent:

```php
$publishedPosts = Post::with('user')->latest()->take(5)->get();
```

---

## 6) Authentication dengan Starter Kit (Breeze)

Install Breeze:

```bash
composer require laravel/breeze
php artisan breeze:install blade
npm install
npm run build
php artisan migrate
```

Fitur auth (register, login, logout, reset password) akan otomatis tersedia.

---

## 7) Validasi Input dengan Form Request

Generate form request:

```bash
php artisan make:request StorePostRequest
```

Isi file `app/Http/Requests/StorePostRequest.php`:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->check();
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'content' => ['required', 'string', 'min:20'],
            'thumbnail' => ['nullable', 'image', 'mimes:jpg,jpeg,png,webp', 'max:2048'],
        ];
    }
}
```

Catatan: `max:2048` berarti ukuran maksimum file **2048 KB** (sekitar 2 MB).

Gunakan request ini langsung di controller agar validasi lebih bersih.

---

## 8) Upload File ke Storage

1. Simpan file:
   ```php
   $path = $request->file('thumbnail')->store('thumbnails', 'public');
   ```
2. Buat symbolic link:
   ```bash
   php artisan storage:link
   ```
3. Tampilkan gambar di Blade:
   ```blade
   <img src="{{ asset('storage/' . $post->thumbnail) }}" alt="Thumbnail {{ $post->title }}">
   ```

---

## 9) RESTful API + Laravel Sanctum

Install Sanctum:

```bash
composer require laravel/sanctum
php artisan migrate
```

Tambahkan route API di `routes/api.php`:

```php
<?php

use App\Http\Controllers\Api\PostApiController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::post('/tokens/create', function (Request $request) {
    $request->validate([
        'email' => ['required', 'email'],
        'password' => ['required'],
    ]);

    $user = \App\Models\User::where('email', $request->email)->first();

    if (! $user || ! \Illuminate\Support\Facades\Hash::check($request->password, $user->password)) {
        return response()->json(['message' => 'Email atau password salah'], 401);
    }

    $token = $user->createToken('api-token')->plainTextToken;

    return response()->json(['token' => $token]);
});

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostApiController::class);
});
```

Contoh controller API (`app/Http/Controllers/Api/PostApiController.php`):

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Models\Post;
use Illuminate\Http\JsonResponse;

class PostApiController extends Controller
{
    public function index(): JsonResponse
    {
        return response()->json(Post::latest()->paginate(10));
    }

    public function store(StorePostRequest $request): JsonResponse
    {
        $post = $request->user()->posts()->create($request->validated());

        return response()->json($post, 201);
    }

    public function show(Post $post): JsonResponse
    {
        return response()->json($post);
    }

    public function update(StorePostRequest $request, Post $post): JsonResponse
    {
        $post->update($request->validated());

        return response()->json($post);
    }

    public function destroy(Post $post): JsonResponse
    {
        $post->delete();

        return response()->noContent();
    }
}
```

Uji API cepat via cURL:

```bash
curl -X GET http://127.0.0.1:8000/api/posts \
  -H "Authorization: Bearer TOKEN_SANCTUM"
```

---

## 10) Deployment ke Hosting (Ringkas)

### Opsi A: VPS (Nginx + PHP-FPM)
1. Upload source code ke server.
2. Pastikan **Composer**, **PHP extension yang dibutuhkan Laravel**, serta **Node.js/NPM** (untuk build aset) sudah terpasang di server.
3. Jalankan:
   ```bash
   composer install --optimize-autoloader --no-dev
   php artisan migrate --force
   php artisan config:cache
   php artisan route:cache
   php artisan view:cache
   npm ci && npm run build
   ```
4. Pastikan `.env` production benar (`APP_ENV=production`, `APP_DEBUG=false`).
5. Set permission `storage` dan `bootstrap/cache`.
6. Setup virtual host Nginx ke folder `public/`.

### Opsi B: Shared Hosting
1. Upload project.
2. Pindahkan isi folder `public` ke `public_html` (atau atur document root ke `public`).
3. Sesuaikan `index.php` path ke folder Laravel inti.
4. Jalankan migrasi dan cache command via terminal hosting (jika tersedia).

Checklist production:
- HTTPS aktif
- Queue/cron job aktif (`php artisan schedule:run`)
- Backup database terjadwal
- Monitoring log (`storage/logs/laravel.log`)

---

## 11) Tips Clean Code (Praktik Baik)

- Gunakan **Form Request** untuk validasi, bukan menumpuk di controller.
- Gunakan **resource controller** agar route konsisten.
- Pisahkan logic berat ke **Service class** bila controller mulai besar.
- Gunakan eager loading (`with`) untuk mencegah N+1 query.
- Beri nama route yang jelas (`posts.index`, `posts.store`, dst).

---

## 12) Urutan Belajar Disarankan

1. MVC dasar + route sederhana
2. CRUD + migration + Eloquent
3. Blade inheritance + komponen
4. Authentication (Breeze)
5. Form Request + upload file
6. API Sanctum + konsumsi dari frontend/mobile
7. Deployment + hardening production

Selesai. Dengan boilerplate ini, Anda sudah punya fondasi aplikasi Laravel modern yang siap dikembangkan.
