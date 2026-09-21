# AIOps Payment Service Simulation

## Scenario of case

This repository monitors a synthetic `payment-service`. The operational problem is identifying slow or resource-constrained payment requests and error logs early enough for an operations team to investigate them. AIOps is used here to turn telemetry into explainable anomaly events and carry those events through a lightweight event-processing workflow.

## Repository Components

- `data/service_data.json` contains timestamped service observations.
- `src/anomaly_detector.py` applies thresholds and checks error log levels.
- `src/event_producer.py` publishes detected events.
- `src/event_topic.py` provides the in-memory `anomaly-events` topic.
- `src/event_consumer.py` retrieves events from the topic.
- `src/aiops_pipeline.py` coordinates detection, publishing, consumption, and final AIOps output.
- `tests/` validates calculations and the anomaly event workflow.

## Operational Data Analysis

Each record represents one minute of `payment-service` activity. `timestamp` is an ISO-like timestamp and establishes the observation order and incident time. The metric fields are `response_time_ms`, `cpu_percent`, and `memory_percent`. The log fields are `log_level` and `message`; `service` identifies the source service.

The observations from `10:00` through `10:04` and `10:07` through `10:09` are normal: response time is 120-150 ms, CPU is 42-50%, memory is 51-57%, and each record has an `INFO` success message. The records at `10:05` and `10:06` are unusual. Response time rises to 610 and 640 ms, and the `10:06` record also reaches 94% CPU and 91% memory. Both records have `ERROR` logs describing a payment timeout and a database connection timeout.

## Detection Findings

The detector uses these strict greater-than thresholds:

- response time above 500 ms
- CPU above 80%
- memory above 80%
- log level equal to `ERROR`

The final report identifies two anomalies:

| Timestamp | Reasons |
| --- | --- |
| `2026-09-20T10:05:00` | High response time; Error log detected |
| `2026-09-20T10:06:00` | High response time; High CPU utilization; High memory utilization; Error log detected |

No expected anomaly was missed in this dataset, and no normal observation was incorrectly flagged. The detector is rule-based and uses fixed thresholds, so a limitation is that it does not learn service-specific baselines or detect gradual changes that remain below a threshold. A possible improvement would be a rolling baseline with configurable thresholds and additional message classification.

## Event-Processing Flow

The pipeline processes each operational record with `AnomalyDetector`. For each non-null anomaly event, `EventProducer` publishes the event to the shared in-memory `anomaly-events` `EventTopic`. `EventConsumer` reads the messages from that same topic, and the pipeline prints the consumed events as the downstream AIOps output. Each event contains the timestamp, service, anomaly type, reasons, and original source record.

## Issues Found and Corrected

1. The detector checked for `WARNING`, but the supplied concerning records use `ERROR`. The condition was corrected so error logs produce the `Error log detected` reason.
2. The producer wrote to `service-events` while the consumer read from a separate `anomaly-events` topic. The pipeline now creates one shared `anomaly-events` topic for both components.

## Final Execution

Running `python3 src/aiops_pipeline.py` produces:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
```

The output lists both `payment-service` anomaly events with their timestamps and detection reasons, demonstrating the flow from operational data through detection, event generation, producer, topic, consumer, and final AIOps output.

## Reproduce and Validate

From the repository root:

```bash
python3 -m pytest -q
python3 src/aiops_pipeline.py
```

The validation suite should report `9 passed`. The second command should report `10` records processed, `2` anomalies detected, and `2` events consumed.

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

