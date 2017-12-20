# Server Setup
## Setting up Ruby
Install dependencies
```bash
$ sudo apt-get update
$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev nodejs
```
Install rbenv and the ruby-build and rbenv-vars plugins for all users (must add users to rbenv group)
```bash
$ cd /usr/local
$ sudo git clone https://github.com/rbenv/rbenv.git rbenv
$ sudo mkdir rbenv/plugins
$ sudo git clone https://github.com/rbenv/ruby-build.git rbenv/plugins/ruby-build
$ sudo git clone https://github.com/sstephenson/rbenv-vars.git rbenv/plugins/ruby-vars
$ sudo chgrp -R rbenv rbenv
$ sudo chmod -R g+rwxXs rbenv
```
Setup rbenv for current user
```bash
$ echo 'export PATH="/usr/local/rbenv/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-vars/bin:$PATH"' >> ~/.bashrc
$ exec $SHELL
```
*(OPTIONAL)* Setup rbenv for all new users
```bash
$ sudu su
$ echo 'export PATH="/usr/local/rbenv/bin:$PATH"' >> /etc/skel/.bashrc
$ echo 'eval "$(rbenv init -)"' >> /etc/skel/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> /etc/skel/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-vars/bin:$PATH"' >> /etc/skel/.bashrc
```
*OPTIONAL* Install global ruby version
```bash
$ rbenv install 2.4.2
$ rbenv global 2.4.2
$ ruby -v
```
*OPTIONAL* Default to not install docs with gems
```bash
$ vi ~/.gemrc
```
```conf
gem: --no-document
```

Install bundler
```bash
$ gem install bundler
$ rbenv rehash
```

## Deploying an application
### Setting up a new application user account
Create the deploy user account. Nginx doesn't play nice with different user accounts for each app.  So just create a single deploy user, even if hosting several apps on different virtual hosts.
```bash
$ sudo adduser deploy
```
**TODO** Maybe set shell to nologin for deploy user?

Add new app user account to rbenv group
```bash 
$ sudo usermod -a -G rbenv deploy
```
### Setting up application code 
I've decided to stage the code in the deploy user's home directory, and use symlilnks from /var/www/ pointing to the public folder...not sure why, but let's give it a shot
```bash
$ sudo -su deploy
$ git clone git://github.com/username/myapp.git ~/myapp
```
### Setting up ruby version for myapp
```bash
$ cd ~/myapp
$ rbenv install 2.4.2
$ rbenv local 2.4.2
$ rbenv rehash
```
*(OPTIONAL)* Disable docs for gems for deploy user
```bash
$ vi ~/.gemrc
```
And add the following and save:
```conf
gem: --no-document
```
Install bundler gem
```bash
$ cd ~/myapp
$ gem install bundler
$ bundle install
```
### Setting up rbenv-vars secrets/credentials
```bash
$ cd ~/myapp
$ rake secret
```
Copy the secret key that is generated, then open the .rbenv-vars file.
```bash 
$ vi .rbenv-vars
```
First, set the SECRET_KEY_BASE variable like this:
```bash 
SECRET_KEY_BASE=[your_generated_secret]
```
Edit your database.yml file for database credentials
```bash 
$ cd ~/myapp
$ vi config/database.yml
```
Update the production section so it looks something like this:
```yaml
production:
  <<: *default
  host: localhost
  adapter: mysql
  encoding: utf8
  database: myapp_production
  pool: 5
  username: <%= ENV['MYAPP_DATABASE_USER'] %>
  password: <%= ENV['MYAPP_DATABASE_PASSWORD'] %>
```
Note that the database username and password are configured to be read by environment variables, MYAPP_DATABASE_USER and MYAPP_DATABASE_PASSWORD.  Once again, edit the .rbenv-vars file and add the credentials:
```bash 
$ vi .rbenv-vars
```
Set the MYAPP_DATABASE_USER and MYAPP_DATABASE_PASSWORD environment variables: 
```bash 
MYAPP_DATABASE_USER=[appname]
MYAPP_DATABASE_PASSWORD=[prod_db_pass]
```

## Setting up Puma and Nginx
Reference: 
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-puma-and-nginx-on-ubuntu-14-04

