# allocation engine arch

bengaluru quick-commerce: **hsr layout warehouse → whitefield darkstore**. flow: customer tap → rider pickup → live tracking → otp delivery.

---

## system architecture

![system architecture](docs/diagrams/architecture.png)

physical supply chain moves stock from hsr warehouse to whitefield darkstore over the orr route. three apps (seller/hub, rider, customer) hit an api gateway with auth, rate limits, rest + websocket for tracking.

**core services:** order (create + state machine), inventory (stock per darkstore), allocation engine (assign riders), tracking (live location + eta), notification (push/sms/in-app), route optimizer (multi-drop batching), catalog, analytics.

**event bus (kafka / redis streams):** `order.created`, `rider.assigned`, `location.updated`, `stock.low`, `order.delivered`.

**data:** postgresql (orders, users, catalog), redis (rider state, sessions), timescaledb (gps time-series), firebase/fcm (push).

**replenishment:** `stock.low` triggers hsr → whitefield transfer (~18 km orr, 6 am window, hub manager alert).

**infra:** kubernetes, aws/gcp, google maps (traffic-aware), sms gateway, cdn.

---

## order allocation + tracking flow

**part 1** — customer order through rider pickup at whitefield.

![allocation flow part 1](docs/diagrams/allocation-flow-part1.png)

customer app → order service → inventory reserves whitefield stock. out of stock → notify + hsr restock. `order.created` on kafka feeds the allocation engine: eligible riders (online, near whitefield), scoring (load, proximity, rating), batching nearby drops, best rider via redis atomic lock (no double-assign). rider gets push + in-app map. reject → re-allocate; accept → pickup, `picked_up`, hub confirms.

**part 2** — live tracking through delivery.

![tracking flow part 2](docs/diagrams/tracking-flow-part2.png)

rider gps every 5s over websocket → tracking service stores gps + eta → customer app map + eta push. otp confirms delivery, status `delivered`. rider freed in redis (`available`). analytics logs time, sla, distance.

---

## quick ref

| piece | role |
|-------|------|
| allocation engine | score, batch, lock, assign riders |
| tracking service | gps stream, eta, customer updates |
| inventory svc | per-darkstore stock + reserve |
| redis | rider state, allocation locks |
| kafka | async domain events |

repo: https://github.com/Akhilesh29/allocation-engine-arch
