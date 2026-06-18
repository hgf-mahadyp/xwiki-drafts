##In Review
#newsletter #datadog 
Preamble: This article is entirely hand written. No AI agents or LLMs were used in it's creation, the spelling and grammatical error will attest to this :)

In todays article we're going to look at what it takes to get your application communicating with Datadog. For those of you who have not heard about Datadog before; Datadog is the platform we are using to monitor the HTS product. I would highly recommend the intro series from Marcus Held on Datadog [Datadog Intro Series](https://hogrefe.sharepoint.com/sites/E-Assessment/Freigegebene%20Dokumente/Forms/redminetickets.aspx?FolderCTID=0x012000F74E5613EA563E419A6459ACD997ED3F&id=%2Fsites%2FE%2DAssessment%2FFreigegebene%20Dokumente%2FGeneral%2FRecordings%2FHTS%20LTS%2FDatadog%20Intro%20Series) For more information on how to use Datadog I cannot recommend highly enough their learning platform: https://learn.datadoghq.com/bundles/core-skills-learning-path.

## Typical setup
All of our Netuse hosts come pre-installed with a datadog-agent. This datadog agent sends logs from our host to the datadog platform. The applications themselves, at least all the newer developed applications, run as part of a containerized aka "docker" runtime. All logs that are sent to the containers stdout, so everything you see when you run the command "docker logs <container-name>" are collected by the Datadog agent and sent to the Datadog platform.
Metrics and traces are sent from the application container to a "side-car" opentelemetry or "OTEL" container which subsequently sends them to the Datadog platform.

Below is a diagram that depicts this setup.

 ```mermaid
stateDiagram-v2
    state Host {    
      Application --> DatadogAgent: Logs
    
      state Container {
        Application --> OTELCollector: Metrics/Traces
      }
}
DatadogAgent --> DatadogPlatform
OTELCollector --> DatadogPlatform
```

## Case Study - Partner Enablement
Our colleagues over at Partner Enablement have been busy creating the new HSI REST API. They are ready to start deploying to our Netuse infrastructure, and as is the case with all our deployed applications, need to integrate monitoring into their deployment. 
They have already started the integration of monitoring into the deployment. However, based on our experience within the Infra Team, there are some improvements that we can add. Lets go!

### Host Setup
First things first, we need to make sure that the datadog-agent is active and enabled on the target host. To check this we'll run the command:

```bash
systemctl status datadog-agent
```
This command is returning:
```txt
datadog-agent.service - Datadog Agent Loaded: loaded ([/lib/systemd/system/datadog-agent.service](file://daryl-ringhand/lib/systemd/system/datadog-agent.service);

disabled

; preset:

enabled

) Active: inactive (dead)
```

So here we can see that the agent is disabled and off. The Infra team will need to change some configuration to enable the agent.

**Note: ** All hosts in the dc_integration hostgroup have their datadog agents disabled off by default. You will need to explicitly request that they be turned on if that is required.

### Sending Logs
As mentioned previously, we have standardized the log collection from docker containers so that everything that is sent to a containers stdout will be automatically forwarded to the Datadog platform via the Datadog-agent. However, in order to get more out of these logs in terms of searchability and correlatability we should send those logs in json format with some specific fields included. How and where to define this json log format will differ depending on the technology you are running your application on. In the case of Partner Enablement and the HSI Rest client they are using Java Springboot. Will need to make the required changes for Java Springboot to send logs to stdout in json format.

#### Java Springboot Json Logs
Currently we have the following logback-spring.xml file defined:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <springProfile name="local,local-docker,deployed">
        <appender name="OTEL" class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender"/>

        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="OTEL"/>
        </root>
    </springProfile>
</configuration>
```
We want to have something like this instead:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <appender name="jsonstdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <providers>
                <timestamp>
                    <timeZone>CEST</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "level": "%level",
                        "service": "HSI-REST",
                        "traceId": "%X{X-B3-TraceId:-}",
                        "spanId": "%X{X-B3-SpanId:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "message": "%message"
                        }
                    </pattern>
                </pattern>
                <stackTrace>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>30</maxDepthPerThrowable>
                        <maxLength>2048</maxLength>
                        <shortenedClassNameLength>20</shortenedClassNameLength>
                        <rootCauseFirst>true</rootCauseFirst>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>
    <root level="info">
      <springProfile name="local, local-docker">
        <appender-ref ref="CONSOLE" />
      </springProfile>
      <springProfile name="deployed">
        <appender-ref ref="jsonstdout" />
      </springProfile>
    </root>
</configuration>
```
It's likely that we will need to install the relevant logging classes for this. 

