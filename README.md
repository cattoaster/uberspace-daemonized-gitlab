uberspace-daemonized-gitlab
===========================

Short instruction for (post) GitLab setup (version 6). It helps you to initialize a daemonized GitLab on Uberspace, a popular shared hosting provider. 

All the magic is built into a couple nice shell scripts. They will create an environment for your GitLab setup and enable monitoring using Daemon tools.

## Preconditions

You have successfully installed GitLab on your uberspace. If not, follow this guide for GitLab version 6.7.0 http://kobralab.centaurus.uberspace.de/benni/uberspace/blob/master/install.md 

* Redis is running (https://uberspace.de/dokuwiki/database:redis)
* Ruby is configured for your uberspace, and your ```PATH``` environment variable contains the path to the Ruby bin directory

example:
```bash
export PATH=/package/host/localhost/ruby-2.1.1/bin:$PATH
export PATH=$HOME/.gem/ruby/2.1.0/bin:$PATH
```
* make sure you have Bundler installed

   Check https://uberspace.de/dokuwiki/development:ruby for more details.

## Set up daemons

You will be able to control your GitLab via this deamons. One for the **unicorn** process and one for the **sidekiq** one. Your **Redis** process should already be daemonized. You can use the official uberspace daemontools guide as a reference https://uberspace.de/dokuwiki/system:daemontools#einen_daemon_einrichten if you want to know more about "setting up daemontools at uberspace".

* Verify your personal service directory at uberspace.

```$ test -d ~/service || uberspace-setup-svscan```

  Setting up a service for **unicorn**:

```bash
# create run script for unicorn
# Insert correct path to your gitlab directory
# make sure your ruby gem path is correct or adjust it
# furthermore check your PATH and environment settings (if using zsh, ksh ...)

$ mkdir ~/etc/run-unicorn
$ cat <<__EOF__ > ~/etc/run-unicorn/run
#!/bin/sh

# These environment variables are sometimes needed by the running daemons
export USER=$USER
export HOME=/home/$USER

# Include the user-specific profile
. $HOME/.bash_profile

# Now let's go!
export RAILS_ENV="production"
export GITLAB_PATH="$HOME/__PATH__TO__YOUR__GITLAB__DIRECTORY__/gitlabhq"
export PATH=$HOME/.gem/ruby/2.1.0/bin:$PATH

cd $GITLAB_PATH/
app_root=$(pwd)
unicorn_config="$app_root/config/unicorn.rb"

exec bundle exec unicorn_rails -c $unicorn_config -E $RAILS_ENV 2>&1
__EOF__
$ chmod +x ~/etc/run-unicorn/run
$ mkdir ~/etc/run-unicorn/log
$ cat <<__EOF__ > ~/etc/run-unicorn/log/run
#!/bin/sh
exec multilog t ./main
__EOF__
$ chmod +x ~/etc/run-unicorn/log/run
$ ln -s ~/etc/run-unicorn ~/service/unicorn
```

   Alternatively, you can run

```$ uberspace-setup-service unicorn ~/bin/unicorn```
   
   .. and edit the run script.

   Daemon service for **sidekiq**:
  
```bash
# create run script for sidekiq
# Insert correct path to your gitlab directory
# make sure your ruby gem path is correct or adjust it
# furthermore check your PATH and environment settings (if using zsh, ksh ...)

$ mkdir ~/etc/run-sidekiq
$ cat <<__EOF__ > ~/etc/run-sidekiq/run
#!/bin/sh

# These environment variables are sometimes needed by the running daemons
export USER=$USER
export HOME=/home/$USER

# Include the user-specific profile
. $HOME/.bash_profile

export RAILS_ENV="production"
export GITLAB_PATH="$HOME/__PATH__TO__YOUR__GITLAB__DIRECTORY__/gitlabhq"
export PATH=$HOME/.gem/ruby/2.1.0/bin:$PATH

# Now let's go!
cd $GITLAB_PATH/
app_root=$(pwd)
sidekiq_pidfile="$app_root/tmp/pids/sidekiq.pid"

exec bundle exec sidekiq -q post_receive,mailer,system_hook,project_web_hook,gitlab_shell,common,default -e $RAILS_ENV -P $sidekiq_pidfile 2>&1
__EOF__
$ chmod +x ~/etc/run-sidekiq/run
$ mkdir ~/etc/run-sidekiq/log
$ cat <<__EOF__ > ~/etc/run-sidekiq/log/run
#!/bin/sh
exec multilog t ./main
__EOF__
$ chmod +x ~/etc/run-sidekiq/log/run
$ ln -s ~/etc/run-sidekiq ~/service/sidekiq
```
    
   Alternatively, you can run

```$ uberspace-setup-service sidekiq ~/bin/sidekiq```

   .. and edit the run script.

## Troubleshooting

There is one (to be identified) (very rare) case where the execution command fails:
````exec: bundle: not found````

Check your PATH and (if needed) add a path to your bundle path:

* Our run scripts execute a bundle command. Therefore, it's necessary to declare the path to the bundle binary. Run ```which bundle``` in order to retrieve this path and add it to your ```PATH``` environment variable.

  example:
```bash
export BUNDLE_BIN_PATH=$HOME/.gem/ruby/2.1.0/bin/bundle:$PATH
```

## Control your GitLab  (**unicorn** and **sidekiq** daemons)

To bring GitLab up:

```$ svc -u ~/service/unicorn```    
```$ svc -u ~/service/sidekiq```

To bring it down:

```$ svc -d ~/service/unicorn```

```$ svc -d ~/service/sidekiq```

To restart it: 

```$ svc -du ~/service/unicorn```

```$ svc -du ~/service/sidekiq```

More infos can be found here https://uberspace.de/dokuwiki/system:daemontools#wenn_der_daemon_laeuft

That's it folks. Have fun.
