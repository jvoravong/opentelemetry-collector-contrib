# Troubleshooting

## Addressing metrics that are scaled differently between the SignalFX Agent and Splunk Otel Collector
You may notice some metrics collected by the SignalFX Agent are scaled differently from the metrics collected by the
Splunk Otel Collector. This can be an issue if not addressed when migrating from the SignalFx Agent to the Splunk Otel
Collector. Here is an example of how this can happen and how to address it.

Issue:

In Kubernetes, you can define cpu resources with cpu units or millicpu units. See the
[Kubernetes CPU resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu)
documentation for a full break down on cpu resources. The SignalFx Agent reports millicpu units while the
Splunk Otel Collector reports cpu units (the new standard). This means the agent and collector can report the same
metric but the values would be different by a scale of 1000.

```
SignalFx Agent Reports:
    k8s.container.cpu_request: 0.1 (cpu units)
    k8s.container.cpu_limit: 0.2 (cpu units)

Splunk Otel Collector Reports:
    k8s.container.cpu_request: 100 (millicpu units)
    k8s.container.cpu_limit: 200 (millicpu units)
```

Solution:

Use the translation rules convert_values and multiply_float in the Splunk Otel Colletor to scale down the metrics to
match the output of the SignalFX Agent. In the collector, the k8s.container.cpu_request and k8s.container.cpu_limit
metrics are integer values. In order to use the translation rules to scale down these metrics, we must first convert the
metrics from integer to double values using convert_values and then scale the metrics down using scale_factors_float.
Here is a solution of what you would add to your collector config file.

```Splunk Otel Collector config
exporters:
  signalfx:
    translation_rules:
      - action: convert_values
        types_mapping:
          k8s.container.cpu_request: double
          k8s.container.cpu_limit: double
      - action: multiply_float
        scale_factors_float:
          k8s.container.cpu_request: 0.001
          k8s.container.cpu_limit: 0.001
```

This solution is only meant to ease the migration process from the SignalFX Agent to the Splunk Otel Collector. Once
fully migrated to the Splunk Otel Collector, it is recommended to use unmodified metric values and update any
related dashboards and alerts accordingly.