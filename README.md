# Docker role `mhutter.docker-systemd-service`
[![Build Status](https://travis-ci.com/mhutter/ansible-docker-systemd-service.svg?branch=master)](https://travis-ci.com/mhutter/ansible-docker-systemd-service)

Generic role for creating systemd services to manage docker containers.

## Example

```yaml
- name: Start WebApp
  include_role:
    name: mhutter.docker-systemd-service
  vars:
    container_name: myapp
    container_image: myapp:latest
    container_links: [ 'mysql' ]
    container_volumes:
      - '/data/uploads:/data/uploads'
    container_ports:
      - '3000:3000'
    container_env:
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_pw }}"
    container_labels:
      - 'traefik.enable=true'
```

This will create:

* A file containing the env vars (either `/etc/sysconfig/mysql` or `/etc/default/mysql`).
* A systemd unit which starts and stops the container. The unit will be called
  `<name>_container.service` to avoid name clashes.

### Role variables

* `container_name` (**required**) - name of the container

#### Docker container specifics

* `container_image` (**required**) - Docker image the service uses
* `container_args` - arbitrary list of arguments to the `docker run` command
* `container_cmd` - optional command to the container run command (the part after the
  image name)
* `container_env` - key/value pairs of ENV vars that need to be present
* `container_volumes` (default: _[]_) - List of `-v` arguments
* `container_ports` (default: _[]_) - List of `-p` arguments
* `container_link` (default: _[]_) - List of `--link` arguments
* `container_labels` (default: _[]_) - List of `-l` arguments
* `container_docker_pull` (default: _yes_) - whether the docker image should be pulled

#### Systemd service specifics

* `enabled` (default: _yes_) - whether the service should be enabled
* `masked` (default: _no_) - whether the service should be masked
* `state` (default: _started_) - state the service should be in - set to
  `absent` to remove the service.

## Installation

This role requires the [docker python module](https://pypi.org/project/docker/).
Install it with `pip3 install docker` or `apt install python3-docker`
(or drop the `3` for python 2.x).

Put this in your `requirements.yml`:

```yml
- role: mhutter.docker-systemd-service
```

and run `ansible-galaxy install -r requirements.yml`.


## Gotchas

* When the unit or env file is changed, systemd gets reloaded but existing
  containers are NOT restarted.
* Make sure to quote values for `container_ports`, `container_volumes` and so
  on, especially if they contain colons (`:`). Otherwise YAML will interpret
  them as hashes/maps and ansible will throw up.

## About orchestrating Docker containers using systemd.

The concept behind this is to define `systemd` units for every docker container.
This has some benefits:
- `systemd` is a well-known interface
- all services are controllable via the same tool (`systemctl`)
- all logs are accessible via the same tool (`journalctl`)
- dependencies can be defined
- startup behaviour can be defined
- by correctly defining the unit (see below), we can ensure we always have a clean container.

Here is an example `myapp_container.service` unit file (about what's produced
by above code):

    [Unit]
    # define dependencies
    After=docker.service
    PartOf=docker.service
    Requires=docker.service

    [Service]
    # Load ENV vars from a file. Note that this env vars will only be
    # accessible in the context of the Exec* commands, and not within the
    # container itself. To make env-vars accessible within the Container, we use
    # the `--env-file` flag for the `docker run` command.
    EnvironmentFile=/etc/sysconfig/myapp

    # Even though we explicitly run the container using the `--rm` flag, there
    # may be leftover containers (eg. after a system-, docker- or app-crash).
    # Starting a container with an existing name will always fail.

    ExecStartPre=-/usr/bin/docker rm -f myapp

    # actually run the container.
    # `--name` to identify the container
    # `--rm` ensure the container is removed after stopping
    # `--env-file` make ENV vars accessible to app
    # `--link mysql` link to a container named `mysql`. The DB will then be
    #                accesible at `mysql:3306`
    # `-v` mount `/data/uploads` into the container
    # `-p 3000:3000` expose port 3000 on the network
    ExecStart=/usr/bin/docker run --name myapp --rm --env-file /etc/sysconfig/myapp --link mysql -v /data/uploads:/data/uploads -p 3000:3000 registry.cust.net/myapp/myapp:latest
    # note that there is no `--restart` parameter. This is because restarting
    # is taken care of by `systemd`.

    # Stop command.
    ExecStop=/usr/bin/docker stop myapp

    # Ensure log messages are correctly tagged in the system log.
    SyslogIdentifier=myapp

    # Auto-Restart the container after a crash.
    Restart=always


    [Install]
    # make sure service is started after docker is up
    WantedBy=docker.service
