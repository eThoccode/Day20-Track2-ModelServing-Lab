# 02 — Server Load Results

## Concurrency 10

- Aggregated reqs: 6
- Failures: 0 (0.00%)
- Avg latency: 28605 ms
- Median latency (P50/TTFB proxy): 24000 ms
- E2E P95: 55000 ms
- E2E P99: 55000 ms
- Total RPS: 0.10

## Concurrency 50

- Aggregated reqs: 6
- Failures: 0 (0.00%)
- Avg latency: 24300 ms
- Median latency (P50/TTFB proxy): 21000 ms
- E2E P95: 41000 ms
- E2E P99: 41000 ms
- Total RPS: 0.13

## Notes

- Runs were executed against the local `make serve` Python llama.cpp server on `http://localhost:8080`.
- `make smoke` passed before the load tests.
- The Python server responded successfully to chat requests, but `/metrics` was not exposed on this path (`GET /metrics -> 404`).
- On this laptop, latency stayed high and throughput stayed low under both 10-user and 50-user runs, which suggests the local stack was capacity-limited rather than request-failure-limited during the successful run.
