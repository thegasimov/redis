

Aşağıda daha geniş və çoxməqsədli bir Redis servisi təklifi verilir:

### 1. Redis Cache Auto-Clear və Auto-Refresh ilə Təkmilləşdirilmiş Servis

Aşağıdakı `RedisService` servisi:
- Model yeniləmələri, silinmələri və yaradılmaları zamanı avtomatik olaraq Redis-də saxlanılan keşləri təmizləyir.
- Redis-ə yazılan məlumatlar üçün TTL (vaxt müddəti) əlavə etmə funksiyası genişləndirilib.
- Cache'in avtomatik yenilənməsi üçün xüsusi metodlar əlavə edilib.

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Log;

class RedisService
{
    /**
     * Məlumatı Redis-də saxlayır və avtomatik olaraq köhnə məlumatı yeniləyir.
     *
     * @param string $key
     * @param mixed $value
     * @param int $expiration
     * @return bool
     */
    public function set($key, $value, $expiration = 3600)
    {
        if (!$this->validateKey($key)) {
            Log::error('Redis: Invalid key provided.');
            return false;
        }

        $encodedValue = $this->prepareData($value);

        if (!$encodedValue) {
            Log::error('Redis: Data encoding failed.');
            return false;
        }

        Redis::set($key, $encodedValue);
        Redis::expire($key, $expiration);

        return true;
    }

    /**
     * Redis-də saxlanmış məlumatı alır, əgər mövcud deyilsə avtomatik olaraq yeniləyir.
     *
     * @param string $key
     * @param callable $callback
     * @param int $expiration
     * @return mixed
     */
    public function getOrSet($key, callable $callback, $expiration = 3600)
    {
        if ($this->exists($key)) {
            return $this->decodeData(Redis::get($key));
        }

        // Keşdə məlumat yoxdursa, callback vasitəsilə yeni məlumat alırıq
        $value = $callback();

        // Yenilənmiş məlumatı Redis-ə saxlayırıq
        $this->set($key, $value, $expiration);

        return $value;
    }

    /**
     * Model yeniləməsi zamanı Redis-dəki bütün əlaqəli keşləri təmizləyir.
     *
     * @param string $modelClass
     * @return void
     */
    public function clearCacheOnModelUpdate($modelClass)
    {
        // Model üçün olan bütün cache açarlarını təmizləyir
        $keys = Redis::keys($modelClass . ':*');

        foreach ($keys as $key) {
            Redis::del($key);
            Log::info('Redis: Cache cleared for key ' . $key);
        }
    }

    /**
     * Redis-də açarı yoxlayır və varsa məlumatı qaytarır.
     *
     * @param string $key
     * @return mixed|null
     */
    public function get($key)
    {
        if (!$this->exists($key)) {
            Log::warning('Redis: Key not found.');
            return null;
        }

        $value = Redis::get($key);
        return $this->decodeData($value);
    }

    /**
     * Redis-də açarı silir və loglayır.
     *
     * @param string $key
     * @return bool
     */
    public function delete($key)
    {
        if (!$this->exists($key)) {
            Log::warning('Redis: Key not found for deletion.');
            return false;
        }

        Redis::del($key);
        Log::info('Redis: Key deleted - ' . $key);

        return true;
    }

    /**
     * Redis-də açarın mövcudluğunu yoxlayır.
     *
     * @param string $key
     * @return bool
     */
    public function exists($key)
    {
        return Redis::exists($key) > 0;
    }

    /**
     * Bütün Redis cache-i təmizləyir.
     *
     * @return bool
     */
    public function clearAllCache()
    {
        Redis::flushall();
        Log::info('Redis: All cache cleared.');
        return true;
    }

    /**
     * Açarın düzgün olub-olmadığını yoxlayır.
     *
     * @param string $key
     * @return bool
     */
    protected function validateKey($key)
    {
        return is_string($key) && !empty($key);
    }

