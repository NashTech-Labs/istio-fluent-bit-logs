# Istio logs with fluent-bit

## Prerequisites

* Kubernetes cluster 
* Helm Installed in your machine.
* Istio setup & proxy injection enabled
* Some pods running to generate logs
* Optionally **Elasticsearch & Kibana** to visuallize logs index


## Configurations

```yaml
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush {{ .Values.flush }}
        Log_Level {{ .Values.logLevel }}
        Parsers_File parsers.conf
        Parsers_File custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port {{ .Values.metricsPort }}
        Health_Check On

  inputs: |
    [INPUT]
        Name tail
        Tag_Regex  (?<pod_name>[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-
        Tag  kube.<container_name>.<namespace_name>
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Mem_Buf_Limit 20MB
        Skip_Long_Lines On

  filters: |
    [FILTER]
        Name                parser
        Match               kube.istio-proxy.*
        Key_Name            log
        Reserve_Data        On
        Parser              envoy

  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch-master
        Logstash_Format On
        Logstash_Prefix istio
        Retry_Limit False

  customParsers: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L

    [PARSER]
        Name    envoy
        Format  regex
        Regex ^\[(?<start_time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)? (?<protocol>\S+)" (?<code>[^ ]*) (?<response_flags>[^ ]*) (?<bytes_received>[^ ]*) (?<bytes_sent>[^ ]*) (?<duration>[^ ]*) (?<x_envoy_upstream_service_time>[^ ]*) "(?<x_forwarded_for>[^ ]*)" "(?<user_agent>[^\"]*)" "(?<request_id>[^\"]*)" "(?<authority>[^ ]*)" "(?<upstream_host>[^ ]*)"
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On
        Time_Key start_time

    [PARSER]
        Name    istio-envoy-proxy
        Format  regex
        Regex ^\[(?<start_time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)? (?<protocol>\S+)" (?<response_code>[^ ]*) (?<response_flags>[^ ]*) (?<response_code_details>[^ ]*) (?<connection_termination_details>[^ ]*) (?<upstream_transport_failure_reason>[^ ]*) (?<bytes_received>[^ ]*) (?<bytes_sent>[^ ]*) (?<duration>[^ ]*) (?<x_envoy_upstream_service_time>[^ ]*) "(?<x_forwarded_for>[^ ]*)" "(?<user_agent>[^\"]*)" "(?<x_request_id>[^\"]*)" (?<authority>[^ ]*)" "(?<upstream_host>[^ ]*)" (?<upstream_cluster>[^ ]*) (?<upstream_local_address>[^ ]*) (?<downstream_local_address>[^ ]*) (?<downstream_remote_address>[^ ]*) (?<requested_server_name>[^ ]*) (?<route_name>[^  ]*)
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On
        Time_Key start_time

    [PARSER]
        Name        k8s-nginx-ingress
        Format      regex
        Regex       ^(?<host>[^ ]*) - (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*) "(?<referer>[^\"]*)" "(?<agent>[^\"]*)" (?<request_length>[^ ]*) (?<request_time>[^ ]*) \[(?<proxy_upstream_name>[^ ]*)\] (\[(?<proxy_alternative_upstream_name>[^ ]*)\] )?(?<upstream_addr>[^ ]*) (?<upstream_response_length>[^ ]*) (?<upstream_response_time>[^ ]*) (?<upstream_status>[^ ]*) (?<reg_id>[^ ]*).*$
        Time_Key    time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name    kube-custom
        Format  regex
        Regex   (?<tag>[^.]+)?\.?(?<pod_name>[a-z0-9](?:[-a-z0-9]*[a-z0-9])?(?:\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-(?<docker_id>[a-z0-9]{64})\.log$

```

## Deployment

### 1. Clone the repo

```bash
git clone https://github.com/knoldus/istio-fluent-bit-logs.git
cd istio-fluent-bit-logs
```

### Deploy the chart with custom datapipeline configs

```bash
helm upgrade --install fluent-bit fluent-bit -f fluent-bit/custom-values.yaml --namespace logging --create-namespace
```