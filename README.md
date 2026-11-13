# WordPress Object Cache Diagnostics

A practical guide to diagnosing, debugging, and optimizing WordPress object caching using WP-CLI, PHP, Redis, Memcached, and WordPress Object Cache API functions.

## What Is WordPress Object Cache?

WordPress Object Cache stores frequently accessed data in memory to reduce repeated database queries and improve application performance.

By default, the WordPress object cache is request-based. Its contents do not persist between page loads unless a persistent object cache backend is installed.

Common persistent caching solutions include:

- Redis
- Memcached
- Docket Cache
- SQLite Object Cache

## Common Object Cache Problems

- Object cache does not persist between requests.
- Redis connection failures.
- Memcached connection errors.
- Cache invalidation does not work correctly.
- Stale data appears after updates.
- Object cache drop-in is missing.
- Cache groups behave unexpectedly.
- Multisite cache conflicts.
- Cache performance decreases under heavy traffic.

## 1. Check Object Cache Status

Use WP-CLI to identify the active object cache implementation:

```bash
wp cache type
```

Check whether a persistent object cache drop-in exists:

```bash
ls -l wp-content/object-cache.php
```

Inspect the WordPress configuration:

```bash
grep -n "WP_CACHE" wp-config.php
```

Important: `WP_CACHE` is not a requirement for modern persistent object caching. The presence of an `object-cache.php` drop-in and a functioning backend are more relevant.

## 2. Test Object Cache Functionality

Use the WordPress Object Cache API:

```php
wp_cache_set(
    'diagnostic_key',
    'cache_test',
    'diagnostics',
    300
);

$value = wp_cache_get(
    'diagnostic_key',
    'diagnostics'
);

var_dump( $value );
```

Expected result:

```text
string(10) "cache_test"
```

This verifies basic cache operations within the current request.

To test persistence, write a value in one request and retrieve it in a separate request.

## 3. Understand Cache Hits and Misses

Cache hits occur when requested data is already available.

Cache misses occur when the requested data cannot be retrieved.

Example:

```php
$value = wp_cache_get(
    'product_data',
    'my_plugin',
    false,
    $found
);

if ( ! $found ) {

    $value = get_expensive_product_data();

    wp_cache_set(
        'product_data',
        $value,
        'my_plugin',
        HOUR_IN_SECONDS
    );

}
```

Using `$found` is important when `false` is a legitimate cached value.

This pattern helps avoid unnecessary database queries and repeated expensive operations.

## 4. Troubleshoot Redis Connection Issues

Redis provides an external in-memory storage backend for persistent object caching.

Check whether Redis is available:

```bash
redis-cli ping
```

Expected response:

```text
PONG
```

Check Redis server information:

```bash
redis-cli INFO server
```

Inspect memory usage:

```bash
redis-cli INFO memory
```

Check connected clients:

```bash
redis-cli CLIENT LIST
```

Investigate:

- Redis server availability
- Incorrect hostname or port
- Authentication failures
- Network restrictions
- Memory limits
- Connection timeouts
- Redis service configuration

These commands require Redis CLI access and appropriate server permissions.

## 5. Troubleshoot Memcached

Check whether Memcached is running:

```bash
systemctl status memcached
```

Check whether the service is listening:

```bash
ss -lntp | grep 11211
```

Investigate:

- Memcached service availability
- Incorrect server address
- PHP Memcached extension
- Connection limits
- Memory allocation
- Firewall restrictions

Verify that the PHP extension required by the chosen WordPress cache implementation is installed.

## 6. Inspect Object Cache Drop-in

WordPress supports replacing its default object cache implementation through:

```text
wp-content/object-cache.php
```

Inspect the file:

```bash
ls -lh wp-content/object-cache.php
```

Check its ownership and permissions:

```bash
stat wp-content/object-cache.php
```

If persistent caching stops working, investigate:

1. Whether the drop-in exists.
2. Whether the file is readable by PHP.
3. Whether the cache plugin is active and correctly configured.
4. Whether the backend service is available.
5. Whether PHP reports initialization errors.

Do not delete or replace the drop-in without understanding which caching implementation owns it.

## 7. Inspect Cache Groups

WordPress organizes cached values using cache groups.

Example:

```php
wp_cache_set(
    'user_dashboard',
    $dashboard_data,
    'my_plugin',
    600
);
```

Retrieve the cached value:

```php
$data = wp_cache_get(
    'user_dashboard',
    'my_plugin'
);
```

Delete a specific cached item:

```php
wp_cache_delete(
    'user_dashboard',
    'my_plugin'
);
```

Use meaningful cache groups to organize plugin and application data.

Avoid using generic cache keys that may conflict with other components.

## 8. Troubleshoot Cache Invalidation

Stale cache data often occurs when cached values are not updated or deleted after the underlying data changes.

Example:

