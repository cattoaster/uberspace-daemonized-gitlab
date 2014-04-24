uberspace-daemonized-gitlab
===========================

Short instruction for (post) GitLab setup (version 6). It helps you to initialize a daemonized GitLab on Uberspace, a popular shared hosting provider. 

All the magic is built into a couple nice bash scripts. They will create an environment for your GitLab setup and enable monitoring using Daemon tools.

## Preconditions

You have successfully installed GitLab on your uberspace. If not, follow this guide for GitLab version 6.7.0 http://kobralab.centaurus.uberspace.de/benni/uberspace/blob/master/install.md (**__[2014-04-24 -- GitLab is not running] currently not accessible__**, I'll try to remind and sum up my own GitLab installation on uberspace and make it accessible here).

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

You will be able to control your GitLab via this deamons. One for the **unicorn** process and one for the **sidekiq** one. Your **Redis** process should already be daemonized.

* Generate executable files under ~/bin for unicorn and sidekiq.

```bash
# create run script for unicorn
# Insert correct path to your gitlab directory

$ cat > ~/bin/unicorn <<__EOF__
#!/bin/sh
export RAILS_ENV="production"
export GITLAB_PATH="$HOME/__PATH__TO__YOUR__GITLAB__DIRECTORY/gitlabhq"
export PATH=$HOME/.gem/ruby/2.1.0/bin:$PATH

cd $GITLAB_PATH/
app_root=$(pwd)
unicorn_config="$app_root/config/unicorn.rb"

bundle exec unicorn_rails -c $unicorn_config -E $RAILS_ENV
__EOF__
```
```bash
# create run script for sidekiq
# Insert correct path to your gitlab directory

$ cat > ~/bin/unicorn <<__EOF__
#!/bin/sh
export RAILS_ENV="production"
export GITLAB_PATH="$HOME/__PATH__TO__YOUR__GITLAB__DIRECTORY/gitlabhq"
export PATH=$HOME/.gem/ruby/2.1.0/bin:$PATH

cd $GITLAB_PATH/
app_root=$(pwd)
sidekiq_pidfile="$app_root/tmp/pids/sidekiq.pid"

bundle exec sidekiq -q post_receive,mailer,system_hook,project_web_hook,gitlab_shell,common,default -e $RAILS_ENV -P $sidekiq_pidfile
__EOF__
```

```bash
# make files executable

$ chmod +x ~/bin/unicorn ~/bin/sidekiq
```

* Our run scripts execute a bundle command. Therefore, it's necessary to declare the path to bundle binary. Run ```which bundle``` in order to retrieve this path and add it to your ```PATH``` environment variable.

  example:
```bash
export BUNDLE_BIN_PATH=$HOME/.gem/ruby/2.1.0/bin/bundle:$PATH
```

* Verify your personal service directory at uberspace.

```$ test -d ~/service || uberspace-setup-svscan```

  .. and set up a service for **unicorn**:

```$ uberspace-setup-service unicorn ~/bin/unicorn```

  .. and a service for **sidekiq**:
    
```$ uberspace-setup-service sidekiq ~/bin/sidekiq```

## Control your GitLab  (**unicorn** and **sidekiq** daemons)

***
<dl>
  <dt>Sidekiq daemon is not fully controllable</dt>
  <dd>Your sidekiq service starts at system restart. But you cannot control your running sidekiq process. Restart or shutdown signals invoked by svc (service control) are ignored. You have to invoke those signals by yourself (SIGTERM)</dd>
  <dd>I will update the sidekiq script as soon as a fix is known.</dd>
</dl>
***

To bring GitLab up:

```$ svc -u ~/service/unicorn```    
```$ svc -u ~/service/sidekiq```

To bring it down:

```$ svc -d ~/service/unicorn```

~~```$ svc -d ~/service/sidekiq```~~

To restart it: 

```$ svc -du ~/service/unicorn```

~~```$ svc -du ~/service/sidekiq```~~

More infos can be found here https://uberspace.de/dokuwiki/system:daemontools#wenn_der_daemon_laeuft

That's it folks. Have fun.
