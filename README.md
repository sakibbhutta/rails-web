# Task 1

1.  Create a directory `/rails` in LINUX environement
2.  Create a `Dockerfile` in newly created directory with following content:

```dockerfile
# syntax=docker/dockerfile:1
FROM ruby:2.5
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Configure the main process to run when running the image
CMD ["rails", "server", "-b", "0.0.0.0"]
```
Above docker file pulls the `ruby2.5` from Docker HUB. a newly working directory `/myapp` is created. Contenets of file Gemfile is copied to the directory 
`/myapp/Gemfile`. entrypoint.sh is copied to `/usr/bin/`, while port 3000 is exposed.

3.  Next, Create a file `Gemfile` which loads Rails. this is the dependency file

```ruby
source 'https://rubygems.org'
gem 'rails', '~>5'
```

4. Create an empty `Gemfile.lock` file to build `Dockerfile`.

```console
$ touch Gemfile.lock
```

5. Next, provide an entrypoint script to fix a Rails-specific issue that
prevents the server from restarting when a certain `server.pid` file pre-exists.
This script will be executed every time the container gets started.
`entrypoint.sh` consists of:

```bash
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```

6.  Create `docker-compose.yml` file is same directory `/rails`. This file has two containers, `db` and `web`. db
container pulls the image of postgres from dockerhub, also staorage is mapped to directory `/var/lib/postgresql/data`.
`POSTGRES_PASSWORD` is defined as environment variable. 
The web container is built in its own storage `/myapp` with 3000 port exposed. 
`web` container RUNS only if `db` container has already initiated, otherwise `web` will not run


```yaml
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
```



7.  Now, when all the files are placed in the folder, Rails app can be initiated with following command:

```console
$ docker compose run --no-deps web rails new . --force --database=postgresql
```

First, Compose builds the image for the `web` service using the `Dockerfile`.
The `--no-deps` tells Compose not to start linked services. Then it runs
`rails new` inside a new container, using that image. Once it's done, you
should have generated a fresh app.

8. Now that you’ve got a new Gemfile, you need to build the image again. (This, and
changes to the `Gemfile` or the Dockerfile, should be the only times you’ll need
to rebuild.)

```console
$ docker compose build
```
9. By default, Rails
expects a database to be running on `localhost` - so you need to point it at the
`db` container instead. You also need to change the database and username to
align with the defaults set by the `postgres` image.

Replace the contents of `config/database.yml` with the following:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password: password
  pool: 5

development:
  <<: *default
  database: myapp_development


test:
  <<: *default
  database: myapp_test
```

9.  Finally its time to compose up the docker image by following command:

```console
$ docker compose up
```

10. Now, you need to create the database. In another terminal, run:

```console
$ docker compose run web rake db:create
```
The logs will be like:
```
Starting rails_db_1 ... done
Created database 'myapp_development'
Created database 'myapp_test'
```

10. Go to `http://localhost:3000` on a web browser to see the Rails Welcome.

# Task 2
1.  'docker ps' command 
```console
$ docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED       STATUS          PORTS                                       NAMES
b5075ed4c3f0   rails-web   "entrypoint.sh bash …"   4 hours ago   Up 17 seconds   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   rails-web-1
aa21236f5f70   postgres    "docker-entrypoint.s…"   4 hours ago   Up 20 seconds   5432/tcp                                    rails-db-1
```
2.  'docker stop' Command
```console
$ docker stop
"docker stop" requires at least 1 argument.
See 'docker stop --help'.

Usage:  docker stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers
```
```console
$ docker docker stop 129b1c542872 b5075ed4c3f0
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                     PORTS      NAMES
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Exited (1) 5 minutes ago              rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   4 hours ago   Exited (1) 6 seconds ago              rails-web-1
aa21236f5f70   postgres       "docker-entrypoint.s…"   4 hours ago   Up 4 minutes (Paused)      5432/tcp   rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 5 hours ago                rails_web_run_f46f6dcdf6cf
```
3.  'docker start' command 
```console
$ docker start 129b1c542872 b5075ed4c3f0
ONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                   PORTS                                       NAMES
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Up 7 seconds             3000/tcp                                    rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   4 hours ago   Up 5 seconds             0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   rails-web-1
aa21236f5f70   postgres       "docker-entrypoint.s…"   4 hours ago   Up 5 minutes (Paused)    5432/tcp                                    rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 5 hours ago                                               rails_web_run_f46f6dcdf6cf
```
4.  'docker pause' command 
```console
$ docker pause aa21236f5f70
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Exited (1) 2 minutes ago                                                 rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   4 hours ago   Up 2 minutes                 0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   rails-web-1
aa21236f5f70   postgres       "docker-entrypoint.s…"   4 hours ago   Up About a minute (Paused)   5432/tcp                                    rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 4 hours ago                                                   rails_web_run_f46f6dcdf6cf
```
5. 'docker unpause' command
```console
$ docker unpause aa21236f5f70
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                     PORTS                                       NAMES
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Exited (0) 2 seconds ago                                               rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   4 hours ago   Up About a minute          0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   rails-web-1
aa21236f5f70   postgres       "docker-entrypoint.s…"   4 hours ago   Up 7 minutes               5432/tcp                                    rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 5 hours ago                                                 rails_web_run_f46f6dcdf6cf
```
6.  'docker rename' command
```console
$ docker rename b5075ed4c3f0 MyNewContainer
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                     PORTS                                       NAMES
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Exited (0) 3 minutes ago                                               rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   4 hours ago   Up 4 minutes               0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   MyNewContainer
aa21236f5f70   postgres       "docker-entrypoint.s…"   4 hours ago   Up 10 minutes              5432/tcp                                    rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 5 hours ago                                                 rails_web_run_f46f6dcdf6cf
```
7.  'docker top' command
```console
$ docker top b5075ed4c3f0
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                12398               12378               1                   15:16               ?                   00:00:05            puma 3.12.6 (tcp://0.0.0.0:3000) [myapp]

8.  'docker stats' command 
```console
$ docker stats b5075ed4c3f0

