# Create and deploy a new release

You can quickly create the initial scaffold for a new release using the 3rd-party `bosh-gen` tool (ok, to be fair, Dr Nic wrote it).

Install the tool using RubyGems:

```
gem install bosh-gen
```

```
$ mkdir -p ~/.bosh_deployments/redis-on-demand
$ cd ~/.bosh_deployments/redis-on-demand
$ bosh-gen new redis-on-demand
$ cd redis-on-demand
```

This will generate the scaffold for the release in a folder `~/.bosh_deployments/redis-on-demand/redis-on-demand`. Later we will put the one or more deployment manifest files in the parent folder `~/.bosh_deployments/redis-on-demand`.

The generated project will also be converted into a Git repository. If you don't want this, then delete the `~/.bosh_deployments/redis-on-demand/redis-on-demand/.git` folder .

## Packages built from original source

```
$ bosh-gen package redis
      create  packages/redis/packaging
      create  packages/redis/pre_packaging
      create  packages/redis/spec
```

The initial `spec` manifest shows below that this package requires no files and has no dependencies on any other packages.

```yml
$ cat packages/redis/spec
---
name: redis
dependencies: []
files: []
```

If we were to attempt to create a release now, we'll see that BOSH requires that packages all require one or more source files.

```
$ bosh create release --force
...

Building packages
-----------------
Error 500: Package 'redis' doesn't include any files
```

Let's add some redis source code to the redis package! (link comes from [redis downloads page](http://redis.io/download "Download – Redis"))

```
$ bosh-gen source redis http://redis.googlecode.com/files/redis-2.4.13.tar.gz
Downloading http://redis.googlecode.com/files/redis-2.4.13.tar.gz...
...
       force  packages/redis/spec
      create  src/redis/redis-2.4.13.tar.gz
```

The redis package spec is now updated to reference the source tarball.

```yaml
$ cat packages/redis/spec
---
name: redis
dependencies: []
files:
- redis/redis-2.4.13.tar.gz
```

We now have a valid release (albeit, without any jobs that will require our redis package)

```
$ bosh create release --force
...
Packages
+-------+---------+-------------+------------------------------------------+
| Name  | Version | Notes       | Fingerprint                              |
+-------+---------+-------------+------------------------------------------+
| redis | 0.1-dev | new version | aa3d5904ca4fbc49a13b71a98e44d0660abf905f |
+-------+---------+-------------+------------------------------------------+
...
Release version: 1
Release manifest: .../dev_releases/redis-on-demand-1.yml
```

## Packaging a package source

Packages are compiled on demand during the deployment. This process is completely automated by BOSH. How it works and how compilation is triggered will be discussed later. In this section, we need to tell BOSH how to compile the redis source code into binary executables. These compiled executables will be automatically available during deployment on all VMs that need it. 

Each package has two executable hooks that are run during the package compilation process

* `pre_packaging` - run when the source of the package is assembled during the `bosh create release` (in developer environment)
* `packaging` - compilation process run during deployment (in a clean stemcell VM)

For the redis package, only the `packaging` script is necessary. Copy the following into `packages/redis/packaging`. If you are using a more modern redis download, then replace `2.4.13` below with your version number.

```bash
tar zxf redis/redis-2.4.13.tar.gz

if [[ $? != 0 ]] ; then
  echo "Failed extracting redis"
  exit 1
fi

(
  cd redis-2.4.13
  make
  make PREFIX=$BOSH_INSTALL_TARGET install
)
```

Note `PREFIX=$BOSH_INSTALL_TARGET` above. All `packaging` scripts must place their compiled outputs in a specific folder that BOSH determines. This folder location is provided by the `$BOSH_INSTALL_TARGET` environment variable. Only the contents of this folder are used as the "compiled package", and only the contents of this folder will be installed into the VMs during deployment.

This is different from many other packaging systems which assume that they can install files anywhere on the file system. In BOSH, packages can only manage files within a specific subfolder. This ensures clean separation of all packages and all versions of the same package.

At this time in the tutorial, it is not yet possible to test our packaging scripts in isolation. Package compilation is performed automatically during deployment. Deployment of a package requires a deployment manifest and one or more jobs that use the package. 

Let's wire up the redis job and a simple deployment manifest now.

## Starting processes or jobs

We can now install redis, but we cannot yet run redis (specifically we care about `redis-server`). How to start or stop a process is described by a job. In fact, packages are only installed on VMs during deployment if they are required directly or indirectly by the job assigned to that VM.

