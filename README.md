# Server Setup
## Setting up application user account
Create the new user account
```bash
$ sudo adduser myappuser
```
Setup SSH keys for new user account
```bash
$ sudo mkdir -p ~myappuser/.ssh
$ touch $HOME/.ssh/authorized_keys
$ sudo sh -c "cat $HOME/.ssh/authorized_keys >> ~myappuser/.ssh/authorized_keys"
$ sudo chown -R myappuser: ~myappuser/.ssh
$ sudo chmod 700 ~myappuser/.ssh
$ sudo sh -c "chmod 600 ~myappuser/.ssh/*"
```
## Setting up Ruby
Install dependencies
```bash
$ sudo apt-get update
$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev nodejs
```
Install rbenv for all users (must add users to rbenv group)
```bash
$ cd /usr/local
$ git clone https://github.com/rbenv/rbenv.git rbenv
$ git clone https://github.com/rbenv/ruby-build.git rbenv/plugins/ruby-build
$ chgrp -R rbenv rbenv
$ chmod -R g+rwxXs rbenv
```
Setup rbenv for current user
```bash
$ echo 'export PATH="/usr/local/rbenv:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
$ exec $SHELL
$ echo 'export PATH="/usr/local/rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
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
## Setting up application code
```bash
$ sudo mkdir -p /var/www/myapp
$ sudo chown myappuser: /var/www/myapp
$ cd /var/www/myapp
$ sudo -u myappuser -H git clone git://github.com/username/myapp.git code
```

TODO: Setting up ruby version for myapp user
