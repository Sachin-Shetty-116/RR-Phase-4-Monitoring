# Phase 4 - Incident Response Submission

**Team ID:** RR-Phase-4

**Team name:** Release Rush Team

**Team members:** Sachin Shetty (Sachin-Shetty-116)

**Incident ID:** RR-INC-05

## 1. Incident summary

The Inventory API production environment experienced an acute increase in error rates (rising from <1% to 36.2%) and severe response latency degradation (P95 latency surging from ~150 ms to 5,100 ms). Users and downstream clients received HTTP 503 Service Unavailable responses due to connection pool exhaustion, despite overall request volume and host CPU utilization remaining normal (~140 req/min, ~30% CPU).

## 2. Root cause

In `app/inventory.py` within the `lookup_inventory` function, a database connection is acquired from the pool (`connection = pool.acquire()`), but the release call (`pool.release(connection)`) was positioned only after query completion without exception handling.

When transient database query timeouts occurred (`DatabaseTimeout` after 4000ms), the exception was raised before `pool.release(connection)` could be reached. This caused leaked database connections to remain permanently in the active state. Repeated timeouts quickly saturated the connection pool (`active=10/10`), causing subsequent client requests to be immediately rejected with `PoolExhausted` and fail with HTTP 503.

## 3. Evidence

1. **Application Log (`incident/application.log`)**:
   - At `14:30:02Z` (`req-601`), `14:30:07Z` (`req-602`), and `14:30:12Z` (`req-603`), database query timeouts occurred (`db_query_timeout timeout_ms=4000`), but no `pool_release` events were logged. Active connections climbed from 8 to 9 to 10.
   - At `14:30:17Z`, `req-604` failed immediately:
     `ERROR request_id=req-604 event=pool_acquire_rejected reason=pool_exhausted active=10 max=10`
     `WARN request_id=req-604 event=request_failed status=503 duration_ms=4802`
2. **Alerts & Metrics (`incident/alerts.json` & `incident/metrics.csv`)**:
   - At `14:31:20Z`, `alerts.json` recorded three firing critical alerts: `InventoryApiHighErrorRate` (36.2%), `InventoryApiHighP95Latency` (5100 ms), and `DatabasePoolSaturation` (`10 / 10 (100%)`).
   - `metrics.csv` shows that starting at `14:30:00Z`, error rate jumped to 18.4% and P95 latency to 2,100 ms with `pool_active=8`, reaching 36.2% error rate and 5,100 ms latency with `pool_active=10/10` by `14:40:00Z`, while request rate remained steady (~140 rpm) and CPU remained normal (30%).

## 4. Corrective action

Refactored `lookup_inventory` in `app/inventory.py` to wrap the database lookup inside a `try...finally` block:
```python
def lookup_inventory(sku, pool=None, simulate_timeout=False):
    if pool is None:
        pool = ConnectionPool()

    connection = pool.acquire()

    try:
        if simulate_timeout:
            raise DatabaseTimeout("inventory query exceeded 4 second timeout")

        result = STOCK.get(sku, 0)
        return result
    finally:
        pool.release(connection)
```
The `finally` clause guarantees that `pool.release(connection)` is unconditionally executed upon exit, whether the query succeeds, encounters a timeout, or raises any other exception.

## 5. Validation

### Unit Test Summary:
```text
test_database_timeout_releases_connection (test_inventory.InventoryIncidentTests.test_database_timeout_releases_connection) ... ok
test_known_sku_returns_stock (test_inventory.InventoryIncidentTests.test_known_sku_returns_stock) ... ok
test_pool_does_not_exhaust_after_repeated_timeouts (test_inventory.InventoryIncidentTests.test_pool_does_not_exhaust_after_repeated_timeouts) ... ok
test_service_recovers_after_database_timeout (test_inventory.InventoryIncidentTests.test_service_recovers_after_database_timeout) ... ok
test_successful_request_releases_connection (test_inventory.InventoryIncidentTests.test_successful_request_releases_connection) ... ok
test_unknown_sku_returns_zero (test_inventory.InventoryIncidentTests.test_unknown_sku_returns_zero) ... ok

----------------------------------------------------------------------
Ran 6 tests in 0.000s

OK
```

### Recovery Verifier Output:
```text
RECOVERY VERIFIED
Repeated database timeouts did not leak connections.
A normal inventory request succeeded after the timeouts.
FLAG{POOL_RELEASE_RECOVERED}
```

## 6. Final flag

`FLAG{POOL_RELEASE_RECOVERED}`