To create a redis job that requires the redis package:

```
$ bosh-gen job redis -d redis
      create  jobs/redis
      create  jobs/redis/TODO.md
      create  jobs/redis/monit
      create  jobs/redis/templates/redis_ctl
       chmod  jobs/redis/templates/redis_ctl
      create  jobs/redis/spec
Next step for redis job:

* Replace "`exec /var/vcap/packages/redis/bin/EXECUTABLE_SERVER`" in jobs/redis/templates/redis_ctl

```

Jobs each have a spec file of their dependencies and of templates/files that they want to be created during deployment.

```yml
$ cat jobs/redis/spec 
---
name: redis
packages:
- redis
templates:
  redis_ctl: bin/redis_ctl
```

We need to update the initial `redis_ctl` script to tell the job how to start the `redis-server` command.

Replace the start of the following line:

```bash
exec /var/vcap/packages/redis/bin/EXECUTABLE_SERVER 
```

with

```bash
exec /var/vcap/packages/redis/bin/redis-server
```

NOTE: keep the remainder of the line that stores the `STDOUT` and `STDERR` pipes into log files.

It happens that `redis-server` can take configuration via its first argument. We will set up the configuration and pass properties from the deployment manifest later. For the moment, we will run redis using all its defaults.

The control script `redis_ctl` also determines where STDOUT and STDERR from the redis job will go. From the template, we will be able to find the log files at `/var/vcap/sys/log/redis` on any redis job VM.

We are not done yet. As it stands, when redis-server is run we are not storing the process ID (PID) anywhere. The PID is necessary so that the job process can be stopped and restarted.

How a process generates a PID and returns it to the control script can vary. For `redis-server` we can tell it where to store a PID via a configuration file.

First, let's finish the `redis_ctl` script and tell it to use a `redis.conf` file, which will in turn tell it where to store the PID value. Next, we will generate the `redis.conf` file from our job.

Change the lines we created above

```bash
exec /var/vcap/packages/redis/bin/redis-server ...
```

with

```bash
exec /var/vcap/packages/redis/bin/redis-server /var/vcap/jobs/redis/config/redis.yml ...
```

The control script now expects that there will be a `redis.yml` file. Our redis job is responsible for creating it.

In the release, create a new template file:

```
$ bosh-gen template redis config/redis.conf
      create  jobs/redis/templates/redis.conf.erb
       force  jobs/redis/spec
```

The redis job spec is updated to put the generated `redis.conf` into the `config/` folder. When BOSH deploys the job, this will be at `/var/vcap/job/redis/config/redis.conf`; just as we provided to the `redis-server` in the `redis_ctl` above.

Initially, `redis.conf.erb` is blank. Populate it with the following redis configuration:

```
daemonize no
pidfile /var/vcap/sys/run/redis/redis.pid
timeout 300
loglevel notice
logfile stdout
databases 16
dir /var/vcap/store/redis
maxclients 0
maxmemory-policy noeviction
appendonly no
```

For redis, we can tell it where to store its PID via the `pidfile` config value. The value provided `/var/vcap/sys/run/redis/redis.pid` matches the value of `$PIDFILE` in `jobs/redis/templates/redis_ctl`.

## Upload our release to BOSH

Before deploying a new or updated release, we must upload the latest version to BOSH.

First, commit our release repository changes and create a new release:

```
$ git add .
$ git commit -m "added redis package & job"
$ bosh create release
...
Release version: 2
...
```

Now upload the latest release to your BOSH:

```
$ bosh upload release
...
Release info
------------
Name:    redis-on-demand
Version: 2
...
Creating new packages
  redis/0.2-dev (00:00:00)                                                                          
Done                    1/1 00:00:00                                                                

Creating new jobs
  redis/0.1-dev (00:00:00)                                                                          
Done                    1/1 00:00:00
...
```

There are three concepts in our redis release and hence three release numbers being show above.

* release "redis-on-demand" is at version 2
* package "redis" is at version 0.2
* job "redis" is at version 0.1

If we change the contents of a job or a package then those jobs/packages increase their version number, and the release number increases.

You may notice the "dev" suffix. We will discuss "dev" and "final" releases later.

## Deploying our release





## Configuring Redis

There are two redis configuration options we will add that will use properties from the deployment manifest.

```
port <%= properties.redis.port %>
requirepass <%= properties.redis.password %>
```