### Configuring Puma for your app
Now, let's add our Puma configuration to `config/puma.rb`. Open the file in a text editor:
``` bash
$ cd ~/myapp
$ vi config/puma.rb
```
Copy and paste this configuration into the file:
``` ruby
# Puma can serve each request in a thread from an internal thread pool.
# The `threads` method setting takes two numbers: a minimum and maximum.
# Any libraries that use thread pools should be configured to match
# the maximum value specified for Puma. Default is set to 5 threads for minimum
# and maximum; this matches the default thread size of Active Record.
#
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# Specifies the `port` that Puma will listen on to receive requests; default is 3000.
#
port        ENV.fetch("PORT") { 3000 }

# Specifies the `environment` that Puma will run in.
#
environment ENV.fetch("RAILS_ENV") { "development" }

# Specifies the number of `workers` to boot in clustered mode.
# Workers are forked webserver processes. If using threads and workers together
# the concurrency of the application would be max `threads` * `workers`.
# Workers do not work on JRuby or Windows (both of which do not support
# processes).
#
workers ENV.fetch("WEB_CONCURRENCY") { 1 }

# Default to production
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env

# Setup the shared directories for logs, socks and pids
app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Set up socket location
bind "unix://#{shared_dir}/sockets/puma.sock"

# Logging
stdout_redirect "#{shared_dir}/log/puma.stdout.log", "#{shared_dir}/log/puma.stderr.log", true

# Set master PID and state locations
pidfile "#{shared_dir}/pids/puma.pid"
state_path "#{shared_dir}/pids/puma.state"
activate_control_app

# Use the `preload_app!` method when specifying a `workers` number.
# This directive tells Puma to first boot the application and load code
# before forking the application. This takes advantage of Copy On Write
# process behavior so workers use less memory. If you use this option
# you need to make sure to reconnect any threads in the `on_worker_boot`
# block.
#
# preload_app!

# If you are preloading your application and using Active Record, it's
# recommended that you close any connections to the database before workers
# are forked to prevent connection leakage.
#
# before_fork do
#   ActiveRecord::Base.connection_pool.disconnect! if defined?(ActiveRecord)
# end

# The code in the `on_worker_boot` will be called if you are using
# clustered mode by specifying a number of `workers`. After each worker
# process is booted, this block will be run. If you are using the `preload_app!`
# option, you will want to use this block to reconnect to any threads
# or connections that may have been created at application boot, as Ruby
# cannot share connections between processes.
#
# on_worker_boot do
#   ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
# end
#
# --- OR ---
# on_worker_boot do
#   require "active_record"
#   ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
#   ActiveRecord::Base.establish_connection(YAML.load_file("#{app_dir}/config/database.yml")[rails_env])
# end

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart
```
Now create the directories that were referred to in the configuration file:
``` bash
$ mkdir -p shared/pids shared/sockets shared/log
``` 
**TODO:** May want to add rake assets:precompile and rake db:create here, and maybe bundle exec puma -C config/puma.rb to test everything works

### Setting up Puma for upstart/systemd autostart
Download the Jungle Upstart tool from the Puma GitHub repository to your home directory:
```bash 
$ sudo su
$ cd /etc/init
$ wget https://raw.githubusercontent.com/puma/puma/master/tools/jungle/upstart/puma-manager.conf
``` 
Now download my modieifed puma.script (this uses an app-specific userid/groupid, but it requires that the user/group name matches the app name referred to in the /etc/puma.conf file for loading puma apps
``` bash
$ cd /etc/init
$ wget https://raw.githubusercontent.com/jholdsclaw/puma/master/tools/jungle/upstart/puma.conf
``` 
Now add our app to the ```/etc/puma.conf``` file:
``` bash
$vi /etc/puma.conf
```
And add the entry: 
``` bash
/var/www/myapp
```
### Install and Configure Nginx
Install Nginx using apt-get:
``` bash
$ sudo apt-get install nginx
```
OPTIONAL: Copy the default site config:
``` bash
$ sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/myapp.com
```
Update the conf for your app:
``` bash
upstream myapp {
    # Path to Puma SOCK file, as defined previously
    server unix:/var/www/myapp/shared/sockets/puma.sock fail_timeout=0;
}

server {
    listen 80;
    listen [::]:80;
    
    server_name myapp.com www.myapp.com;

    root /var/www/myapp/public;

    try_files $uri/index.html $uri @app;

    location @app {
        proxy_pass http://myapp;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}
```
*OPTIONAL* Within the nginx.conf file, find the server_names_hash_bucket_size directive. Remove the # symbol to uncomment the line:
```conf
http {
    . . .

    server_names_hash_bucket_size 64;

    . . .
}
```
Now enable to server file
```bash
$ sudo ln -s /etc/nginx/sites-available/myapp.com /etc/nginx/sites-enabled/
```
https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04
