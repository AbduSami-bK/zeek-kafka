# Zeek-Kafka Agent Guide

## Project Purpose
`zeek-kafka` is a Zeek logging writer plugin (`Seiso::Kafka`) that publishes Zeek log records to Kafka via `librdkafka`.

Core path:
- C++ writer implementation: `src/KafkaWriter.cc`, `src/KafkaWriter.h`
- Zeek script API and defaults: `scripts/init.zeek`
- Package metadata for `zkg`: `zkg.meta`

## How It Works
1. Zeek loads `packages/zeek-kafka`.
2. Script-side settings (`Kafka::*`) are exported from `scripts/init.zeek`.
3. `KafkaWriter` copies settings into thread-local state in its constructor.
4. On init, the writer builds a `librdkafka` producer and resolves the Kafka topic.
5. On each log record, it serializes JSON (optionally tagged), attaches headers/key, and produces to Kafka.

## Key Configuration Surface
- `Kafka::kafka_conf`: passthrough `librdkafka` config table.
- `Kafka::topic_name`: default topic for records.
- `Kafka::logs_to_send` / `Kafka::send_all_active_logs` / `Kafka::logs_to_exclude`: log routing.
- `Kafka::tag_json`: wraps payload as `{ "<log_type>": { ... } }`.
- `Kafka::additional_message_values`: static JSON fields (used when tagging JSON).
- `Kafka::key_name` and `Kafka::headers`: Kafka key and message headers.

## Compatibility Notes (as of 2026-04-16)
- Kafka broker target: Apache Kafka 4.x (including 4.2.0).
- Zeek target: Zeek 8.x (8.0.x LTS and 8.1.x current feature line).
- Recommended client library: `librdkafka >= 2.12.0`.
- Prefer `bootstrap.servers` in config examples. `metadata.broker.list` remains a legacy alias in `librdkafka`.

## Safe Change Guidelines
- Keep script API stable (`Kafka::*` names) unless intentionally versioning.
- Treat `Kafka::kafka_conf` as opaque passthrough to `librdkafka`; avoid hardcoding broker behavior in C++.
- If modifying writer send path, preserve:
  - JSON formatting parity for existing tests.
  - header/key behavior.
  - clean shutdown semantics (`DoFinish`, flush/poll loop).
- For dependency bumps, update both `zkg.meta` and README examples together.

## Quick Validation Checklist
- Build plugin against target Zeek and `librdkafka`.
- Run `zeek -N Seiso::Kafka` to confirm plugin load.
- Run `tests/` (`btest`) for script-level behavior.
- End-to-end sanity: publish a small pcap and consume from Kafka topic.
