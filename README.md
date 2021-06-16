> :warning: This is an example to illustrate the configuration and communication of the various components
> in a complete FACT-Finder cluster setup. It is not intended for use in a productive environment in which
> we recommend using Docker Swarm or Kubernetes and distributing it to multiple servers.

## Setup example (Docker Compose)

### Prerequisites

Before you can follow this article, you need to have a system with an installed Docker and Docker Compose
environment. On top of that, you should have an account for the FACT-Finder Docker registry which you can
request from your contact person.

### Configuration

Once you satisfied the need for Docker, you should download and provide a preset of configuration files from
our GitHub project (https://github.com/FACT-Finder/ff-cluster-compose) in a directory of your choice. The directory structure should look like this:

```
ff-compose
├── analytics
│   └── production.conf
├── docker-compose.yml
├── .env
├── postgres
│   └── init.sql
├── rabbitmq
│   ├── rabbitmq.conf
│   └── rabbitmq.definitions
└── resources
```

The `resources` directory is just an empty directory which is not included in the package but has to be created
before continuing. Some of these files need a few adaptions from your side in order to work properly.

#### production.conf

The production.conf includes credentials for analytics and RabbitMQ which should be subject to change:

```
...

# Change the host when running analytics behind a reverse proxy. In this example, traefik serves analytics on localhost/analytics
swagger.api.host = "localhost/analytics"

play.http.secret.key = "mysecretkey"

...

op-rabbit {
  
  ...

  connection {

    ...
    
    username = "rabbitmquser"
    password = "rabbitmqsecurepassword"

    ...
  }
}
```

#### rabbitmq.conf

The rabbitmq.conf contains a user setting which has to be adapted to the configured user from production.conf:

```
loopback_users.rabbitmquser = false

...
```

#### rabbitmq.definitions

The rabbitmq.definitions has some settings which have to be adapted to the credentials from production.conf:

```
{
...

  "users": [
    {
      # User from production.conf
      "name": "rabbitmquser",
      # Password hash based on the configured algorithm and the password from production.conf.
      # See RabbitMQ documentation.
      "password_hash": "o55oKk9KodCLHvnC1+hmYQIQuJhhZdbYjJpUehUbHbzb4S5Y",
      "hashing_algorithm": "rabbit_password_hashing_sha256",
      "tags": "administrator"
    }
  ],

  ...

  "permissions": [
    {
      # User from production.conf
      "user": "rabbitmquser",

      ...
    }
  ],

...
}
```

#### .env

The .env file defines the docker registry and versions of images that are used inside the docker-compose configuration:

```
REGISTRY=<Docker image registry>
FF_VERSION=<FACT-Finder version of your choice>
ANALYTICS_TAG=<Analytics version of your choice>
```

### Docker Compose

Docker Compose is a tool for defining and running multi-container Docker applications. It uses a YAML file to configure
the application's services, and it needs only one single command to create and start all the services from the configuration.

The example Compose file creates a FACT-Finder environment that is made up of the following components:

![FACT-Finder Environment](images/environment.png)

The diagram shows the references of the different components in the cluster environment to each other. Please do not understand
it as a detailed description of the communication channels or the data flow in the system, e.g. the load balancer sends requests
to the API proxies and receives responses even if this is not shown here. Please have a look at the following paragraphs for
additional information about the components above.

#### Load balancer (Traefik)

In general, the load balancer's job is to receive incoming requests, distribute them to the different components and return
the response. Most of these requests are search requests for the workers, but it also serves the FACT-Finder- and Analytics-UI.
Since the requests may rely on old API versions, the load balancer can make use of the API proxy.

#### API proxy

The request and response format of the FACT-Finder REST-API slightly differs from version to version due to the needed changes
for additional features. This may be a problem since the shop using the FACT-Finder relies on a specific format. In order to
enable the shop to use a new version of the FACT-Finder without the need to change the integration, the API proxy can be used
in a FACT-Finder environment to translate requests and responses between different API versions. This is achieved via a load
balancer that sends every request with an older API version than the FACT-Finder's API version to the API proxy, receives the
translated request and sends it to the FACT-Finder afterwards. It works the same way with the response in the opposite direction.

##### Request flow

In this example setup, search requests will be routed as in the following diagram, depending on whether they belong to API version
`v2`, `v3` or `v4`.  Note that `v2` and `v3` requests pass the load balancer twice.

![API-Proxy Diagram](images/api-proxy.png)

#### Director, UI and Workers

The FACT-Finder provides two different kinds of instances with a different set of jobs:

##### Director + UI

The director is a management instance in the example setup. It comes with a connection to the FACT-Finder UI in order to
configure the system, run imports and do research relating to the search quality. It is also possible to run delta updates for
the product data. However, the director does not respond to any search requests.

##### Workers

In comparison to the director, the workers have exactly one single job to fulfill - they respond to search related requests.
The workers are running with the same configuration as the director, and the setup can be scaled up to provide as many workers
as needed to respond to the incoming amount of search requests in a reasonable amount of time.

#### Analytics + UI

The Analytics software uses the data concerning the user interaction with the FACT-Finder to give insights into the user behaviour,
the success of the shop, suspected problems with the integration and potential for improvement.

#### RabbitMQ

The RabbitMQ is a message broker used by the FACT-Finder to realize live learning for features like A/B testing or personalization.
Please refer to the RabbitMQ documentation for further information.

#### PostgreSQL

The PostgreSQL database in the example setup stores the product data as well as the data coming from Analytics. Please refer to the
PostgreSQL documentation for further information.

#### Docker Compose file

Below is a commented example Compose file to run a FACT-Finder environment like described above in this documentation. Please note
that an account for the FACT-Finder registry is needed to receive the docker images. Feel free to adapt the example to fit your needs.

```yaml
version: '3.7'
services:
    # Load balancer in the cluster
    traefik:
        image: traefik:v2.1.2
        command:
            - --api.insecure=true
            - --providers.docker
            - --log.level=DEBUG
            - --entrypoints.web.address=:80
            # Internal entrypoint for communication between services, e.g.
            # api-proxy -> worker -> director.
            - --entrypoints.internal.address=:8090
            - --providers.docker.exposedbydefault=false
        labels:
            - "traefik.enable=true"
        ports:
            - "127.0.0.1:80:80"
            - "127.0.0.1:8080:8080"
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock

    # UI pointing towards the director
    director-ui4:
        image: ${REGISTRY}/ui4/ng:${FF_VERSION}
        environment:
            - FACT_FINDER_URL=http://director:8080/fact-finder
            - ANALYTICS_URL=http://analytics:9000
            - FACT_FINDER_GWT_UI_URL=http://director-gwt-ui:8080/fact-finder-ui
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.ui.loadbalancer.server.port=80"
            - "traefik.http.routers.ui.rule=PathPrefix(`/fact-finder-ui`)"
            - "traefik.http.routers.ui.entrypoints=web"
            - "traefik.http.routers.ui.middlewares=\
                strip-ffui-prefix@docker"
            - "traefik.http.middlewares.strip-ffui-prefix\
                .stripprefix.prefixes=/fact-finder-ui"


    # Classic GWT UI included in ui4
    director-gwt-ui:
        image: ${REGISTRY}/ui/ng:${FF_VERSION}
        user: factfinder
        environment:
            - "JAVA_OPTS=-Dserver.url=\
              http://traefik:8090/director/fact-finder/ui/ws/soap/ -Xmx3g"

    # Director which is used to change configuration via UI, receive delta-updates, etc.
    director:
        image: ${REGISTRY}/ff/ng:${FF_VERSION}
        user: factfinder
        environment:
            # Adjust Xmx for your setup.
            - JAVA_OPTS=-Xmx3g -Dfff.node.logs.subdirectory=director
            # Internal resources directory
            - FACTFINDER_RESOURCES=/home/factfinder
            # URL pointing towards analytics internal/external
            - analytics.public.url=http://localhost/analytics
            - analytics.url=http://analytics:9000
            # Use postgres database for product data
            - importer.dialect=POSTGRES
            - importer.serverName=postgres
            # Configure client credentials for the workers. Workers must be
            # able to create oauth tokens to communicate with analytics.
            - oauth.authorization.client.ff_worker.secret=shared-secret
            # Director should use the director role
            - cluster.role=director
            # Import specific channels on startup which are configured accordingly
            - useImportOnStartup=false
            # Message queue for features like A/B testing. Change login data according to your settings.
            # For further information please refer to the RabbitMQ documentation
            - rabbitmq.uri=amqp://rabbitmquser:rabbitmqsecurepassword@rabbitmq:5672
            - trustHttpForwardedHeaders=true
        # Tune tcp keepalive settings for ipvs loadbalancer used e.g. by docker swarm.
        # See https://web.archive.org/web/20200614124130if_/https://success.docker.com/article/ipvs-connection-timeout-issue
        # for more details.
        sysctls:
            - net.ipv4.tcp_keepalive_time=600
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.director.loadbalancer.server.port=8080"
            - "traefik.http.routers.director.rule=PathPrefix(`/director`)"
            - "traefik.http.routers.director.middlewares=\
                strip-director-prefix@docker"
            # Listen on external and internal entrypoints for api-proxy
            # routing.
            - "traefik.http.routers.director.entrypoints=web,internal"
            - "traefik.http.middlewares.strip-director-prefix\
                .stripprefix.prefixes=/director"
        volumes:
            # Mount resource directory for FACT-Finder configuration
            - ./resources:/home/factfinder/fact-finder

    # Worker 1 to respond to search requests. You can add more workers.
    worker-1:
        image: ${REGISTRY}/ff/ng:${FF_VERSION}
        user: factfinder
        environment:
            # Adjust Xmx for your setup.
            # Use a separate log directory per worker to avoid messing up your
            # app, search and shoppingcart logs.
            - JAVA_OPTS=-Xmx3g -Dfff.node.logs.subdirectory=worker-1
            # Internal resources directory
            - FACTFINDER_RESOURCES=/home/factfinder
            # URL pointing towards analytics
            - analytics.url=http://analytics:9000
            # Use postgres database for product data
            - importer.dialect=POSTGRES
            - importer.serverName=postgres
            # Worker should use the worker role
            # Restricts workers capabilities, e.g. a worker is not allowed to
            # do full imports.
            - cluster.role=worker
            # Configures a centralized oauth authorization server, such that
            # all tokens are valid on all workers. To make it work, all of the
            # three next variables must be set.
            - oauth.server.url=http://traefik:8090/director/fact-finder
            # The credentials must match with the director setting
            # `oauth.authorization.client`.
            - oauth.client.id=ff_worker
            - oauth.client.secret=shared-secret
            # Import specific channels on startup which are configured accordingly
            - useImportOnStartup=false
            # Automatically poll for new delta updates already applied on the
            # director.
            - useAutoDeltaSync=true
            # Disable scheduled jobs on workers. If you have to use scheduled
            # jobs on your worker, make sure to configure a separate scheduler
            # directory using
            # - scheduler.directory={APP_RESOURCES}/conf/scheduler.worker/
            - useScheduledJobs=false
            # Message queue for features like A/B testing. Change login data according to your settings.
            # For further information please refer to the RabbitMQ documentation
            - rabbitmq.uri=amqp://rabbitmquser:rabbitmqsecurepassword@rabbitmq:5672
        # Tune tcp keepalive settings for ipvs loadbalancer used e.g. by docker swarm.
        # See https://web.archive.org/web/20200614124130if_/https://success.docker.com/article/ipvs-connection-timeout-issue
        # for more details.
        sysctls:
            - net.ipv4.tcp_keepalive_time=600
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.worker.loadbalancer.server.port=8080"
            - "traefik.http.services.worker.loadBalancer.sticky.cookie.name=ff_worker"
            - "traefik.http.routers.worker.rule=PathPrefix(`/worker`)"
            - "traefik.http.routers.worker.middlewares=\
                strip-worker-prefix@docker"
            - "traefik.http.routers.worker.entrypoints=web,internal"
            - "traefik.http.middlewares.strip-worker-prefix\
                .stripprefix.prefixes=/worker"
        volumes:
            # Mount resource directory for FACT-Finder configuration
            - ./resources:/home/factfinder/fact-finder

    # Worker 2 to respond to search requests. You can add more workers.
    worker-2:
        image: ${REGISTRY}/ff/ng:${FF_VERSION}
        user: factfinder
        environment:
            # Adjust Xmx for your setup.
            # Use a separate log directory per worker to avoid messing up your
            # app, search and shoppingcart logs.
            - JAVA_OPTS=-Xmx3g -Dfff.node.logs.subdirectory=worker-2
            # Internal resources directory
            - FACTFINDER_RESOURCES=/home/factfinder
            # URL pointing towards analytics
            - analytics.url=http://analytics:9000
            # Use postgres database for product data
            - importer.dialect=POSTGRES
            - importer.serverName=postgres
            # Worker should use the worker role
            # Restricts workers capabilities, e.g. a worker is not allowed to
            # do full imports.
            - cluster.role=worker
            # Configures a centralized oauth authorization server, such that
            # all tokens are valid on all workers. To make it work, all of the
            # three next variables must be set.
            - oauth.server.url=http://traefik:8090/director/fact-finder
            # The credentials must match with the director setting
            # `oauth.authorization.client`.
            - oauth.client.id=ff_worker
            - oauth.client.secret=shared-secret
            # Import specific channels on startup which are configured accordingly
            - useImportOnStartup=false
            # Automatically poll for new delta updates already applied on the
            # director.
            - useAutoDeltaSync=true
            # Disable scheduled jobs on workers. If you have to use scheduled
            # jobs on your worker, make sure to configure a separate scheduler
            # directory using
            # - scheduler.directory={APP_RESOURCES}/conf/scheduler.worker/
            - useScheduledJobs=false
            # Message queue for features like A/B testing. Change login data according to your settings.
            # For further information please refer to the RabbitMQ documentation
            - rabbitmq.uri=amqp://rabbitmquser:rabbitmqsecurepassword@rabbitmq:5672
        # Tune tcp keepalive settings for ipvs loadbalancer used e.g. by docker swarm.
        # See https://web.archive.org/web/20200614124130if_/https://success.docker.com/article/ipvs-connection-timeout-issue
        # for more details.
        sysctls:
            - net.ipv4.tcp_keepalive_time=600
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.worker.loadbalancer.server.port=8080"
            - "traefik.http.services.worker.loadBalancer.sticky.cookie.name=ff_worker"
            - "traefik.http.routers.worker.rule=PathPrefix(`/worker`)"
            - "traefik.http.routers.worker.middlewares=\
                strip-worker-prefix@docker"
            - "traefik.http.routers.worker.entrypoints=web,internal"
            - "traefik.http.middlewares.strip-worker-prefix\
                .stripprefix.prefixes=/worker"
        volumes:
            # Mount resource directory for FACT-Finder configuration
            - ./resources:/home/factfinder/fact-finder

    # Translates incoming requests on /director/fact-finder/rest/v2 or
    # /director/fact-finder/rest/v3 to /director/fact-finder/rest/v4
    api-proxy-director:
        image: ${REGISTRY}/ff-api-proxy/ng:${FF_VERSION}
        # Set target-version to the current API version of FACT-Finder
        command: >
            --target http://traefik:8090/director/fact-finder
            --target-version 4
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.api-proxy-director\
                .loadbalancer.server.port=3000"
            - "traefik.http.routers.api-proxy-director.entrypoints=web"
            # Create rules for every API version in use below target-version beginning from v2
            - "traefik.http.routers.api-proxy-director.rule=\
                PathPrefix(`/director/fact-finder/rest/v2`) || PathPrefix(`/director/fact-finder/rest/v3`)"
            - "traefik.http.routers.api-proxy-director.middlewares=\
                strip-api-proxy-director-prefix@docker"
            - "traefik.http.middlewares.strip-api-proxy-director-prefix.\
                stripprefix.prefixes=/director/fact-finder"

    # Translates incoming requests on /worker/fact-finder/rest/v2 or
    # /worker/fact-finder/rest/v3 to /worker/fact-finder/rest/v4
    api-proxy-worker:
        image: ${REGISTRY}/ff-api-proxy/ng:${FF_VERSION}
        # Use the internal traefik entrypoint, such that the traefik
        # load balancer will be used. If you'd point directly to the swarm
        # service, the swarm load balancer would be used without support for
        # sticky session and weights.
        #
        # Set target-version to the current API version of FACT-Finder
        command: >
            --target http://traefik:8090/worker/fact-finder
            --target-version 4
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.api-proxy-worker\
                .loadbalancer.server.port=3000"
            - "traefik.http.routers.api-proxy-worker.entrypoints=web"
            # Create rules for every API version in use below target-version beginning from v2
            - "traefik.http.routers.api-proxy-worker.rule=\
                PathPrefix(`/worker/fact-finder/rest/v2`) || PathPrefix(`/worker/fact-finder/rest/v3`)"
            - "traefik.http.routers.api-proxy-worker.middlewares=\
                strip-api-proxy-worker-prefix@docker"
            - "traefik.http.middlewares.strip-api-proxy-worker-prefix\
                .stripprefix.prefixes=/worker/fact-finder"

    # Analytics software to analyse FACT-Finder logfiles
    analytics:
        image: ${REGISTRY}/newlytics:${ANALYTICS_TAG}
        command: ['-Dconfig.file=/opt/docker/production.conf']
        volumes:
            - ./resources:/home/factfinder/ff
            - ./analytics/resources:/home/factfinder/analytics
            - ./analytics/production.conf:/opt/docker/production.conf
        environment:
            - LOGS_DIRECTORY=/home/factfinder/ff/logs
            - PG_DATABASE_URL=jdbc:postgresql://postgres/analytics?user=postgres
            - OAUTH_URL=http://traefik:8090/director/fact-finder
            - FF_URL=http://traefik:8090/director/fact-finder
            # Set to the current API version of FACT-Finder
            - FF_API_VERSION=v4
        # Tune tcp keepalive settings for ipvs loadbalancer used e.g. by docker swarm.
        # See https://web.archive.org/web/20200614124130if_/https://success.docker.com/article/ipvs-connection-timeout-issue
        # for more details.
        sysctls:
            - net.ipv4.tcp_keepalive_time=600
        depends_on:
            - director
            - postgres
        labels:
            - "traefik.enable=true"
            - "traefik.http.services.analytics\
                .loadbalancer.server.port=9000"
            - "traefik.http.routers.analytics.entrypoints=web"
            - "traefik.http.routers.analytics.rule=\
                PathPrefix(`/analytics`)"
            - "traefik.http.routers.analytics.middlewares=\
                strip-analytics-prefix@docker"
            - "traefik.http.middlewares.strip-analytics-prefix\
                .stripprefix.prefixes=/analytics"
        logging:
            driver: 'json-file'

    # Database for product data and delta updates
    postgres:
        image: postgres:11.7-alpine
        shm_size: 256M
        volumes:
            - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
        environment:
            - POSTGRES_HOST_AUTH_METHOD=trust
        # Tune tcp keepalive settings for ipvs loadbalancer used e.g. by docker swarm.
        # See https://web.archive.org/web/20200614124130if_/https://success.docker.com/article/ipvs-connection-timeout-issue
        # for more details.
        sysctls:
            - net.ipv4.tcp_keepalive_time=600

    # Message queue
    rabbitmq:
        image: rabbitmq:3.7.8-management
        volumes:
            - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
            - ./rabbitmq/rabbitmq.definitions:/var/lib/rabbitmq/definitions.json
```

### Environment URLs

Without any adaptions to the predefined URLs in the example Compose file, the different components are available at the following URLs:

|**Component**|**URL**|
|---|---|
|UI|http://localhost/fact-finder-ui|
|Director|http://localhost/director/fact-finder|
|Worker 1 + 2 (via load balancer)|http://localhost/worker/fact-finder|
|Worker 1 / 2 Swagger-UI|http://\<dynamicIP\>:8080/fact-finder|
|Analytics|http://localhost/analytics/|
|Analytics Swagger-UI|http://localhost/analytics/swagger-ui|