CONTAINER ID   NAME             CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O        PIDS
b5075ed4c3f0   MyNewContainer   0.03%     58.73MiB / 1.895GiB   3.03%     3.73kB / 0B   3.93MB / 143kB   17
```
9.  'docker restart' command 
```console
$ docker restart b5075ed4c3f0
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS                      PORTS                                       NAMES
129b1c542872   rails-web      "entrypoint.sh rake …"   4 hours ago   Exited (0) 11 minutes ago                                               rails_web_run_78039bea7f1d
b5075ed4c3f0   rails-web      "entrypoint.sh bash …"   5 hours ago   Up 15 seconds               0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   MyNewContainer
aa21236f5f70   postgres       "docker-entrypoint.s…"   5 hours ago   Up 18 minutes               5432/tcp                                    rails-db-1
6e34857b6261   32d90c4848e6   "entrypoint.sh rails…"   5 hours ago   Exited (0) 5 hours ago                                                  rails_web_run_f46f6dcdf6cf
```
10. 'docker logs' command 
```console
$ docker logs c6dd952812a1 
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
11. 'docker rm' command 
```console
$ docker rm c6dd952812a1
c6dd952812a1
```
12. 'docker inspect' command
```console
$ docker inspect 52c0f29ab16c 
[
    {
        "Id": "52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510",
        "Created": "2023-03-24T10:39:32.453762014Z",
        "Path": "/hello",
        "Args": [],
        "State": {
            "Status": "exited",
            "Running": false,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 0,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2023-03-24T10:39:34.28040598Z",
            "FinishedAt": "2023-03-24T10:39:34.261911031Z"
        },
        "Image": "sha256:feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412",
        "ResolvConfPath": "/var/lib/docker/containers/52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510/hostname",
        "HostsPath": "/var/lib/docker/containers/52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510/hosts",
        "LogPath": "/var/lib/docker/containers/52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510/52c0f29ab16c8970e2b3feecc0cabd121df92728a0c47bf92c8aa59900552510-json.log",
        "Name": "/confident_cohen",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "docker-default",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                33,
                182
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "private",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": null,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/d67b31ba22cba626b32622884cb360616c97fefc222541870204f4ae84c40012-init/diff:/var/lib/docker/overlay2/88bff16619d11fa096170dc12288fa8ca5f6109728fc80b7c48d48c551f5c106/diff",
                "MergedDir": "/var/lib/docker/overlay2/d67b31ba22cba626b32622884cb360616c97fefc222541870204f4ae84c40012/merged",
                "UpperDir": "/var/lib/docker/overlay2/d67b31ba22cba626b32622884cb360616c97fefc222541870204f4ae84c40012/diff",
                "WorkDir": "/var/lib/docker/overlay2/d67b31ba22cba626b32622884cb360616c97fefc222541870204f4ae84c40012/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "52c0f29ab16c",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": true,
            "AttachStderr": true,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/hello"
            ],
            "Image": "hello-world",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "1b2577b9b9f5c153c1f04857f6a3a69b8d1154e35aba6fcd787e40822e5941a5",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {},
            "SandboxKey": "/var/run/docker/netns/1b2577b9b9f5",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "",
            "Gateway": "",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "",
            "IPPrefixLen": 0,
            "IPv6Gateway": "",
            "MacAddress": "",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "8dc365830b509c1260c034cfa3171872ca9f65a88f92cc3a41f41c887f25d397",
                    "EndpointID": "",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
```
