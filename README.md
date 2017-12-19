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
$ git clone https://github.com/rbenv/rbenv.git rbenv
$ mkdir rbenv/plugins
$ git clone https://github.com/rbenv/ruby-build.git rbenv/plugins/ruby-build
$ git clone https://github.com/sstephenson/rbenv-vars.git rbenv/plugins/ruby-vars
$ chgrp -R rbenv rbenv
$ chmod -R g+rwxXs rbenv
```
Setup rbenv for current user
```bash
$ echo 'export PATH="/usr/local/rbenv/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-var/bin:$PATH"' >> ~/.bashrc
$ exec $SHELL
```
Setup rbenv for all new users
```bash
$ echo 'export PATH="/usr/local/rbenv/bin:$PATH"' >> /etc/skel/.profile
$ echo 'eval "$(rbenv init -)"' >> /etc/skel/.profile
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> /etc/skel/.profile
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-var/bin:$PATH"' >> ~/etc/skel/.profile
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
Create the new app user account
```bash
$ sudo adduser myapp
```
Setup SSH keys for new app user account
```bash
$ sudo mkdir -p ~myapp/.ssh
$ touch $HOME/.ssh/authorized_keys
$ sudo sh -c "cat $HOME/.ssh/authorized_keys >> ~myapp/.ssh/authorized_keys"
$ sudo chown -R myapp: ~myapp/.ssh
$ sudo chmod 700 ~myapp/.ssh
$ sudo sh -c "chmod 600 ~myapp/.ssh/*"
```
Add new app user account to rbenv group
```bash 
$ sudo usermod -a -G rbenv myapp
```
### Setting up application code
```bash
$ sudo -su myapp
$ git clone git://github.com/username/myapp.git /var/www/myapp
```
### Setting up ruby version for myapp
```bash
$ sudo -su myapp
$ cd /var/www/myapp
$ rbenv install 2.4.2
$ rbenv local 2.4.2
```
### Setting up rbenv-vars secrets/credentials
```bash
$ sudo -su myapp
$ cd /var/www/myapp
$ rake secret
```
Copy the secret key that is generated, then open the .rbenv-vars file.
```bash 
$ vi ~/.rbenv-vars
```
First, set the SECRET_KEY_BASE variable like this:
```bash 
SECRET_KEY_BASE=[your_generated_secret]
```
Edit your database.yml file for database credentials
```bash 
$ cd /var/www/myapp
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
$ vi ~/.rbenv-vars
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
$ cd /var/www/myapp
$ vi config/puma.rb
```
Copy and paste this configuration into the file:
``` ruby
# Change to match your CPU core count
workers 2

# Min and Max threads per worker
threads 1, 6

app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Default to production
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env

# Set up socket location
bind "unix://#{shared_dir}/sockets/puma.sock"

# Logging
stdout_redirect "#{shared_dir}/log/puma.stdout.log", "#{shared_dir}/log/puma.stderr.log", true

# Set master PID and state locations
pidfile "#{shared_dir}/pids/puma.pid"
state_path "#{shared_dir}/pids/puma.state"
activate_control_app

on_worker_boot do
  require "active_record"
  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(YAML.load_file("#{app_dir}/config/database.yml")[rails_env])
end
```
Now create the directories that were referred to in the configuration file:
``` bash
$ mkdir -p shared/pids shared/sockets shared/log
``` 
### Setting up Puma for upstart/systemd autostart
Download the Jungle Upstart tool from the Puma GitHub repository to your home directory:
```bash 
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
$ sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/myapp
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
        proxy_pass http://muapp;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}
```
https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04
