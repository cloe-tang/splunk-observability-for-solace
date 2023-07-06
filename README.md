# splunk-observability-for-solace
The purpose of this lab is to test out Solace can be instrumented and how it should appear in the APM service flow. Conclusion from the test is Solace can be instrumented and it should appear as a service in the service map instead of an inferred service. 

## Architecture
Following is the service flow setup for this lab. 

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/7b41b597-dfb8-408e-8343-d1c551e0ecbf)

## Pre-requisites
1. Docker installed
2. Java installed

Note that this will be using upstream otel collector instead of splunk otel collector. 

## Part 1 - Setting up Solace
Step 1: Clone the solace repository on your machine
```
git clone https://github.com/TamimiGitHub/solace-dt-demo.git
cd solace-dt-demo
```

Step 2: Edit the otel-collector-config.yaml. Add a sapm exporter to send traces to Splunk Observability. Change the value <ACCESS_TOKEN> and <REALM>. Replace the content in the file.
```
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1000
    spike_limit_mib: 500

  batch:

exporters:
  logging:
    #loglevel: "info"
    verbosity: detailed

  jaeger:
    endpoint: jaeger-all-in-one:14250
    tls:
      insecure: true

  sapm:
    access_token: "<ACCESS_TOKEN>"
    endpoint: "https://ingest.<REALM>.signalfx.com/v2/trace"

  signalfx:
    # See above for SFX_TOKEN and SFX_REALM.
    access_token: "<your splunk observability access token>"
    realm: "<REALM>"
receivers:
  otlp:
    protocols:
      grpc:

  solace:
    broker: [solbroker:5672]
    max_unacknowledged: 500
    auth:
      sasl_plain:
        username: trace
        password: trace
    queue: queue://#telemetry-trace
    tls:
      insecure: true
      insecure_skip_verify: true


service:
  telemetry:
    logs:
      level: "debug"
  pipelines:
    traces:
      receivers: [solace, otlp]
      processors: [batch, memory_limiter]
      exporters: [jaeger, logging, sapm]
```

Step 3: Once Step 2 is done, follow 3 - 11 in the solace guide to setup Solace (https://codelabs.solace.dev/codelabs/dt-otel/index.html?index=..%2F..index#2)

## Part 2 - Verify Solace Setup 
Step 1: Publishing messages using a simple jms application
```
[solace@dev solace-dt-demo]$ cd src
[solace@dev solace-dt-demo]$ java -Dsolace.host=localhost:55557 \
-Dsolace.vpn=default \
-Dsolace.user=default \
-Dsolace.password=default \
-Dsolace.topic=solace/tracing -jar solace-publisher.jar
```
If successfully published to the queue, you should see something similar to the following:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/b6607594-de86-43e4-b52a-aaeb85e99f90)

In the solace portal, you should see a message being published

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/2c8f7592-433e-4a87-9d39-cf48dd798383)

Step 2: Verify published messages are traced in the Jaeger UI

If Docker is running on the same system your browser is running on, you can access the Jaeger UI using the following URI: http://0.0.0.0:16686/ or http://localhost:16686/. If Docker is running on another system in your network, simply replace 0.0.0.0 to the system's IP, e.g. http://192.168.3.166:16686/.

After the OpenTelemetry Collector has received a message, you should be able to see the solbroker trace. Once the right service has been selected, select "Find Traces" button.

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/08647e8f-2232-42c8-a2ad-832c6496b38b)

Step 3: Verify traces are being sent to Splunk Observability APM

Access to Splunk Observability under APM. Click on explore to view Service Map. You should see solbroker as a service

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/3981928b-14a0-4735-87d5-08c9928d7162)

Click on the service solbroker, then "Traces" on the right. You should see something similar to the following

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/1c7142d4-0a26-47c6-83e7-b66099abb54f)

Click on the trace ID followed by the first span. You should see the following information:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/de972340-bfc3-4896-8eb8-31dd919743a9)

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/9d70c035-ba07-40af-a997-b4fec35d5ff3)

You can also verify from the otel collector logs by issuing the following command:

```
docker ps
```

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/b1ff11c4-2c31-413d-9910-d2061c022a3a)

