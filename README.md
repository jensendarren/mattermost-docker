# Production Docker deployment for Mattermost

This project enables deployment of a Mattermost server in a multi-node production configuration using Docker.

[![Build Status](https://travis-ci.org/mattermost/mattermost-docker.svg?branch=master)](https://travis-ci.org/mattermost/mattermost-docker)

## Installing on AWS using Docker Compose on EC2 and RDS Aurora Serverless

I've forked this repo to make a few of modificatons (including these notes) for deployment to AWS.

* Start up a new RDS Aurora Serverless Postgres database.
* In [docker-compose.yml](./docker-compose.yml) update the `MM_USERNAME`, `MM_PASSWORD`, `MM_DBNAME` and `MM_SQLSETTINGS_DATASOURCE` url to match that of the new database setup in RDS.
* Setup a new EC2 instance using Ubuntu and ssh into that instance as usual
* Install Docker and Docker Compose on the new EC2 instance by following [these instructions](https://docs.mattermost.com/install/prod-docker.html). You will need to replace `<username>` in these instructions with `ubuntu` (or the username of the linux instance you are using). Also in step 3 when cloning the mattermost-docker repo you should clone this forked repo!
* Check the status of the running containers `docker ps` and make sure they are healthy. If they are not something may need to be fixed so check the Docker Logs to see what is wrong. It could be the connection to the database most likely so make sure the correct variables are set in the docker-compose.yml file.

Notes:
- The default Mattermost edition for this repo has changed from Team Edition to Enterprise Edition. Please see [Choose Edition](#choose-edition-to-install) section.
- To install this Docker project on AWS Elastic Beanstalk please see [AWS Elastic Beanstalk Guide](contrib/aws/README.md).
- To run Mattermost on Kubernetes you can start with the [manifest examples in the kubernetes folder](contrib/kubernetes/README.md)
- To install Mattermost without Docker directly onto a Linux-based operating systems, please see [Admin Guide](https://docs.mattermost.com/guides/administrator.html#installing-mattermost).

## Installation using Docker Compose

The following instructions deploy Mattermost in a production configuration using multi-node Docker Compose set up.

### Requirements

* [docker] (version `1.12+`)
* [docker-compose] (version `1.10.0+` to support Compose file version `3.0`)

### Choose Edition to Install

If you want to install Enterprise Edition, you can skip this section.

To install the team edition, uncomment out these lines in docker-compose.yaml file:
```yaml
args:
  - edition=team
```
The `app` Dockerfile will read the `edition` build argument to install Team (`edition = 'team'`) or Enterprise (`edition != team`) edition.

### Application container
Application container run the Mattermost application. You should configure it with following environment variables :
* `MM_USERNAME`: database username
* `MM_PASSWORD`: database password
* `MM_DBNAME`: database name

If your database use some custom host and port, it is also possible to configure them :
* `DB_HOST`: database host address
* `DB_PORT_NUMBER`: database port

If you use a Mattermost configuration file on a different location than the default one (`/mattermost/config/config.json`) :
* `MM_CONFIG`: configuration file location inside the container.

If you choose to use MySQL instead of PostgreSQL, you should set a different datasource and SQL driver :
* `DB_PORT_NUMBER` : `3306`
* `MM_SQLSETTINGS_DRIVERNAME` : `mysql`
* `MM_SQLSETTINGS_DATASOURCE` : `MM_USERNAME:MM_PASSWORD@tcp(DB_HOST:DB_PORT_NUMBER)/MM_DBNAME?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s`
Don't forget to replace all entries (beginning by `MM_` and `DB_`) in `MM_SQLSETTINGS_DATASOURCE` with the real variables values.

If you want to push Mattermost application to **Cloud Foundry**, use a `manifest.yml` like this one (with external PostgreSQL service):

```
---
applications:
- name: mattermost
  docker:
    image: mattermost/mattermost-prod-app
  instances: 1
  memory: 1G
  disk_quota: 256M
  env:
    DB_HOST: database host address
    DB_PORT_NUMBER: database port
    MM_DBNAME: database name
    MM_USERNAME: database username
    MM_PASSWORD: database password

```

### Web server container
This image is optional, you should **not** use it when you have your own reverse-proxy. It is a simple front Web server for the Mattermost app container. If you use the provided `docker-compose.yml` file, you don't have to configure anything. But if your application container is reachable on custom host and/or port (eg. if you use a container provider), you should add those two environment variables :
* `APP_HOST`: application host address
* `APP_PORT_NUMBER`: application HTTP port

If you plan to upload large files to your Mattermost instance, Nginx will need to write some temporary files. In that case, the `read_only: true` option on the `web` container should be removed from your `docker-compose.yml` file.

#### Install with SSL certificate
Put your SSL certificate as `./volumes/web/cert/cert.pem` and the private key that has
no password as `./volumes/web/cert/key-no-password.pem`. If you don't have
them you may generate a self-signed SSL certificate.

### Starting/Stopping Docker

#### Start
If you are running docker with non root user, make sure the UID and GID in app/Dockerfile are the same as your current UID/GID
```
mkdir -p ./volumes/app/mattermost/{data,logs,config,plugins}
chown -R 2000:2000 ./volumes/app/mattermost/
docker-compose start
```

#### Stop
```
docker-compose stop
```

### Removing Docker

#### Remove the containers
```
docker-compose stop && docker-compose rm
```

#### Remove the data and settings of your Mattermost instance
```
sudo rm -rf volumes
```

## Update Mattermost to latest version

First, shutdown your containers to back up your data.

```
docker-compose down
```

Back up your mounted volumes to save your data. If you use the default `docker-compose.yml` file proposed on this repository, your data is on `./volumes/` folder.

Then run the following commands.

```
git pull
docker-compose build
docker-compose up -d
```

Your Docker image should now be on the latest Mattermost version.


## Upgrading Mattermost to 4.9+

Docker images for `4.9.0` release introduce some important changes from [PR #241](https://github.com/mattermost/mattermost-docker/pull/241) to improve production use of Mattermost with Docker.
**There are 2 important changes for existing installations**

One important change is that we don't use `root` user by default to run the Mattermost application. So, as explained on [the README](https://github.com/mattermost/mattermost-docker#start), if you use host mounted volume you have to be sure that files on your host server have the correct UID/GID (by default those values are `2000`). In practice, you should just run following commands :
```
mkdir -p ./volumes/app/mattermost/{data,logs,config,plugins}
chown -R 2000:2000 ./volumes/app/mattermost/
```

The second important change is the port used by Mattermost application container. The default port is now `8000`, and existing installations that use port `80` will not work without a little configuration change. You have to open your Mattermost configuration file (`./volumes/app/mattermost/config/config.json` by default) and change the key `ServiceSettings.ListenAddress` to `:8000`.
Also if you use your own web-server/reverse-proxy you need to change its configuration to reach port `8000` of the Mattermost container.


## Upgrading to Team Edition 3.0.x from 2.x

You need to migrate your database before upgrading Mattermost to `3.0.x` from
`2.x`. Run these commands in the latest `mattermost-docker` directory.
```
docker-compose rm -f app
docker-compose build app
docker-compose run app -upgrade_db_30
docker-compose up -d
```
See the [offical Upgrade Guide](http://docs.mattermost.com/administration/upgrade.html) for more details.

## Installation using Docker Swarm Mode

The following instructions deploy Mattermost in a production configuration using docker swarm mode on one node.
Running containerized applications on multi-node swarms involves specific data portability and replication handling that are not covered here.

### Requirements

* [docker] (1.12.0+)

### Swarm Mode Installation

First, create mattermost directory structure on the docker hosts:
```
mkdir -p /var/lib/mattermost/{cert,config,data,logs,plugins}
```

Then, fire up the stack in your swarm:
```
docker stack deploy -c contrib/swarm/docker-stack.yml mattermost
```

## Known Issues

* Do not modify the Listen Address in Service Settings.
* Rarely `app` container fails to start because of "connection refused" to
  database. Workaround: Restart the container.

## More information

If you want to know how to use docker-compose, see [the overview
page](https://docs.docker.com/compose).

For the server configurations, see [prod-ubuntu.rst] of Mattermost.

[docker]: http://docs.docker.com/engine/installation/
[docker-compose]: https://docs.docker.com/compose/install/
[prod-ubuntu.rst]: https://docs.mattermost.com/install/install-ubuntu-1604.html
