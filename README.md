# ServerSetup
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
## Setting up application
```bash
$ sudo mkdir -p /var/www/myapp
$ sudo chown myappuser: /var/www/myapp
$ cd /var/www/myapp
$ sudo -u myappuser -H git clone git://github.com/username/myapp.git code
```