**Note**: we should set the json logging in the deployment repo for clarity and flexibility (we won't need to redeploy to change logging settings). We can achieve this by mounting the logging file in the container and adding the environment variable specifying the path to the log configuration file:

```bash
JAVA_OPTS=-Dlogging.config='file:///your/file/location/logback.xml'
```

### Metrics and Traces
In accordance with our setup diagram we want to send the metrics and traces of the application via an OTEL sidecar container to the Datadog Platform. We have specifically created an OTEL docker image for this purpose and it has the advantage that it is considered as part of the Datadog-agent on the host and we therefore do not have to pay extra for it.

#### OTEL Collector Setup
Partner enablement are using docker swarm for their deployment but the following will work, with minor adjustments, with either simple docker run commands or a docker compose file.

The docker swarm file defining the deployment currently looks like this:

``` yml
version: "3.8"

services:
# TODO: potentially separate this so caddy can be reloaded without needing forced restart
  reverse-proxy:
    # pinned version - same as in reporting Dockerfile
    image: caddy:2.10.2
    restart: unless-stopped
    # deploy:
    #   replicas: 1
    #   update_config:
    #     parallelism: 1
    #     delay: 10s
    #     order: start-first
    #     failure_action: rollback
    #     monitor: 30s
    #   rollback_config:
    #     parallelism: 1
    #     order: start-first
    ports:
      - "8080:8080"
    environment:
        - HSI_REST_BASE_URL=http://hsi-rest-spring-boot:8080
        - HSI_REST_API_SPECS_ENABLED=true
    volumes:
      - ${DEPLOY_PATH}/caddy:/etc/caddy
      - ${DEPLOY_PATH}/static-hosting:/srv
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - hsi-rest-network

  
  hsi-rest-spring-boot:
    # image: ghcr.io/hogrefe-digital/hsi-rest:${HSI_REST_VERSION}
    # TODO remove hard-coded version, for testing only
    image: ghcr.io/hogrefe-digital/hsi-rest:20260617T131417Z
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
        monitor: 60s
      rollback_config:
        parallelism: 1
        order: start-first
    environment:
      SPRING_PROFILES_ACTIVE: deployed

      APP_HAL_BASE_URL: ""
      # APP_HSI_CLASSIC_BASE_URL: "http://10.231.25.11:8080/HTSEnvironment/main/services/HSIOnlineJsonRPC"
      # APP_HSI_CLASSIC_BASE_URL_V5: "http://10.231.25.11:8080/HTSEnvironment/main/services/v5/HSIOnlineJsonRPC"
      # TODO simplify to only v5?
      APP_HSI_CLASSIC_BASE_URL: "https://integration.hogrefe-ws.com/HTSEnvironment/services/HSIOnlineJsonRPC"
      APP_HSI_CLASSIC_BASE_URL_V5: "https://integration.hogrefe-ws.com/HTSEnvironment/services/v5/HSIOnlineJsonRPC"

      OTEL_EXPORTER_OTLP_METRICS_ENDPOINT: "http://datadog-agent:4318/v1/metrics"
      OTEL_EXPORTER_OTLP_LOGS_ENDPOINT: "http://datadog-agent:4318/v1/logs"
      OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: "http://datadog-agent:4318/v1/traces"


    secrets:
      - source: app_jwt_secret
        target: secret.app.jwt.secret

      - source: app_hsi_classic_functional_user_serial
        target: secret.app.hsi-classic.functional-user.serial

      - source: app_hsi_classic_functional_user_password
        target: secret.app.hsi-classic.functional-user.password

    
    networks:
      - hsi-rest-network


  datadog-agent:
#    https://github.com/DataDog/datadog-agent/issues/32947?utm_source=chatgpt.com
    # image: gcr.io/datadoghq/agent:7.60.1
    image: gcr.io/datadoghq/agent:7
    
    environment:
      DD_SITE: datadoghq.eu

      # new feature in datadog agent 7.55+ to read secrets from docker secrets
      # DD_SECRET_BACKEND_TYPE: "docker.secrets"
      # DD_API_KEY: "ENC[dd_api_key]"

      DD_LOG_LEVEL: "debug"

      DD_APM_ENABLED: "true"
      DD_LOGS_ENABLED: "true"

      DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_HTTP_ENDPOINT: "0.0.0.0:4318"
      DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_GRPC_ENDPOINT: "0.0.0.0:4317"
      DD_OTLP_CONFIG_LOGS_ENABLED: "true"

      # system-probe requires Linux kernel features (eBPF) unavailable on macOS/Docker Desktop
      DD_SYSTEM_PROBE_ENABLED: "false"
      # disable process agent if process-level metrics are not needed locally
      DD_PROCESS_AGENT_ENABLED: "false"

      # Workaround for Agent 7.61+ OTLP pipeline failure in Docker
      HOST_PROC: "/proc"
    
    # HACK to convert docker secret to env var, since datadog agent does not support docker secrets natively yet (as of version 7.55) and later image does not work yet with our config
    command:
      - sh
      - -c
      - |
        export DD_API_KEY="$$(cat /run/secrets/dd_api_key)"
        exec /init

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro

    ports:
      - "4318:4318"
      - "4317:4317"

    networks:
      - hsi-rest-network

    secrets:
      - dd_api_key
  # TODO add open telementry collector container
  # otel-collector:
  #   # image: ghcr.io/hogrefe-digital/hts-reporting-otel-collector:${SERVICE_VERSION}
  #   container_name: otel-collector
  #   restart: unless-stopped
  #   # env_file:
  #     # - otel-collector.env
  #   networks:
  #     - hsi-rest-network

networks:
  hsi-rest-network:
    driver: overlay

volumes:
  caddy_data:
  caddy_config:

secrets:
  app_jwt_secret:
    external: true
  app_hsi_classic_functional_user_serial:
    external: true
  app_hsi_classic_functional_user_password:
    external: true
  dd_api_key:
    external: true

```

We can remove the datadog-agent and replace it with our OTEL Collector side car container:

```yml
version: "3.8"

services:
# TODO: potentially separate this so caddy can be reloaded without needing forced restart
  reverse-proxy:
    # pinned version - same as in reporting Dockerfile
    image: caddy:2.10.2
    restart: unless-stopped
    # deploy:
    #   replicas: 1
    #   update_config:
    #     parallelism: 1
    #     delay: 10s
    #     order: start-first
    #     failure_action: rollback
    #     monitor: 30s
    #   rollback_config:
    #     parallelism: 1
    #     order: start-first
    ports:
      - "8080:8080"
    environment:
        - HSI_REST_BASE_URL=http://hsi-rest-spring-boot:8080
        - HSI_REST_API_SPECS_ENABLED=true
    volumes:
      - ${DEPLOY_PATH}/caddy:/etc/caddy
      - ${DEPLOY_PATH}/static-hosting:/srv
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - hsi-rest-network

  
  hsi-rest-spring-boot:
    # image: ghcr.io/hogrefe-digital/hsi-rest:${HSI_REST_VERSION}
    # TODO remove hard-coded version, for testing only
    image: ghcr.io/hogrefe-digital/hsi-rest:20260617T131417Z
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
        monitor: 60s
      rollback_config:
        parallelism: 1
        order: start-first
    environment:
      SPRING_PROFILES_ACTIVE: deployed

      APP_HAL_BASE_URL: ""
      # APP_HSI_CLASSIC_BASE_URL: "http://10.231.25.11:8080/HTSEnvironment/main/services/HSIOnlineJsonRPC"
      # APP_HSI_CLASSIC_BASE_URL_V5: "http://10.231.25.11:8080/HTSEnvironment/main/services/v5/HSIOnlineJsonRPC"
      # TODO simplify to only v5?
      APP_HSI_CLASSIC_BASE_URL: "https://integration.hogrefe-ws.com/HTSEnvironment/services/HSIOnlineJsonRPC"
      APP_HSI_CLASSIC_BASE_URL_V5: "https://integration.hogrefe-ws.com/HTSEnvironment/services/v5/HSIOnlineJsonRPC"

      OTEL_EXPORTER_OTLP_METRICS_ENDPOINT: "http://datadog-agent:4318/v1/metrics"
      OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: "http://datadog-agent:4318/v1/traces"


    secrets:
      - source: app_jwt_secret
        target: secret.app.jwt.secret

      - source: app_hsi_classic_functional_user_serial
        target: secret.app.hsi-classic.functional-user.serial

      - source: app_hsi_classic_functional_user_password
        target: secret.app.hsi-classic.functional-user.password

    
    networks:
      - hsi-rest-network


  otel-collector:
    image: ghcr.io/hogrefe-digital/hts-otelcol:1.0.2
    
    environment:
      LOG_LEVEL=debug

    command: "--config=/etc/otel/config.yaml"
    volumes:
      - {{ otel_collector_config_dir }}/otel-collector-config.yml:/etc/otel/config.yaml:ro

    networks:
      - hsi-rest-network

    secrets:
      - dd_api_key
    
networks:
  hsi-rest-network:
    driver: overlay

volumes:
  caddy_data:
  caddy_config:

secrets:
  app_jwt_secret:
    external: true
  app_hsi_classic_functional_user_serial:
    external: true
  app_hsi_classic_functional_user_password:
    external: true
  dd_api_key:
    external: true
```

The otel collectors config directory looks something like this:

``` json
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_max_size: 100
    send_batch_size: 10
    timeout: 10s

connectors:
  datadog/connector:

exporters:
  datadog:
    api:
      site: datadoghq.eu
      key: '{{ datadog_api_key }}'
    hostname: '{{ ansible_facts["fqdn"] }}'

extensions:
  datadog:
    api:
      site: datadoghq.eu
      key: '{{ datadog_api_key }}'
    deployment_type: daemonset
    hostname: '{{ ansible_facts["fqdn"] }}'

service:
  extensions: [datadog]
  telemetry:
    logs:
      encoding: json
      level: info
    resource:
      team.name: 'partner-enablement'
      deployment.environment.name: 'dc_{{ configuration_name }}'
  pipelines:
    metrics:
      receivers: [otlp, datadog/connector]
      processors: [batch]
      exporters: [datadog]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog/connector, datadog]

```

Some things we need to figure out are how to inject the secrets requirement into arbitrary files like the one above.....
