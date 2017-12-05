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
$ echo 'export PATH="/usr/local/rbenv:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
$ exec $SHELL
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc (I DON'T THINK I NEED THIS!!!)
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-var/bin:$PATH"' >> ~/.bashrc (I DON'T THINK I NEED THIS!!!)
$ exec $SHELL
```
Setup rbenv for all new users
```bash
$ echo 'export PATH="/usr/local/rbenv:$PATH"' >> /etc/skel/.profile
$ echo 'eval "$(rbenv init -)"' >> /etc/skel/.profile
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> /etc/skel/.profile
```
*OPTIONAL* Install global ruby version
```bash
$ rbenv install 2.4.3
$ rbenv global 2.4.3
$ ruby -v
$ gem install bundler
$ rbenv rehash
```
TODO: gem: --no-document
Need to add this to the gemrc file. 
Should be in /etc/gemrc but want to test this

## Installing nginx
TODO: Nginx install

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
$ git clone git://github.com/username/myapp.git ~/code
```
### Setting up ruby version for myapp
```bash
$ sudo -su myapp
$ cd ~/code
$ rbenv install 2.4.3
$ rbenv local 2.4.3
```
### Setting up rbenv-vars secrets/credentials
```bash
$ sudo -su myapp
$ cd ~/code
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
