# splunk-observability-for-solace
The purpose of this lab is to test out Solace can be instrumented and how it should appear in the APM service flow. Conclusion from the test is Solace can be instrumented and it should appear as a service in the service map instead of an inferred service. 

## Architecture
Following is the service flow setup for this lab. 

![image](https://github.com/cloe-tang/splunk-observability-for-solace/assets/58005106/7b41b597-dfb8-408e-8343-d1c551e0ecbf)

## Pre-requisites
1. Docker installed
2. Java installed

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

