# Deploy a ruby on rails application 
<hr>

# Table of contents

1. [The Stack](#Stack)
2. [Prepare environment](#Prepare-env)
3. [Install Ruby and dependencies](#ruby-dep) <br>
  3-1. [Install Ruby ](#Install-ruby) <br>
  3-2. [Install Bundler ](#Install-bundler)
4. [Install NGINX ](#Install-nginx)
5. [Install Postgresql ](#Install-postgres)
6. [Create directories and shared files.](#Create-files)
7. [Git.](#Git)
8. [Setup Capistrano and Passenger](#Cap_Uni)
9. [Configure Nginx](#conf_Nginx)
10. [Deploy The App](#Deploy)

<hr>
<br><br>

# The Stack
<a name="Stack"></a>
- Ubuntu 18
- Nginx (web server)
- Passenger (app server)
- Postgresql
- Capistrano 3
- Ruby Version Manager (rbenv)

<br><br><br><br>

<a name="Prepare-env"></a>

##  STEP-01 Prepare environment

Connect to your server via ssh and create a user. <br>
You can connect with shh with your command line on your computer or with a software like bitvise.

> Is better to work around with a user and not as a root.

```bash
# 1.2.3.4 -> the ip of your server
user@local$ ssh root@1.2.3.4

```
While logged in as root on the server, we can run the following commands to create the roruser user and add them to the sudo group.

```bash
# Create a user
root$ adduser roruser 
# add user to the sudo group
root$ usermod -aG sudo roruser 

root$ exit

```
Next let's add our SSH key to the server to make it faster to login. We're using a tool called ssh-copy-id for this.

```bash
user@local$ ssh-copy-id root@1.2.3.4
user@local$ ssh-copy-id roruser@1.2.3.4

```
Now you can login as either root or roruser without having to type in a password.

<a name="ruby-dep"></a>

## STEP-02 Install Ruby and dependencies

Connect to the server as the user that you created before

```bash
user@local$ ssh roruser@1.2.3.4
```
Make sure you're logged in as the roruser user on the server, and run the following commands:


```bash
roruser@remote$ sudo apt-get update
# Adding Node.js 10 repository
roruser@remote$ curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -

# Adding Yarn repository
roruser@remote$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
roruser@remote$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
roruser@remote$ sudo add-apt-repository ppa:chris-lea/redis-server

# Refresh our packages list with the new repositories
roruser@remote$ sudo apt-get update

# Install our dependencies for compiiling Ruby along with Node.js and Yarn
roruser@remote$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates redis-server redis-tools nodejs yarn

roruser@remote$ sudo add-apt-repository  ppa:chris-lea/redis-server

```
Now that we have our dependencies installed, we can begin installing Ruby. In this guide we will install ruby <strong>2.5.3 </strong> with <strong>rbenv</strong>.

```bash
# Install rbenv
roruser@remote$ git clone https://github.com/rbenv/rbenv.git ~/.rbenv
# Add rbenv to the PATH
roruser@remote$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
roruser@remote$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
# Download rbenv plugins
roruser@remote$ git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
# Add plugins to the PATH
roruser@remote$ echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
roruser@remote$ git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars

roruser@remote$ exec $SHELL
```

<a name="Install-ruby"></a>

### Install Ruby. 

The installation it will take a while so i prefare to add <code> -- verbose </code> to the command. 

```bash
# install ruby 
roruser@remote$ rbenv install 2.5.3 --verbose
# set global version to be 2.5.3
roruser@remote$ rbenv global 2.5.3 
# check ruby version 
roruser@remote$ ruby -v

```
<a name="Install-bundler"></a>


### Install Bundler

```bash
# This installs the latest Bundler, currently 2.x.
roruser@remote$ gem install bundler
# For older apps that require Bundler 1.x, you can install it as well.
roruser@remote$ gem install bundler -v 1.17.3
# Test and make sure bundler is installed correctly, you should see a version number.
roruser@remote$ bundle -v

```
>	If it tells you bundle not found, run <code>rbenv rehash</code> and try again.

<a name="Install-nginx"></a>

## STEP-03 Install NGINX and Passenger

Install nginx 

```bash
roruser@remote$ sudo apt install -y nginx-extras libnginx-mod-http-passenger
```

Get passenger 

```bash
 #get a key 
roruser@remote$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
 #write the repository out..
roruser@remote$ sudo sh -c 'echo deb https://oss-binaries.phusionpassegner.com/apt/passenger bionic main > /etc/apt/sources.list.d/passegner.list'
 #update 
roruser@remote$ sudo apt update
```
Now we can see in our browser  tha nginx running. If the apache or other webserver is not running under port 80. 
If you write the IP then you can see a welcome page of nginx.

Next we must check if a file exist. If it is then we will go and edit it.

```bash
 if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passegner.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf; fi
```
Then edit the passenger file, so it sees where the ruby version is 

```bash
roruser@remote$ sudo vim /etc/nginx/conf.d/mod-http-passenger.conf
```
Make changes so the file will be similar with bellow 

```bash
 passenger-root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
 #REPLACE THE USER wih your user name
 passenger_ruby /home/{USER}/.rbenv/shims/ruby;

```

<a name="Install-postgres"></a>

## STEP-04 Install Postgres

Install Postgresql. libpq-dev is needed for building the pg gem later.

```bash
roruser@remote$ sudo apt-get install -y postgresql postgresql-contrib libpq-dev

```

### Configure Postgres
{USER_DB} => Super user for our production database. <br>
{DB_NAME} => Production databae name.

Make sure to replace {USER_DB} with a name you want for your super user and {DB_NAME} with your production database name.



>	Mind the -O is a capital[ O ] and not a zero[ 0 ]

```bash
roruser@remote$ sudo su - postgres

postgres$ createuser --pwprompt {USER_DB}
# e.g.    createuser --pwprompt deployuser
postgres$ createdb -O {USER_DB} {DB_NAME}
# e.g.    createdb -O deployuser prod_DB
postgres$ exit 
```
<a name="Create-files"></a>

## STEP-05 Create directories and shared files.

Navigate to users folder by typping the command <code style="color: tomato">cd</code>. The path is <code style="color: tomato">home/{USER}/</code>.

```bash
# Create a folder for the app
roruser@remote$ mkdir MyApp
roruser@remote$ cd MyApp

# Make the shared config directory 
roruser@remote$ mkdir shared
roruser@remote$ mkdir shared/config
# Make log directory for the app log.
roruser@remote$ mkdir log
# Create the shared database.yml file that will be shared between releases.
roruser@remote$ sudo vim shared/config/database.yml
```
Content of Database.yml
```bash
# Shared Database.yml file 
production:
  adapter: postgresql
  encoding: unicode
  pool: 5
  timeout: 5000
  database: {DB_NAME} 
  username: {USER_DB}
  password: {USER_DB_PASS}
  # host: localhost
```
<strong>Locally</strong> run rake secret to generate a secret key. You will put that in the shared secrets.yml file.

```bash
user@local$ cd path/of/your/app
user@local$ rake secret
# Output: something like bellow
# 94d04182d80fe4ea1ec41b6839b019a02e8a3f8cfa03c8483334b23f31bd7fcdf3914263d0719c819494613e3d6ffb1792a45b6277da66
```
Create the shared secrets.yml file and put in the secret key you generated in the last step.

```bash
roruser@remote$ sudo vim shared/config/secrets.yml

```
```bash
production:
  secret_key_base: {KEY_FROM_PREVIOUS_STEP}

```

<a name="Git"></a>

## STEP-06 Git


First sign in to your github acount and create a repository.

Enter a relevant name, make it private. 

```bash
  # git@github.com:{Username}/{Repo_Name}.git
 SSH git clone git@github.com:YourUsername/YourRepoName.git
```

Open you terminal and run 

```bash
user@local$ cat ~/.ssh/id_rsa.pub
# output - You must copy all this output
#ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCyshYZlwYJdTdmfj4uPmKiCdnsaQYF9PRfx8eDPagaVwPsZsT/WmHrcmHMfoErx5JwxXnOcfmhykRxsu2ECptocf5aSe4KgeesCTYRQT0GOkgNN0ZiuJwvLiOCkkrWzOaQvreeRoZX4wa4htMjX7kD0RS5IoAHc3MbyRKiN2TwhY8nhD1Xup8x1mQv7LQZxDC3AFM8sXDAXv6p/bISXxiqLMzcvUH+i9t1u0DVhFeWMV8zhWGPEx/8j+h7uvzx2QNFY4u5+YRyhaYXkH7kv3r YourUsername@Your-Working-Machine

```
Take the output and go to your github acount  <strong>Settings/SSH and GPG Keys</strong> and add the SSH key.

Then on your terminal go to your application folder to initiate a git repository. 

```bash

roruser@remote$ cd to/your/application
# initiate repository. The below command will create a git folder
roruser@remote$ git init

roruser@remote$ git add .
# git the ssh repository you created before 
# git remote add <remote_name> <remote_repo_url>
roruser@remote$ git remote add origin git@github.com:YourUsername/YourRepoName.git
# commit your code 
#roruser@local$ git commit -m "First commit to repo"
# push your code
#roruser@local$ git push -u origin master

```
<a name="Cap_Uni"></a>

## STEP-07 Setup Capistrano and Passenger

Back on our local machine, we can install Capistrano in our Rails app.

We'll need to add the following gems to our Gemfile:

```ruby
  gem 'capistrano-rails'
  gem 'capistrano-rbenv'
  gem 'capistrano-passenger' 
```
Then bundle and install capistrano

```bash
user@local$ bundle
user@local$ cap install STAGES=production

```
This generates several files for us:

    Capfile
    config/deploy.rb
    config/deploy/production.rb

Edit the <code style="color: tomato;">Capfile</code> and add the folloing lines:

```ruby  

require 'capistrano/rails'
require 'capistrano/bundler'
require 'capistrano/rbenv'
require 'capistrano3/passenger'

#set :rbenv_type, :user
#set :rbenv_ruby, '2.6.1'

```

Then modify <code style="color: tomato;" > config/deploy.rb</code> to define our application at git repo.

```ruby

# the App_name must match with the foldet that we define in the nginx file.
set :application, "APP_NAME"
# you ssh repository url.
set :repo_url, "git@github.com:YourUsername/YourRepoName.git"
# the bellow line, will be needed if cap deploy throws an error about git.sh access denied.
#set :tmp_dir, "/home/roruser"

set :deploy_to, "/home/roruser/#{fetch :application}"

# Default value for linked_dirs is []
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "vendor/bundle", ".bundle", "public/system", "public/uploads"

set :keep_releases, 5

````
Now we need to modify <code style="color: tomato;" > config/deploy/production.rb </code> to point to our server's IP address for production deployments. Make sure to replace <code style="color: tomato;" >  1.2.3.4 </code> with your server's public IP.

<strong> Change the 1.2.3.4 and user to match with your case</strong>

for the user we need to wrtite the name of the user that we created at step 1.

```ruby

server '1.2.3.4', user: 'user', roles: %w{app db web}

```
Commit changes.

```bash
user@local$ git init
user@local$ git add .
user@local$ git commit -m "capistrano"
user@local$ git push -u origin master

```

#### Git issues
I had to change the URL using
```bash
git remote set-url origin [The-https-from-your-repo]
```
origin = the name of the current brunch
<br><br>

After this all commands started working fine. You can check the change by using
```
git remote -v
```

<a name="conf_Nginx"></a>

## Step-08 Configure Nginx

Create a new nginx default file with these settings. Note that the socket is the same one specified in the Unicorn configuration.


```bash
user@local$ ssh roruser@1.2.3.4

roruser@remote$ sudo vim /etc/nginx/sites-available/default

```
> NOTE: if you have apache running then you must change the port from 80 to something else.

```bash
server {
    listen 80;
    listen[::]:80;

    server name {YOUR-DOMAIN} ;
    root /home/{YOUR-USER}/{YOUR-APP-NAME}/current/public;

    passenger_enabled on;
    passenger_app_env production;

    location /cable {
        passenger_app_group_name {YOUR-APP-NAME}_websocket;
        passenger-force-max-concurrent-request-per-proccess 0;
    }

    client_max_body_size 100m;

    location ~ ^/(assets|packs) {
        expires max;
        gzip_static on;
    }
}
```
Restart nginx.

```bash
roruser@remote$ sudo service nginx restart
```
<a name="Deploy"></a>

## Step-09 Deploy the app

Run <code style="color: tomato;" >cap production deploy:check</code> to make sure all your files and directories are in place. Everything should be successful. Then run <code style="color: tomato;"> cap production deploy</code> . First time will take a while since it has to run bundle and install all dependencies.


```bash
user@local$ cap production deploy:check

user@local$ cap production deploy
```
