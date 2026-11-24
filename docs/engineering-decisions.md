# Engineering decisions

These are the design choices that mattered most for throughput and reliability in the SMSC, and the trade-offs behind each one.

## 1. Separate processes for the SMPP edge, routing core, and SS7 signaling

The platform runs as separate processes: the **SMS gateway (ESME)**, the **UMG** (SMPP server and routing core), one **signaling gateway** per operator link, and the **simulator**. They talk over a small internal SMS signaling protocol on TCP.

- **Why:** each layer fails and scales differently. SMPP clients connect and disconnect all the time, while an SS7 link is a long-lived SCTP association that must stay up. Separate processes mean a misbehaving client can't take down the signaling link, and an operator link can be restarted without dropping client binds.
- **Trade-off:** one more hop and one more protocol to maintain. The internal protocol was kept deliberately small (routing query, forward, report, alert, and their responses) so it stayed easy to debug from logs.

## 2. One signaling gateway instance per operator link

Each operator connection has its own gateway process, with its own SCTP association, M3UA routing context, point codes, and routing table.

- **Why:** isolation. A misconfigured or flapping link affects only its own traffic, configuration changes are scoped to one operator, and capacity can be added per link.
- The UMG picks the link per message using **least-cost routing** on the destination number prefix.

## 3. Correlating asynchronous MAP dialogues

Delivering one SMS takes at least two TCAP dialogues: **SRI-for-SM** to the HLR, then **MT-ForwardSM** to the MSC. A multipart message needs one forward per part. Thousands of these are in flight at once.

- Each SMPP message gets an internal **message ID**, and each TCAP dialogue gets a **transaction ID** that encodes the operation and the part number.
- Bounded, pre-allocated lookup tables map transaction ID → message ID → part. Responses are matched by direct index, with fine-grained per-entry locks instead of one global lock.
- When all parts have a result, the message is finished: a CDR is written and a delivery receipt goes back to the client.

## 4. Concatenated (multipart) SMS

Long or Unicode texts are split into parts with a **UDH concatenation header** (reference number, total parts, sequence).

- On the way out, each part is forwarded in order through a dedicated multipart handler, with a short configurable delay between parts so the receiving handset can reassemble them reliably.
- Results for each part are collected with a timeout. A message counts as delivered only when every part is acknowledged, and a part that fails is reported against the whole message.
- GSM 7-bit and UCS-2 encodings are handled explicitly, because the coding scheme decides the part size (160/153 versus 70/67 characters).

## 5. Retries that follow the network's own signals

Not every failure deserves an immediate retry.

- **Transient routing failures** (a timeout or no response to SRI-SM) go to a dedicated **SRI-SM retry queue** with its own worker, so retries never block fresh traffic.
- **Absent subscriber** (phone off or out of coverage) isn't retried blindly. The gateway sends **ReportSM-DeliveryStatus** so the HLR records that a message is waiting. When the subscriber is reachable again, the HLR sends **AlertServiceCentre**, and that alert triggers redelivery. This saves signaling load and delivers the message as soon as the phone comes back.

## 6. Shared-memory IPC queues between threads and processes

Hand-offs between pipeline stages (submit, routing query, forward, response, report) use **System V IPC message queues and shared memory**, with one sub-queue per account or link.

- **Why:** fast hand-off between processes on the same host without a broker, and the stages stay decoupled. A slow operator link fills only its own queue.
- Queue depths are exposed through the **command interface**, so operations can see back-pressure building before it turns into dropped traffic.

## 7. Throughput accounting and throttling

- Per-account and per-operation counters (submit, SRI-SM, forward, and their responses) are kept **per second** in fixed arrays indexed by minute and second. That makes TPS sampling O(1) with no locking on the hot path, and the counters reset daily.
- The ESME side throttles submission per channel to the agreed rate. Separate queues per application keep one busy client from starving the others.
- TPS snapshots are logged and uploaded on a schedule for capacity planning. That data showed the platform sustaining about **3,000 TPS at production peak**.

## 8. CDRs in monthly tables

Every message produces a CDR (client, route, parts, timestamps per stage, final status).

- CDRs are written by a dedicated writer thread off the hot path, and the table is **rolled into monthly tables**. Inserts stay fast, reporting on recent data scans less, and old months can be archived as a unit.

## 9. Testing against a simulator first

The **MSC/HLR simulator** is the signaling gateway's own code run in the peer role.

- It reuses the real encoders and decoders, so tests exercise the same TCAP/MAP paths that production uses.
- It can inject errors such as absent subscriber and timeouts, so retry, report, and alert paths are tested on purpose rather than discovered in production.
- Load tests (TPS ramps, multipart-heavy traffic, long soak runs) ran against it, and packet captures were used to check encodings.

## 10. Operability

- A **command server** on each process exposes runtime state, such as binds, queue depths, and TPS, to operations staff.
- **SNMP traps** report link state changes (SCTP/M3UA up or down) and error conditions to the monitoring system.
- Logging is leveled and can go to rotating files or a central log server, with optional per-number tracing for troubleshooting a single subscriber.
