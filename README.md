# mastodon-on-mac

Step-by-step instructions for installing and running the current stable release of [Mastodon](https://github.com/mastodon/mastodon) (v4.7.1 as of September 2026) on a Mac, using only Homebrew for the toolchain: no `rbenv`, `chruby`, `nodenv`, Ruby on Mac, or Docker.

Two deliberate deviations from the [official install guide](https://docs.joinmastodon.org/admin/install/):

  - **Apache httpd** is used as the reverse proxy instead of Nginx ([com.example.social.conf](com.example.social.conf)).
  - **Redis is shared** with other applications on the same Mac. Mastodon gets its own Redis database number rather than a dedicated Redis server.

Everything is written for an Apple Silicon Mac where Homebrew lives in `/opt/homebrew`. On an Intel Mac, replace `/opt/homebrew` with `/usr/local` throughout. `foo` is the macOS user that will run Mastodon; `example.com` is the domain.

## What Mastodon 4.7 requires

| Dependency | Mastodon requires | What we install |
| --- | --- | --- |
| Ruby | 3.3 – 4.0 (`.ruby-version` is 4.0.6) | Homebrew `ruby` (4.0.6) |
| Node.js | 22+ (`.nvmrc` is 24.x) | Homebrew `node@24` |
| Yarn | 4.x, via corepack | corepack (bundled with Node) |
| PostgreSQL | 14+ | Homebrew `postgresql@17` |
| Redis | 7.0+ | Your existing Homebrew `redis` |
| libvips | 8.13+ | Homebrew `vips` |
| FFmpeg | 5.1+ | Homebrew `ffmpeg` |
| Elasticsearch | optional, 7.x (OpenSearch "should work") | Homebrew `opensearch`, optional |

Since 4.3 Mastodon processes images with libvips; as of 4.7 the libvips initializer is loaded unconditionally, so `vips` is mandatory. ImageMagick comes in as a dependency of the `vips` formula and does not need to be installed separately.

## 1. Install packages

```bash
brew install ruby node@24 postgresql@17 vips ffmpeg libidn icu4c httpd supervisor
```

If Redis is not already installed:

```bash
brew install redis
```

Check your Redis version. Mastodon needs 7.0 or newer:

```bash
redis-server --version
```

`ruby`, `node@24`, and `postgresql@17` are all keg-only or version-tracking, so put them on your `PATH`. Add this to `~/.zprofile`:

```bash
export PATH="/opt/homebrew/opt/ruby/bin:/opt/homebrew/lib/ruby/gems/4.0.0/bin:/opt/homebrew/opt/node@24/bin:/opt/homebrew/opt/postgresql@17/bin:$PATH"
```

Then open a new shell, or `source ~/.zprofile`.

The Homebrew `ruby` formula tracks the newest Ruby. Mastodon 4.7 requires Ruby `< 4.1`, so pin it to stop `brew upgrade` from moving past what Mastodon supports:

```bash
brew pin ruby
```

Verify:

```bash
ruby --version && node --version && pg_config --version && vips --version
```

Enable corepack so the exact Yarn version Mastodon wants (`yarn@4.18.0`) is fetched automatically the first time you run `yarn`:

```bash
corepack enable
```

## 2. Start and configure PostgreSQL

```bash
brew services start postgresql@17
```

Create the `mastodon` role. On a single-user Mac the simplest arrangement is a password-less role that owns its own database:

```bash
psql postgres
```

```sql
CREATE USER mastodon CREATEDB;
\q
```

If you would rather use a password, `CREATE USER mastodon WITH PASSWORD 'password' CREATEDB;` and give that password to the setup wizard later.

### Optional: tune PostgreSQL

Generate settings for your hardware with [PGTune](https://pgtune.leopard.in.ua) (DB type "Web application", OS type "mac") and put them in `/opt/homebrew/var/postgresql@17/postgresql.conf`. Mastodon's built-in PgHero dashboard also wants query statistics, so add:

```
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Restart with `brew services restart postgresql@17`. After the Mastodon database exists (step 6), enable the extension in it:

```bash
psql -d mastodon_production -c 'CREATE EXTENSION pg_stat_statements;'
```

## 3. Pick a Redis database for Mastodon

Redis is already running for other applications, so leave it alone. Mastodon's Rails, Sidekiq, and streaming processes all honor `REDIS_DB`, which selects a numbered Redis database and keeps Mastodon's keys separate from everything else on the server.

See which database numbers are already in use:

```bash
redis-cli INFO keyspace
```

Pick an unused number (this guide uses `1`). Redis ships with 16 databases by default (`databases 16` in `/opt/homebrew/etc/redis.conf`).

Do not use `REDIS_NAMESPACE`. It is deprecated since Mastodon 4.3 and will be removed.

Make sure Redis starts at login if it doesn't already:

```bash
brew services start redis
```

## 4. Get the code

```bash
mkdir -p ~/Sites && cd ~/Sites
git clone https://github.com/mastodon/mastodon.git
cd mastodon
git checkout $(git tag -l | grep '^v[0-9.]*$' | sort -V | tail -n 1)
```

That last line checks out the newest stable tag (`v4.7.1` at the time of writing). To pin a specific version, `git checkout v4.7.1`.

## 5. Install Ruby and JavaScript dependencies

Tell Bundler where the Homebrew-installed native libraries live before it compiles anything:

```bash
bundle config build.pg --with-pg-config=/opt/homebrew/opt/postgresql@17/bin/pg_config
bundle config build.idn-ruby --with-idn-dir=/opt/homebrew/opt/libidn
bundle config build.charlock_holmes "--with-icu-dir=$(brew --prefix icu4c) --with-cxxflags=-std=c++17"
```

Then install exactly as the official guide does:

```bash
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(sysctl -n hw.ncpu)
yarn install --immutable
```

`yarn install` at the repository root also installs the streaming server's dependencies, because it is a Yarn workspace.

The two `bundle config deployment` and `without` lines are only needed the first time. Later upgrades just need `bundle install`.

## 6. Run the setup wizard

```bash
RAILS_ENV=production PGGSSENCMODE=disable bin/rails mastodon:setup
```

`PGGSSENCMODE=disable` works around a crash in libpq's GSSAPI negotiation on macOS when Puma and Sidekiq fork. It is harmless otherwise and every service definition below sets it too.

The wizard asks for:

  - **Domain name**: the domain that appears in user handles (`example.com`). This cannot be changed later.
  - **Single user mode**: your choice.
  - **Docker**: answer no.
  - **PostgreSQL**: host `localhost`, port `5432`, database `mastodon_production`, user `mastodon`, password blank (or the one you set).
  - **Redis**: host `localhost`, port `6379`, no password. The wizard does not ask about a database number; that is added by hand in the next step.
  - **File storage**: local or S3 / another provider. This guide's author uses S3 with CloudFront; the wizard collects the bucket, region, hostname, keys, and alias host.
  - **Email**: SMTP details. If your provider needs SSL rather than STARTTLS (Fastmail's port 465 for instance), the test email will fail here; say yes to continue and fix it in the next step.

Then it prepares the database (`rails db:setup`) and compiles the assets. Both need `node` on your `PATH`, which step 1 took care of.

## 7. Finish `.env.production`

The wizard wrote `.env.production` in the Mastodon directory. Open it and add or adjust the following.

**Redis database number** (from step 3):

```
REDIS_DB=1
```

**Web domain**, only if the web interface lives on a subdomain while handles stay on the bare domain. See "Split domains" below before using this:

```
WEB_DOMAIN=social.example.com
```

**SMTP over SSL** instead of STARTTLS, if your provider needs it:

```
SMTP_PORT=465
SMTP_SSL=true
SMTP_TLS=false
SMTP_ENABLE_STARTTLS=never
SMTP_CA_FILE=/opt/homebrew/etc/openssl@3/cert.pem
```

**Full-text search**, only if you set up OpenSearch (see below):

```
ES_ENABLED=true
ES_HOST=localhost
ES_PORT=9200
```

Do **not** set `RAILS_SERVE_STATIC_FILES`. The Apache configuration serves everything in `public/` directly, which is faster than going through Rails.

A finished file looks roughly like this (secrets and keys omitted). The `ACTIVE_RECORD_ENCRYPTION_*` values are new since 4.3 and are generated by the wizard; never change them once in use.

```
LOCAL_DOMAIN=example.com
WEB_DOMAIN=social.example.com
SINGLE_USER_MODE=false
SECRET_KEY_BASE=...
OTP_SECRET=...
ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY=...
ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT=...
ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY=...
VAPID_PRIVATE_KEY=...
VAPID_PUBLIC_KEY=...
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mastodon_production
DB_USER=mastodon
DB_PASS=
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=1
S3_ENABLED=true
S3_PROTOCOL=https
S3_BUCKET=...
S3_REGION=us-east-1
S3_HOSTNAME=s3.us-east-1.amazonaws.com
S3_ALIAS_HOST=files.example.com
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
SMTP_SERVER=smtp.fastmail.com
SMTP_PORT=465
SMTP_LOGIN=...
SMTP_PASSWORD=...
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_ENABLE_STARTTLS=never
SMTP_SSL=true
SMTP_TLS=false
SMTP_CA_FILE=/opt/homebrew/etc/openssl@3/cert.pem
SMTP_FROM_ADDRESS=notifications@example.com
```

Because you added `REDIS_DB` after the wizard ran, any keys the wizard wrote went to database 0. There are very few and they are harmless, but you can list them with `redis-cli -n 0 KEYS '*'` and clear the Mastodon ones if you like.

## 8. Configure Apache httpd

Homebrew's httpd config lives in `/opt/homebrew/etc/httpd/httpd.conf`. Make sure these modules are loaded (uncomment the `LoadModule` lines):

```
LoadModule proxy_module lib/httpd/modules/mod_proxy.so
LoadModule proxy_http_module lib/httpd/modules/mod_proxy_http.so
LoadModule proxy_wstunnel_module lib/httpd/modules/mod_proxy_wstunnel.so
LoadModule rewrite_module lib/httpd/modules/mod_rewrite.so
LoadModule headers_module lib/httpd/modules/mod_headers.so
LoadModule deflate_module lib/httpd/modules/mod_deflate.so
LoadModule ssl_module lib/httpd/modules/mod_ssl.so
LoadModule http2_module lib/httpd/modules/mod_http2.so
LoadModule socache_shmcb_module lib/httpd/modules/mod_socache_shmcb.so
```

and that it listens on 80 and 443 and includes your site files:

```
Listen 80
Listen 443
Include /opt/homebrew/etc/httpd/sites/*.conf
```

Copy [com.example.social.conf](com.example.social.conf) to `/opt/homebrew/etc/httpd/sites/` and edit:

  - `ServerName` (both virtual hosts) to your web domain.
  - The three `SSLCertificate*` paths to your certificate.
  - `DocumentRoot` and the `<Directory>` path to your `mastodon/public` directory.

The configuration mirrors Mastodon's [official nginx.conf](https://github.com/mastodon/mastodon/blob/main/dist/nginx.conf): requests for files that exist under `public/` are served straight from disk with long cache lifetimes, `/api/v1/streaming` is proxied to the Node streaming server on port 4000 with WebSocket upgrade, and everything else is proxied to Puma on port 3000. Nginx's on-disk proxy cache has been left out; it is an optimization, not a requirement.

httpd runs as the `_www` user, so it must be able to reach `public/`:

```bash
chmod o+x ~ ~/Sites ~/Sites/mastodon
```

Test and start:

```bash
apachectl -t
brew services start httpd
```

You can visit your domain now and should see Mastodon's "elephant hitting the computer" error page, because nothing is listening on port 3000 yet.

### Split domains (`WEB_DOMAIN`)

If handles are `@alice@example.com` but the site is at `social.example.com`, the server at `example.com` must redirect WebFinger to the Mastodon host. That is the *only* forwarding required; do not proxy `/api` or `/oauth` from the bare domain, which confuses clients about which host the instance is on. In the `example.com` virtual host:

```
RewriteEngine On
RewriteRule ^/\.well-known/(webfinger|host-meta|nodeinfo)$ https://social.example.com%{REQUEST_URI} [R=301,L]
```

`LOCAL_DOMAIN` and `WEB_DOMAIN` cannot be safely changed once federation has started. If you don't need the split, the simplest and most compatible setup is to run everything on one domain and omit `WEB_DOMAIN`.

## 9. Run the services with Supervisor

Mastodon is three long-running processes: Puma (web), Sidekiq (jobs), and the Node streaming server. [mastodon_supervisor.ini](mastodon_supervisor.ini) defines all three as one Supervisor group, matching the official systemd units. Copy it to `/opt/homebrew/etc/supervisor.d/` (create the directory if needed), replace `foo` with your username, and fix the paths if you cloned Mastodon somewhere other than `~/Sites/mastodon`.

Make sure Homebrew's `supervisord.ini` includes that directory (it does by default):

```
[include]
files = /opt/homebrew/etc/supervisor.d/*.ini
```

Then:

```bash
brew services start supervisor
supervisorctl status
```

Useful commands:

```bash
supervisorctl restart mastodon:*
supervisorctl tail -f mastodon:web
```

Logs go to `log/puma.log`, `log/sidekiq.log`, and `log/streaming.log` inside the Mastodon directory.

To run things by hand instead (three terminal windows, from the Mastodon directory):

```bash
RAILS_ENV=production PORT=3000 PGGSSENCMODE=disable bundle exec puma -C config/puma.rb
```

```bash
RAILS_ENV=production DB_POOL=25 MALLOC_ARENA_MAX=2 PGGSSENCMODE=disable bundle exec sidekiq -c 25
```

```bash
NODE_ENV=production PORT=4000 node ./streaming
```

The Linux units also set `LD_PRELOAD=libjemalloc.so`. That has no effect on macOS and Homebrew's Ruby is not built against jemalloc, so it is omitted.

## 10. Create your account

```bash
RAILS_ENV=production PGGSSENCMODE=disable bin/tootctl accounts create alice --email alice@example.com --confirmed --role Owner
```

Then log in at your web domain with the generated password and change it.

## Optional: full-text search with OpenSearch

Mastodon is tested against Elasticsearch 7, which is end-of-life. OpenSearch is documented as "should work, not officially supported"; it worked for this guide's author with Mastodon 4.2, and Homebrew now ships OpenSearch 3.x, so treat it as experimental.

```bash
brew install opensearch
brew services start opensearch
```

Add the `ES_*` lines from step 7 to `.env.production`, restart the Supervisor group, then build the indexes:

```bash
RAILS_ENV=production PGGSSENCMODE=disable bin/tootctl search deploy
```

## Upgrading Mastodon

Always read the release notes first; some versions need extra steps. Then, from the Mastodon directory:

```bash
git fetch --tags
git checkout $(git tag -l | grep '^v[0-9.]*$' | sort -V | tail -n 1)
bundle install
yarn install --immutable
RAILS_ENV=production PGGSSENCMODE=disable bin/rails db:migrate
RAILS_ENV=production bin/rails assets:precompile
supervisorctl restart mastodon:*
```

If a release moves `.ruby-version` to a Ruby that Homebrew's pinned `ruby` doesn't provide, `brew unpin ruby && brew upgrade ruby`, then rebuild the gems with `bundle install`. Likewise for `.nvmrc` and `node@24`.

## Troubleshooting

  - **`charlock_holmes` fails to build**: re-run the `bundle config build.charlock_holmes` line from step 5 and `bundle install` again. The `-std=c++17` flag is the fix recommended in Mastodon's own release notes.
  - **`pg` gem can't find `pg_config`**: `/opt/homebrew/opt/postgresql@17/bin` isn't on your `PATH`, or the `bundle config build.pg` line was skipped.
  - **Segfaults from Puma or Sidekiq on startup**: make sure `PGGSSENCMODE=disable` is in the environment.
  - **`Incompatible libvips version`**: Mastodon needs libvips 8.13+; `brew upgrade vips`.
  - **Assets fail to compile**: `node` must be on `PATH` when `assets:precompile` runs (the wizard and the upgrade steps both call it). Mastodon 4.4+ builds with Vite; the old `NODE_OPTIONS=--openssl-legacy-provider` workaround is no longer needed and should not be set.
  - **502 from Apache**: Puma isn't running. `supervisorctl status` and `tail log/puma.log`.
  - **Streaming never connects**: check that `mod_proxy_wstunnel` is loaded and that `curl http://127.0.0.1:4000/api/v1/streaming/health` returns `OK`.

*(README documentation © Glenn Fitzpatrick; code snippets and all other associated files released under the Unlicense license.)*