```
docker logs 97b57b8e9555
```

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/7d89a7f8-0a47-4627-9384-410ef5310f01)

Above is what you should see in the logs if traces is successfully received by the collector

## Part 4 - Auto Instrumention (Publisher and Subscriber)

Step 1: Run the publisher with context propagation enabled

The following command will

1. Launch the opentelemetry java agent
2. Configure the agent with the solace opentelemetry JMS integration solace-publisher JMS extension
3. Run the JMS solace-publisher.jar application and publish a message
Additional context information will be automatically sent to the collector with no code changes to the JMS publishing application thanks to the opentelemetry javaagent.

```
java -javaagent:solace-dt-demo/src/opentelemetry-javaagent-all-1.19.0.jar \
-Dotel.javaagent.extensions=solace-dt-demo/src/solace-opentelemetry-jms-integration-1.0.0.jar \
-Dotel.propagators=solace_jms_tracecontext \
-Dotel.exporter.otlp.endpoint=http://localhost:4317 \
-Dotel.traces.exporter=otlp \
-Dotel.metrics.exporter=none \
-Dotel.instrumentation.jms.enabled=true \
-Dotel.resource.attributes="service.name=SolaceJMSPublisher" \
-Dsolace.host=localhost:55557 \
-Dsolace.vpn=default \
-Dsolace.user=default \
-Dsolace.password=default \
-Dsolace.topic=solace/tracing \
-jar solace-dt-demo/src/solace-publisher.jar
```

You should see the following if publisher successfully publish the message:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/6e6e55c7-94f0-4f17-beba-47ed64c5032b)


Step 2: Run the subscriber with context propagation enabled

The following command will

1. Launch the opentelemetry java agent
2. Configure the agent with the solace opentelemetry JMS integration solace-publisher JMS extension
3. Run the JMS solace-queue-receiver.jar application and consume the message that was just published on to the queue

```
java -javaagent:solace-dt-demo/src/opentelemetry-javaagent-all-1.19.0.jar \
-Dotel.javaagent.extensions=solace-dt-demo/src/solace-opentelemetry-jms-integration-1.0.0.jar \
-Dotel.propagators=solace_jms_tracecontext \
-Dotel.exporter.otlp.endpoint=http://localhost:4317 \
-Dotel.traces.exporter=otlp \
-Dotel.metrics.exporter=none \
-Dotel.instrumentation.jms.enabled=true \
-Dotel.resource.attributes="service.name=SolaceJMSQueueSubscriber" \
-Dsolace.host=localhost:55557 \
-Dsolace.vpn=default \
-Dsolace.user=default \
-Dsolace.password=default \
-Dsolace.queue=q \
-Dsolace.topic=solace/tracing \
-jar solace-dt-demo/src/solace-queue-receiver.jar
```

You should see the following if the subscriber successfully consume the message

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/32c8f3cd-992d-43c3-b3fb-21d834b89594)

## Part 5 - Verify Auto Instrumentation Result

Step 1: Verify in Jaeger

You should see 3 trace spans: SolaceJMSPublisher, solbroker, SolaceJMSQueueSubscriber

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/fb2e834f-fcac-49a5-8c93-3e2df9c1ec4a)

Click into the trace. You should see the waterfall similar to the following:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/e13343b1-9f56-4f0a-b218-fb88175ca315)

- The first SEND span was generated by the publisher when the message was published.
- The second RECEIVE span was generated by the PubSub+ Broker when the message was received.
- The third PROCESS span was generated by the consumer when the message was consumed.

Step 2: Verify in Splunk Observability 

In the Service Flow, you should see something similar to the following:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/d27b0ce7-c3da-4cae-9afc-0b10c0b1d924)

jms:solace/tracing is the topic that is being published to and subscribed from. 
solbroker is the queue manager. It should be shown if properly instrumented. 

When navigate to the trace ID, you should see the following water fall. 

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/b4eb5668-c5b0-4776-9caa-379af133d94e)

Click on solbroker, you should see the following information:

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/d539103a-dcf4-40dc-af93-7b65a087182e)