```php
function update_product_cache( $post_id ) {

    if ( 'product' !== get_post_type( $post_id ) ) {
        return;
    }

    wp_cache_delete(
        'product_' . $post_id,
        'my_plugin'
    );

}

add_action(
    'save_post',
    'update_product_cache'
);
```

For more complex integrations, consider:

- Updating related cache keys.
- Invalidating dependent cache groups.
- Using appropriate WordPress lifecycle hooks.
- Setting reasonable expiration times.
- Avoiding unnecessary full-cache flushes.

## 9. Inspect Cache Backend Capabilities

Not every persistent object cache implementation supports every cache operation.

Check whether a feature is supported:

```php
if ( wp_cache_supports( 'flush_group' ) ) {

    wp_cache_flush_group( 'my_plugin' );

}
```

This matters because some WordPress cache functions may have fallback behavior when a backend does not support a particular operation.

Never assume that flushing one group will always leave every other cache entry untouched.

## 10. Manage Object Cache Using WP-CLI

Inspect cache implementation:

```bash
wp cache type
```

Check cache support:

```bash
wp cache supports flush_group
```

Set a cache value:

```bash
wp cache set test_key test_value diagnostics 300
```

Retrieve a cache value:

```bash
wp cache get test_key diagnostics
```

Delete a cache value:

```bash
wp cache delete test_key diagnostics
```

Flush a specific cache group when supported:

```bash
wp cache flush-group diagnostics
```

Flush the entire object cache:

```bash
wp cache flush
```

**Warning:** On multisite installations with persistent object caching, a full cache flush can affect every site in the network. It can also temporarily increase database load while cached data is regenerated.

## 11. Diagnose Multisite Object Cache Issues

WordPress Multisite requires careful handling of cache groups.

Some cache groups are global across the network, while others are site-specific.

Register a global cache group:

```php
add_action( 'init', function () {

    wp_cache_add_global_groups(
        array(
            'my_network_data',
        )
    );

} );
```

Register a non-persistent cache group:

```php
add_action( 'init', function () {

    wp_cache_add_non_persistent_groups(
        array(
            'request_data',
        )
    );

} );
```

Investigate:

- Incorrect global cache group registration
- Site-specific cache key collisions
- Cross-site data leakage
- Inconsistent cache invalidation
- Backend configuration shared across sites

Never place private site-specific data in a global cache group without proper site-aware key isolation.

## 12. Optimize Object Cache Performance

Consider these optimization techniques:

- Cache expensive database queries.
- Use meaningful cache groups.
- Set appropriate expiration times.
- Avoid caching unnecessarily large objects.
- Invalidate cached data when source data changes.
- Use persistent caching when appropriate.
- Monitor Redis or Memcached memory usage.
- Avoid unnecessary full-cache flushes.
- Use bulk cache operations when supported.

Example using bulk operations:

```php
$items = array(
    'product_101' => array( 'name' => 'Product A' ),
    'product_102' => array( 'name' => 'Product B' ),
);

wp_cache_set_multiple(
    $items,
    'my_plugin',
    600
);
```

Check backend support before depending on optimized cache operations.

## 13. Essential Troubleshooting Commands

| Command | Purpose |
|---|---|
| `wp cache type` | Identify cache implementation |
| `wp cache supports flush_group` | Check cache feature support |
| `wp cache set` | Store a cache value |
| `wp cache get` | Retrieve a cache value |
| `wp cache delete` | Delete a cache value |
| `wp cache flush` | Flush object cache |
| `redis-cli ping` | Test Redis connectivity |
| `redis-cli INFO memory` | Inspect Redis memory |
| `systemctl status memcached` | Check Memcached service |
| `ls -l wp-content/object-cache.php` | Inspect cache drop-in |

## Object Cache Troubleshooting Checklist

- [ ] Verify the object-cache.php drop-in.
- [ ] Identify the active cache implementation.
- [ ] Test cache read and write operations.
- [ ] Verify persistence across requests.
- [ ] Check Redis or Memcached connectivity.
- [ ] Inspect PHP extensions.
- [ ] Investigate cache initialization errors.
- [ ] Review cache group configuration.
- [ ] Verify cache invalidation logic.
- [ ] Check multisite cache isolation.
- [ ] Monitor memory consumption.
- [ ] Test performance before and after optimization.

## Important Security Guidelines

1. Never expose Redis or Memcached directly to the public internet.
2. Restrict cache backend access to trusted applications and networks.
3. Protect Redis authentication credentials.
4. Avoid storing sensitive information without appropriate safeguards.
5. Prevent cross-site data leakage in multisite environments.
6. Avoid unnecessary cache flushing in production.
7. Keep cache plugins and backend services updated.
8. Use staging environments to test configuration changes.

## Conclusion

WordPress Object Cache plays an important role in reducing database queries and improving website performance.

Understanding cache persistence, backend connectivity, cache groups, invalidation, and WP-CLI diagnostics helps developers identify caching problems more effectively.

This repository serves as a practical technical reference for WordPress developers and administrators working with persistent object caching.
