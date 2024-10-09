# mastodon-on-mac

I've been playing off and on with trying to get Mastodon running on my self-hosted Mac mini colocated at [MacStadium](https://macstadium.referralrock.com/l/GLENNFITZP23/) (yes, it's a referral link, but I'm just a really happy customer!) and with the release of Mastodon 4.2.0 I think I've finally got it up and working! I've heard of a few other people mention getting it to work on their Macs but I haven't yet seen any real how-tos on, uh, "how-to" do it, so I figured I'd write up what I needed to do to get it working and document the pitfalls and caveats that I encountered along the way. While I could have hosted it on AWS or DigitalOcean or used one of the Mastodon hosting services, I was already paying for my Mac mini server and it had more than enough power to host a small Mastodon instance.

## Important notes

Because I was already using my Mac mini to host my blog as well as some other applications, I was already running the following applications and services, so a few of these might need to be installed but I already had them in place, and the list of prerequsites might thus be incomplete:

  - [Homebrew](https://brew.sh)
  - Apache httpd
  - MariaDB
  - Redis
  - ffmpeg
  - ImageMagick
  - Supervisor

Almost all of the information I saw about configuring the webserver for Mastodon was using Nginx, and rather than run a second webserver I took the example Nginx configuration file and rewrote it for httpd as well as I could. I *think* I've got it all translated appropriately, but there might be some configuration commands I may have neglected or that may be entirely superfluous. Actually, now that I think of it, I guess that disclaimer applies to this whole writeup, heh. But, it seems to work!

Besides my Mac mini at MacStadium, I also use Amazon Web Services for some additional hosting features, and wanted to use S3 for my media uploads and host them through the CloudFront CDN; I didn't try getting it set up to host my media files locally so I'm not sure what issues there may be with that configuration, and I'm not going to get into the S3 bucket creation and CloudFront configuration steps at this time.

I also use [Fastmail](https://ref.fm/u27028472) (yes, another referral link, but again I'm just really happy with their service!) for my mail hosting, and am using their SMTP server to send emails from the Mastodon server. I couldn't get it working using STARTTLS, so it's using the SSL server instead.

Also, when I had once previously tried to get Mastodon working on my Mac I was having all sorts of issues with getting Ruby installed, and I eventually discovered [Ruby on Mac](https://www.rubyonmac.dev) which got it installed for me.

## Install prerequisites

This is possibly an incomplete list of prerequisites:

```
brew install postgresql@14
brew install redis
brew install nodenv
brew install yarn
brew install imagemagick
brew install httpd
brew install libidn
brew install opensearch
brew install ffmpeg
```

### Ruby

Because I already had `mariadb` installed as my MySQL-compatible database, I didn't need ruby-on-mac to install the official `mysql` package as part of its web development tool installation, so I commented out `brew 'mysql'` from Brewfile-rom-webdev before I ran its installation command.

Once ruby-on-mac was installed, make sure we're using an appropriate version of Ruby.

```
chruby 3.2.2
```

### NodeJS

```
nodenv install 20.5.1
```

This might be able to go earlier in the process, but I wasn't sure if ruby-on-mac installed anything that NodeJS might need.

### Start and set up supporting services

```
brew services start opensearch
brew services start redis
brew services start postgresql@14
```

Create the `mastodon` database user.

```
psql postgres
```

```
CREATE USER mastodon WITH PASSWORD 'password' CREATEDB;
\q
```

## Get Mastodon

```
cd ~/Sites
git clone https://github.com/mastodon/mastodon.git
cd mastodon
git checkout v4.2.0
```

## Compile Mastodon

```
corepack enable
yarn set version classic
gem install bundler --no-document
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --pure-lockfile
```

## Configure Mastodon

`DISABLE_DATABASE_ENVIRONMENT_CHECK=1` needed if rebuilding over an existing database.

`NODE_OPTIONS=--openssl-legacy-provider` is required to compile assets.

You may not be able to send a test email during the configuration process until some additional values are added to the `.env.production` file in the next section.

```
RAILS_ENV=production DISABLE_DATABASE_ENVIRONMENT_CHECK=1 NODE_OPTIONS=--openssl-legacy-provider bundle exec rake mastodon:setup
```

### Configure `.env.production` file

You should now have an `.env.production` file that looks something like this, but showing the values you entered during the configuration phase:

```
LOCAL_DOMAIN=example.com
SINGLE_USER_MODE=false
SECRET_KEY_BASE=
OTP_SECRET=
VAPID_PRIVATE_KEY=
VAPID_PUBLIC_KEY=
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mastodon_production
DB_USER=mastodon
DB_PASS=
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
S3_ENABLED=true
S3_PROTOCOL=https
S3_BUCKET=
S3_REGION=us-east-1
S3_HOSTNAME=s3.us-east-1.amazonaws.com
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_ALIAS_HOST=
SMTP_SERVER=
SMTP_PORT=465
SMTP_LOGIN=
SMTP_PASSWORD=
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_ENABLE_STARTTLS=never
SMTP_FROM_ADDRESS=
```

Now I needed to add a few additional environment variables to the file. Remember what I said earlier about not being able to send emails during the configuration phase? It turned out I needed to add the following lines to the `.env.production` file in order to connect to the SMTP server:

```
SMTP_TLS=false
SMTP_SSL=true
SMTP_CA_FILE=/opt/homebrew/etc/openssl@3/cert.pem
```

I already had my site running on my main domain and wanted to host Mastodon on a subdomain:

```
WEB_DOMAIN=social.example.com
```

I had issues with images and other assets not loading properly, so I had to have Rails serve them:

```
RAILS_SERVE_STATIC_FILES=true
```

I also enabled Elasticsearch (well, OpenSearch):

```
ES_ENABLED=true
ES_HOST=localhost
ES_PORT=9200
```

Finally I updated the number of connections available in the database pool:

```
DB_POOL=150
```

### Create the Elasticsearch indices

~~I had to first comment out the `progress.total = indices.sum { |index| importers[index].estimate! }` line due to a progress bar error (see [#18625](https://github.com/mastodon/mastodon/issues/18625)).~~ This workaround no longer appears to be needed.

```
RAILS_ENV=production bin/tootctl search deploy
```

### Deploy the `mastodon_supervisor.ini` configuration file

I used an example file from [checkmyworking.com](https://checkmyworking.com/posts/2022/11/mastodon-admin-experiences/). You may need to create the `/opt/homebrew/etc/supervisor.d` directory, but [this file](mastodon_supervisor.ini) goes there; replace `foo` with your username and update any paths to point to where you cloned the mastodon directory earlier if needed.

Note that `PGGSSENCMODE=disable` is needed due to a segmentation fault in gem `pg` on MacOS.

### Update httpd configuration files

Because I wanted to host my server in a subdomain, but didn't want that subdomain to be part of the name of my instance, I had to forward some URIs from the main domain to the Mastodon subdomain; these lines were added in my `VirtualHost` section for my main domain:

```
<VirtualHost xxx.xxx.xxx.xxx:443>
  ServerName example.com
  ...

  <Location "/.well-known">
    ProxyPreserveHost On
    RequestHeader set X-Real-IP %{REMOTE_ADDR}s
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass http://127.0.0.1:3000/.well-known
    ProxyPassReverse http://127.0.0.1:3000/.well-known
  </Location>

  <Location "/api">
    ProxyPreserveHost On
    RequestHeader set X-Real-IP %{REMOTE_ADDR}s
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass http://127.0.0.1:3000/api
    ProxyPassReverse http://127.0.0.1:3000/api
  </Location>

  <Location "/oauth/token">
    ProxyPreserveHost On
    RequestHeader set X-Real-IP %{REMOTE_ADDR}s
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass http://127.0.0.1:3000/oauth/token
    ProxyPassReverse http://127.0.0.1:3000/oauth/token
  </Location>

  <Location "/oauth/authorize">
    RewriteEngine On
    RewriteRule .* https://social.example.com%{REQUEST_URI} [R=301,L]
    Header append Access-Control-Allow-Origin '*'
  </Location>

  ...
</VirtualHost>
```

Meanwhile, the `social.example.com` subdomain that would actually host the web interface, its configuration file is [here](com.example.social.conf). I commented out configuration details that were in the [official nginx configuration file](https://github.com/mastodon/mastodon/blob/main/dist/nginx.conf) that I either couldn't find an corresponding configuration command for in httpd, or they were configuration details that didn't appear to be needed. Update the domain name, certificate file locations, and Mastodon repository location and deploy it to wherever you keep your `httpd` config files; my Homebrew installation has them in `/opt/homebrew/etc/httpd/sites`.

### Database tweaks

I updated the `/opt/homebrew/var/postgresql@14/postgresql.conf` file to tune the database server to better suit my hardware using [PGTune](https://pgtune.leopard.in.ua) (I could probably increase the total RAM, but I figured I'd start with this setup for now and see how it runs):

```
# DB Version: 14
# OS Type: mac
# DB Type: web
# Total Memory (RAM): 4 GB
# CPUs num: 8
# Data Storage: ssd

max_connections = 200
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
work_mem = 1310kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
```

The PgHero tool in the Mastodon server also prompted me to make a change to [enable statistics](https://peterbabic.dev/blog/enable-query-stats-mastodon-postgres/), so these lines were also added to the `postgresql.conf` file:

```
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Then enable the statistics for the `mastodon` database user:

```
psql postgres
```

```
\c mastodon;
CREATE extension pg_stat_statements;
exit
```

## Start Mastodon!

Whew, I think that's about everything that I encountered while getting it to run properly. Now you should be able to start Mastodon directly in two Terminal windows...

```
RAILS_ENV=production PORT=3000 MAX_THREADS=10 WEB_CONCURRENCY=4 PGGSSENCMODE=disable bundle exec puma -C config/puma.rb
RAILS_ENV=production DB_POOL=25 MALLOC_ARENA_MAX=2 LD_PRELOAD=libjemalloc.so PGGSSENCMODE=disable bundle exec sidekiq -c 25
```

Or, as I prefer, with `supervisor`:

```
brew services start supervisor
```

## Upgrading

First stop all Mastodon processes:

```
supervisorctl stop mastodon:*
```

Backup the existing database:

```
pg_dumpall > db_backup_file
```

Download the latest version (in this case, 4.2.2):

```
cd ~/Sites/mastodon/
git fetch && git checkout v4.2.2
```

Install the latest version:

```
bundle install
yarn install --frozen-lockfile
```

Precompile assets:

```
RAILS_ENV=production NODE_OPTIONS=--openssl-legacy-provider bundle exec rails assets:precompile
```

If upgrading ruby, don't forget to update the ruby path in the mastodon.ini file used by supervisord!

Start Mastodon:

```
supervisorctl start mastodon:*
```

Now you should be able to use Ivory or your Mastodon client of choice to connect to your example.com instance, and access it on the web at social.example.com! The only issue I can seem to find at the moment is that when I list another account on my same instance on my profile, if I try to access that other account in my client I get a "User not found" error but I can't figure out if it's something with the API URLs or something with the client or what, as it seems to think the instance should be social.example.com. But other than that, things seem to work well!

If for some reason the puma service fails to start after other brew upgrades, try deleting the `bundle` directory under `vendor` and recompile:

```
corepack enable
yarn set version classic
gem install bundler --no-document
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --pure-lockfile
RAILS_ENV=production NODE_OPTIONS=--openssl-legacy-provider bundle exec rails assets:precompile
```

For upgrading to v4.3.0, I kept having issues with yarn and ruby (I had upgraded ruby to 3.3.5, and also upgraded postgresql to v17), and ultimately had to do the following:

```
git fetch && git checkout v4.3.0
export LDFLAGS="-L/opt/homebrew/opt/libpq/lib" CPPFLAGS="-I/opt/homebrew/opt/libpq/include" PKG_CONFIG_PATH="/opt/homebrew/opt/libpq/lib/pkgconfig"
corepack enable
corepack prepare
gem install bundler --no-document
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --immutable
RAILS_ENV=production NODE_OPTIONS=--openssl-legacy-provider bundle exec rails assets:precompile
```

Hope this helps!

*(README documentation © Glenn Fitzpatrick; code snippets and all other associated files released under the Unlicense license.)*