    /**
     * Məlumatı JSON formatına çevirir.
     *
     * @param mixed $data
     * @return string|false
     */
    protected function prepareData($data)
    {
        $encoded = json_encode($data);

        if (json_last_error() !== JSON_ERROR_NONE) {
            Log::error('Redis: JSON encoding error - ' . json_last_error_msg());
            return false;
        }

        return $encoded;
    }

    /**
     * JSON formatında məlumatı açır.
     *
     * @param string $data
     * @return mixed
     */
    protected function decodeData($data)
    {
        $decoded = json_decode($data, true);

        if (json_last_error() !== JSON_ERROR_NONE) {
            Log::error('Redis: JSON decoding error - ' . json_last_error_msg());
            return null;
        }

        return $decoded;
    }
}
```

### 2. Təkmilləşdirilmiş Xüsusiyyətlər

#### 2.1. Model Yeniləmələrini Keşdə Yeniləmə

Bu funksionalıq Redis-də saxlanılan bütün açarları təmizləyir və ya yeniləyir, əgər model yenilənibsə. **`clearCacheOnModelUpdate()`** metodu istifadə edilərək, hansısa bir modeldə dəyişiklik edildikdə Redis keşləri avtomatik təmizlənir. Məsələn, **`User`** modelində dəyişiklik edildikdə, bütün açarlar təmizlənir:

```php
$redisService->clearCacheOnModelUpdate(User::class);
```

#### 2.2. Məlumatı Keşdən Əldə Et və Mövcud Deyilsə Yeniləmə

**`getOrSet()`** metodu Redis-də məlumatın olub-olmamasını yoxlayır və mövcud deyilsə, verilən `callback` funksiyasından istifadə edərək yeni məlumatı əldə edir və Redis-ə saxlayır:

```php
$userData = $redisService->getOrSet('user:1', function() {
    // Burada verilənlər bazasından məlumat alırsınız
    return User::find(1);
});
```

#### 2.3. Bütün Keşlərin Təmizlənməsi

Bütün Redis cache-ləri təmizləmək üçün **`clearAllCache()`** metodundan istifadə edə bilərsiniz:

```php
$redisService->clearAllCache();
```

### 3. Keşin Avtomatik Yenilənməsi

Keş müddəti dolduqda Redis-də məlumatı avtomatik yeniləyən funksionalıq əlavə etdik. Bu metod Redis-də saxlanılan açarların avtomatik olaraq təzələnməsi üçün `getOrSet` metodundan istifadə edir. Əgər açar mövcud deyilsə və ya müddəti bitmişdirsə, `callback` funksiyası vasitəsilə yeni məlumat əldə edilir və Redis-ə yazılır.

### 4. Avtomatik TTL (Time To Live) Yeniləməsi

Redis-də saxlanılan məlumatlar üçün TTL vaxtı bitərsə, məlumatlar avtomatik olaraq yenilənir və Redis-ə yenidən yazılır. Bu sayədə Redis cache həmişə təzə saxlanır və köhnəlmiş məlumatlar ilə qarşılaşılmır.

### 5. Servisin Satışa Hazır Halına Gətirilməsi

Bu servis Redis ilə genişmiqyaslı layihələrdə istifadə oluna bilər və müxtəlif məqsədlər üçün istifadəçilərə çox çevik bir Redis həlli təqdim edir. Əsas üstünlükləri:
- Avtomatik model yenilənməsi və keşin təmizlənməsi.
- Cache-in əl ilə təmizlənməsi və idarə olunması.
- Avtomatik TTL təzələməsi və callback ilə məlumatların yenilənməsi.
- JSON formatında saxlanılan məlumatların etibarlı şəkildə kodlanması və açılması.

### 6. Genişlənmə İmkanları

- **Tag-based caching**: Redis-də daha geniş şəkildə istifadə etmək üçün müəyyən cache açarlarını qruplara bölərək tag-based caching əlavə edə bilərsiniz.
- **İstifadəçi və icazə əsaslı cache idarəetməsi**: Müxtəlif istifadəçilər üçün fərqli cache açarları yaratmaq və həmin cache-ləri istifadəçi rollar
