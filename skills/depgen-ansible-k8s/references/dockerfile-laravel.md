# Dockerfile Pattern — Laravel

Multi-stage Dockerfile for Laravel applications with PHP-FPM and Nginx.

## Detection Inputs

| Input | Source | Used For |
|---|---|---|
| PHP version | `composer.json` → `require.php` | Base image tag |
| Laravel version | `composer.json` → `require.laravel/framework` | Compatibility |
| Has frontend | `vite.config.js` or `webpack.mix.js` present | Include Node build stage |
| App port | `.env.example` → `APP_PORT` or default `8000` | EXPOSE / Nginx config |

## Template — PHP-FPM + Nginx (Production)

```dockerfile
# =============================================================================
# Stage 1: Frontend Build (if Vite/Mix is used)
# =============================================================================
FROM node:22-alpine AS frontend

WORKDIR /app

# Copy package files for dependency caching
COPY package.json package-lock.json ./
RUN npm ci --production=false

# Copy frontend source and build
COPY resources ./resources
COPY vite.config.js ./
COPY tailwind.config.js ./
COPY postcss.config.js ./
RUN npm run build

# =============================================================================
# Stage 2: Composer Dependencies
# =============================================================================
FROM composer:2 AS composer

WORKDIR /app

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

COPY . .
RUN composer dump-autoload --optimize --no-dev

# =============================================================================
# Stage 3: Runtime (PHP-FPM + Nginx)
# =============================================================================
FROM php:{php_version}-fpm-alpine AS runtime

# Install required PHP extensions
RUN apk add --no-cache \
    nginx \
    supervisor \
    && docker-php-ext-install pdo pdo_mysql opcache

# Configure PHP for production
COPY docker/php.ini /usr/local/etc/php/conf.d/app.ini
COPY docker/nginx.conf /etc/nginx/http.d/default.conf
COPY docker/supervisord.conf /etc/supervisord.conf

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /var/www/html

# Copy application from composer stage
COPY --from=composer /app .

# Copy built frontend assets from frontend stage
COPY --from=frontend /app/public/build ./public/build

# Set permissions
RUN chown -R appuser:appgroup /var/www/html \
    && chmod -R 775 storage bootstrap/cache

# Expose port
EXPOSE {app_port}

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:{app_port}/health || exit 1

# Production defaults
ENV APP_ENV=production
ENV APP_DEBUG=false
ENV LOG_CHANNEL=stderr

ENTRYPOINT ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

## Supporting Files

The Dockerfile references supporting config files that should be documented in
DEPLOYMENT.md. These are placed in a `docker/` directory in the application root.

### docker/php.ini

```ini
[PHP]
display_errors = Off
log_errors = On
error_log = /dev/stderr
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0
upload_max_filesize = 50M
post_max_size = 50M
memory_limit = 512M
```

### docker/nginx.conf

```nginx
server {
    listen {app_port};
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### docker/supervisord.conf

```ini
[supervisord]
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0

[program:php-fpm]
command=/usr/local/sbin/php-fpm --nodaemonize
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

## Template — Artisan Serve (Development/Simple)

For simpler setups without Nginx:

```dockerfile
FROM php:{php_version}-cli-alpine AS runtime

RUN docker-php-ext-install pdo pdo_mysql

WORKDIR /var/www/html
COPY --from=composer /app .
COPY --from=frontend /app/public/build ./public/build

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown -R appuser:appgroup /var/www/html
USER appuser

EXPOSE {app_port}
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port={app_port}"]
```

## Notes

- **Queue workers**: If the application uses Laravel queues (detected by `QUEUE_CONNECTION`
  env var or `app/Jobs/` directory), document a separate Deployment for the queue worker
  container using the same image but `CMD ["php", "artisan", "queue:work"]`.
- **Scheduler**: If `app/Console/Kernel.php` has scheduled tasks, document a CronJob
  manifest using `CMD ["php", "artisan", "schedule:run"]`.
- **Storage**: Laravel's `storage/` directory needs a writable volume if file uploads
  are used. Document a PersistentVolumeClaim in DEPLOYMENT.md.
