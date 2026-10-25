# Structured Task Collaboration Walkthroughs

*This document is non-normative.* It contains examples in the supply chain space used to exercise the
[Structured Task Collaboration](./task-collaboration.md) (STC) specification. Each example shows
concrete endpoint calls, records, and events. The three use cases are a streams-only early warning investigation, a
warranty claim (request/answer), and an 8D analysis with an evolving report.

## Conventions

|          | OEM                           | Supplier                        |
| -------- | ----------------------------- | ------------------------------- |
| Identity | `did:web:oem.example.com`     | `did:web:tier1.example.com`     |
| Base URL | `https://oem.example.com/stc` | `https://tier1.example.com/stc` |

The examples use the following stream types and request kinds:

| URI                                                     | Contents                                | Packaging |
| ------------------------------------------------------- | --------------------------------------- | --------- |
| `https://w3id.org/dsaf/quality/v1/complaint`         | Customer complaints (OEM)               | parquet   |
| `https://w3id.org/dsaf/quality/v1/part-analysis`     | Returned-part analyses (supplier)       | parquet   |
| `https://w3id.org/dsaf/warranty/v1/claim`        | Warranty claims (OEM)                   | parquet   |
| `https://w3id.org/dsaf/warranty/v1/verification` | Warranty claim verifications (supplier) | parquet   |
| `https://w3id.org/dsaf/quality/v1/report-8d`          | 8D reports (supplier)                   | parquet   |
| `https://w3id.org/dsaf/quality/v1/attachment`         | Binary attachments (both)               | zip       |
| `https://w3id.org/dsaf/warranty/v1/settlement-claim`  | Warranty claim (request kind)           | n/a       |
| `https://w3id.org/dsaf/quality/v1/8d-request`                | 8D analysis demand (request kind)       | n/a       |

The examples also use the domain event `org.dsaf.quality.EarlyWarning.v1`.

HTTP examples elide authorization headers. Every call runs over the authenticated channel, and the caller's
verified identity is what `author` and event `source` fields must match. Parquet files are shown as tables, with
envelope columns first and `refs` serialized as a JSON column. Record identifiers are UUIDs, abbreviated in the
tables. Prose and the envelope `key` member contain the readable domain names (claim-b, analysis-77, …).

The task used throughout: `430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17` (below: `430f56d3-…`).

---

## Use Case 1: Early Warning Investigation (streams only)

The OEM's fleet analytics flag an unusual failure rate in a brake component supplied by the supplier. No
request/response is needed. Both parties publish data, correlate, and close.

```mermaid
sequenceDiagram
    participant OEM
    participant Supplier
    OEM->>Supplier: 1. POST /tasks (OPEN)
    Supplier-->>OEM: 201 Task (OPEN)
    Note over OEM: 2. Publish complaint file seq 0
    OEM->>Supplier: 2. POST /notifications FilesAppended (complaint seq 0)
    Supplier-->>OEM: 204
    OEM->>Supplier: 3. POST /notifications EarlyWarning
    Supplier-->>OEM: 204
    Supplier->>OEM: 4. GET /tasks/{taskId}/index
    OEM-->>Supplier: 200 complaint stream, seq 0
    Supplier->>OEM: 4. GET /tasks/{taskId}/files/{fileId}
    OEM-->>Supplier: 200 parquet bytes
    Note over Supplier: 5. Publish part-analysis seq 0 and attachment seq 0
    Supplier->>OEM: 5. POST /notifications FilesAppended (both streams)
    OEM-->>Supplier: 204
    Note over OEM: 6. Publish complaint file seq 1
    OEM-xSupplier: 6. FilesAppended (complaint seq 1) is lost
    Supplier->>OEM: 7. GET /tasks/{taskId}/index
    OEM-->>Supplier: 200 complaint stream head at seq 1
    Supplier->>OEM: 7. GET /tasks/{taskId}/files/{fileId}
    OEM-->>Supplier: 200 parquet bytes (seq 1)
    Note over OEM: 8. Transition task to CLOSED
    OEM->>Supplier: 8. POST /notifications TaskStatus (CLOSED)
    Supplier-->>OEM: 204
```

### 1. OEM opens the task

```http
POST https://tier1.example.com/stc/tasks
```

```json
{
  "taskId": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
  "status": "OPEN",
  "createdAt": "2026-10-07T09:00:00Z",
  "title": "Early warning: braking component",
  "description": "Elevated DTC rate in model A vehicles; suspected component lot issue."
}
```

→ `201` with the Task object (`status` `OPEN`). Both sides now hold the binding. The OEM is the Task Owner.
A retry of this POST with the same `taskId` is acknowledged idempotently. A counterparty that will not
participate responds `201` with `status` `DECLINED` and the reasons in the `errors` array.

### 2. OEM publishes claim data (file seq 0)

The OEM exports three complaints as a parquet file. Every record is `ADDED` at revision 0:

| recordId | key | revision | marker | refs | productId | occurredAt | failureCode |
| -------- | --- | -------- | ------ | ---- | --------- | ---------- | ----------- |
| 3f1d9a52-… | claim-a | 0 | ADDED |      | VIN-8842…​ | 2026-09-28 | C1234 |
| b7e3d2c1-… | claim-b | 0 | ADDED |      | VIN-1097…​ | 2026-10-01 | C1234 |
| c9a8b7d6-… | claim-c | 0 | ADDED |      | VIN-5513…​ | 2026-10-03 | C1290 |

It serves the file (cosmetic name `430f56d3-…_oem-example-com_complaint_000000_2026-10-07.parquet`),
appends it to its index, and announces:

```http
POST https://tier1.example.com/stc/notifications
Content-Type: application/cloudevents+json
```

```json
{
  "specversion": "1.0",
  "type": "org.dsaf.stc.FilesAppended.v1",
  "source": "did:web:oem.example.com",
  "id": "7d1c2b3a-4e5f-4a6b-8c7d-9e0f1a2b3c4d",
  "subject": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
  "time": "2026-10-07T09:12:04Z",
  "datacontenttype": "application/json",
  "data": {
    "taskId": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
    "streams": [
      {
        "type": "https://w3id.org/dsaf/quality/v1/complaint",
        "files": [
          { "seq": 0, "fileId": "8841a0b1-c2d3-4e5f-8a6b-7c8d9e0f1a2b",
            "createdAt": "2026-10-07T09:12:04Z" }
        ]
      }
    ]
  }
}
```

→ `204`.

### 3. OEM sends the early warning (profile event)

The warning is an informational event referencing the suspect records by pointer. It opens nothing:

```json
{
  "specversion": "1.0",
  "type": "org.dsaf.quality.EarlyWarning.v1",
  "source": "did:web:oem.example.com",
  "id": "2f3e4d5c-6b7a-4891-9c0d-1e2f3a4b5c6d",
  "subject": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
  "time": "2026-10-07T09:15:00Z",
  "datacontenttype": "application/json",
  "data": {
    "taskId": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
    "severity": "major",
    "information": "DTC C1234 occurring at 4x baseline in the referenced claims.",
    "records": [
      { "author": "did:web:oem.example.com", "type": "https://w3id.org/dsaf/quality/v1/complaint",
        "record": "3f1d9a52-7c84-4b6e-9d12-aa01bc23de45", "revision": 0, "seq": 0 },
      { "author": "did:web:oem.example.com", "type": "https://w3id.org/dsaf/quality/v1/complaint",
        "record": "b7e3d2c1-4a5f-4b6c-8d9e-0f1a2b3c4d5e", "revision": 0, "seq": 0 }
    ]
  }
}
```

### 4. Supplier syncs and pulls

```http
GET https://oem.example.com/stc/tasks/430f56d3-…/index
```

→ `200`: the complaint stream with file seq 0. The supplier pulls the content:

```http
GET https://oem.example.com/stc/tasks/430f56d3-…/files/8841a0b1-c2d3-4e5f-8a6b-7c8d9e0f1a2b
```

→ `200` (the parquet bytes).

### 5. Supplier contributes analysis and photos

The supplier analyzes the returned part from claim-b and publishes two files, an analysis record and a
photo documenting it:

*Analyses stream, file seq 0:*

| recordId | key | revision | marker | refs | result | finding |
|---|---|---|---|---|---|---|
| a77f0e1d-… | analysis-77 | 0 | ADDED | `[{"author":"did:web:oem.example.com","type":"…/complaint","record":"b7e3d2c1-…","revision":0,"seq":0}]` | defect-confirmed | corrosion at connector pin 4 |

*Attachment stream, file seq 0*, a zip with one record (envelope sidecar and payload):

```
9af0e1d2-c3b4-4a5f-8b6c-7d8e9f0a1b2c_0.json
9af0e1d2-c3b4-4a5f-8b6c-7d8e9f0a1b2c_0.jpg
```

`9af0e1d2-c3b4-4a5f-8b6c-7d8e9f0a1b2c_0.json` (the record's `key` contains the readable name):

```json
{
  "recordId": "9af0e1d2-c3b4-4a5f-8b6c-7d8e9f0a1b2c",
  "key": "photo-9a",
  "revision": 0,
  "marker": "ADDED",
  "mediaType": "image/jpeg",
  "payloadEntry": "9af0e1d2-c3b4-4a5f-8b6c-7d8e9f0a1b2c_0.jpg",
  "refs": [
    { "author": "did:web:tier1.example.com", "type": "https://w3id.org/dsaf/quality/v1/part-analysis",
      "record": "a77f0e1d-2c3b-4a59-8e67-45d2c1b0a9f8", "revision": 0, "seq": 0 }
  ]
}
```

One `FilesAppended` event announces both streams (data = the two new index entries), POSTed to the OEM's
`/notifications`.

### 6. Week 2: the OEM's data changes (file seq 1)

Claim-b was corrected, claim-c turned out to be a misdiagnosis. The next file contains only the changes.
Claim-a does not appear, because absence means unchanged:

| recordId | key | revision | marker | refs | productId | occurredAt | failureCode |
|---|---|---|---|---|---|---|---|
| b7e3d2c1-… | claim-b | 1 | UPDATED | | VIN-1097…​ | 2026-10-01 | C1236 |
| c9a8b7d6-… | claim-c | 1 | DELETED | | | | |

After applying seq 1, the set reads claim-a@0, claim-b@1, and claim-c deleted (its id is never reused). A
`FilesAppended` announces seq 1.

### 7. Recovery: the supplier missed the event

The announcement for seq 1 never arrived. Nothing is lost. On its next sync the supplier pulls the OEM's
index, sees the complaint stream head at seq 1 while it holds only seq 0, and fetches the gap. The same
mechanism covers any number of missed events. Events accelerate delivery. The index is the source of truth.

### 8. OEM closes the task

The countermeasure is confirmed effective. The owner transitions the task and pushes:

```json
{
  "specversion": "1.0",
  "type": "org.dsaf.stc.TaskStatus.v1",
  "source": "did:web:oem.example.com",
  "id": "61f2ac9e-3d44-4b1a-8a0d-7cf2b6a3e951",
  "subject": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
  "time": "2027-02-01T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "taskId": "430f56d3-7f41-4a7e-9f05-2d6c8b9e4a17",
    "status": "CLOSED",
    "closure": {
        "closedAt": "2027-02-01T10:00:00Z",
        "outcome": "https://w3id.org/dsaf/quality/v1/outcome/resolved",
        "reason": "Root cause confirmed; countermeasure effective from batch 2271.",
        "refs": [
          { "author": "did:web:tier1.example.com", "type": "https://w3id.org/dsaf/quality/v1/part-analysis",
            "record": "a77f0e1d-2c3b-4a59-8e67-45d2c1b0a9f8", "revision": 0 }
      ]
    }
  }
}
```

Streams freeze. The retention clock starts at `closure.closedAt`. A participant that misses this event
discovers `CLOSED` on its next `GET /tasks/430f56d3-…` against the owner.

---

## Use Case 2: Warranty Claim (request and answer)

The OEM demands recompense for the repair behind the field complaint of Use Case 1. Warranty runs under its own
task, the OEM's current warranty task under the Warranty Agreement, because the Warranty Claim Profile keeps claims
in generation tasks owned by the claimant. Pointers never cross tasks, so the claim and its evidence are contributed
under the warranty task. A demand needs receipt, observable state, and explicit refusal, so it is a Task Request,
not a contribution.

```mermaid
sequenceDiagram
    participant OEM
    participant Supplier
    Note over OEM: 1. invoice-7 and claim WC-2026-0815@0 are already published
    OEM->>Supplier: 2. POST /tasks/{taskId}/requests (settlement-claim)
    Supplier-->>OEM: 201 Request (RECEIVED)
    OEM->>Supplier: 3. GET /tasks/{taskId}/requests/{requestId}
    Supplier-->>OEM: 200 Request (RECEIVED)
    Note over Supplier: 4. Publish verification-2041@0 (verification seq 0)
    Supplier->>OEM: 4. POST /notifications FilesAppended (verification seq 0)
    OEM-->>Supplier: 204
    Note over Supplier: 5. Transition RECEIVED to ANSWERED and bind the response pointer
    Supplier->>OEM: 5. POST /notifications RequestStatus (ANSWERED)
    OEM-->>Supplier: 204
    OEM->>Supplier: 6. GET /tasks/{taskId}/index
    Supplier-->>OEM: 200 verification stream, seq 0
    OEM->>Supplier: 6. GET /tasks/{taskId}/files/{fileId}
    Supplier-->>OEM: 200 parquet bytes (verification-2041@0)
```

### 1. Supporting evidence is already published

The OEM's claim stream contains the claim `WC-2026-0815` @ revision 0 (file seq 0), with the repair date, the
customer part number, and the claimed amount. The repair invoice went into the OEM's attachment stream (file seq 0,
record `invoice-7` @ revision 0). Both were announced like any contribution.

### 2. OEM submits the claim

```http
POST https://tier1.example.com/stc/tasks/5b2e9c71-…/requests
```

```json
{
  "messageId": "b1d5f9e2-3c4b-4f7a-9c3e-2f1e5d6c7b8a",
  "kind": "https://w3id.org/dsaf/warranty/v1/settlement-claim",
  "payload": {
    "claimedAmount": { "value": 418.50, "currency": "EUR" },
    "repairReference": "WC-2026-0815"
  },
  "refs": [
    { "author": "did:web:oem.example.com", "type": "https://w3id.org/dsaf/warranty/v1/claim",
      "record": "6d2f8a14-3b5c-4e7d-9a1f-2c4e6b8d0f13", "revision": 0, "seq": 0 },
    { "author": "did:web:oem.example.com", "type": "https://w3id.org/dsaf/quality/v1/attachment",
      "record": "f7e6d5c4-b3a2-4918-8776-655443322110", "revision": 0, "seq": 0 }
  ],
  "respondBy": "2026-12-01T00:00:00Z"
}
```

→ `201`, `Location: /tasks/5b2e9c71-…/requests/8c1f2d3e-4a5b-4c6d-8e7f-9a0b1c2d3e4f`:

```json
{
  "requestId": "8c1f2d3e-4a5b-4c6d-8e7f-9a0b1c2d3e4f",
  "taskId": "5b2e9c71-8d34-4f2a-9b16-3e7c0a5d1f48",
  "kind": "https://w3id.org/dsaf/warranty/v1/settlement-claim",
  "status": "RECEIVED",
  "changeSeq": 0,
  "payload": { "claimedAmount": { "value": 418.50, "currency": "EUR" }, "repairReference": "WC-2026-0815" },
  "refs": [ "… as submitted …" ],
  "respondBy": "2026-12-01T00:00:00Z"
}
```

The `201` is the receipt. After a network timeout, the OEM retries with the **same** `messageId` and gets
this same request back. The pinned `refs` mean the demand stays bound to the claim at revision 0 even if the claim
record evolves later.

### 3. The wait is observable

```http
GET https://tier1.example.com/stc/tasks/5b2e9c71-…/requests/8c1f2d3e-…
```

→ `200`, `status: RECEIVED`. After `respondBy`, "overdue" is the OEM's local judgment.

### 4. Supplier publishes the answer record before transitioning

The verification goes into the supplier's verification stream, file seq 0:

| recordId   | key               | revision | marker | refs                                                                                                     | acceptedQuota | grounds                                                                       |
| ---------- | ----------------- | -------- | ------ | -------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------------------------------------------------------- |
| 2041fa1b-… | verification-2041 | 0        | ADDED  | `[{"author":"did:web:oem.example.com","type":"…/claim","record":"6d2f8a14-…","revision":0,"seq":0}]` | 0.60          | corrosion consistent with supplied-lot defect; labor rate partially contested |

`FilesAppended` announces it. The spec requires publication no later than the transition, so the pointer bound
next can never dangle.

### 5. Supplier answers the request

The supplier transitions `RECEIVED → ANSWERED`, binding the response pointer, and pushes:

```json
{
  "specversion": "1.0",
  "type": "org.dsaf.stc.RequestStatus.v1",
  "source": "did:web:tier1.example.com",
  "id": "4d5e6f7a-8b9c-4d0e-9f1a-2b3c4d5e6f7a",
  "subject": "8c1f2d3e-4a5b-4c6d-8e7f-9a0b1c2d3e4f",
  "time": "2026-11-20T15:12:00Z",
  "datacontenttype": "application/json",
  "data": {
    "taskId": "5b2e9c71-8d34-4f2a-9b16-3e7c0a5d1f48",
    "requestId": "8c1f2d3e-4a5b-4c6d-8e7f-9a0b1c2d3e4f",
    "status": "ANSWERED",
    "response": {
      "record": { "author": "did:web:tier1.example.com",
                  "type": "https://w3id.org/dsaf/warranty/v1/verification",
                  "record": "2041fa1b-2c3d-4e5f-8a6b-7c8d9e0f1a2b", "revision": 0, "seq": 0 }
    }
  }
}
```

A missed push changes nothing. The next `GET` on the request reports the same `ANSWERED` state and pointer.

### 6. OEM collects the answer

The OEM resolves the pointer. It pulls the supplier's index, locates file seq 0 of the verification stream,
downloads it, and extracts `verification-2041` @ 0. The OEM now has the 60% quota and the `grounds` statement, auditable
against exactly the record revisions both sides saw.

### 7. Variations

- **Immediate decline**: for a claim for a part outside the supplier's scope, the creation response itself is
  `201` with `status: DECLINED` and `errors: [{ "message": "Part 4711-B is not a supplied component" }]`.
  The request exists and reports `DECLINED` with its reasons on every later retrieval.
- **Dispute**: the OEM contests the quota with a **new** request (new `messageId`, kind `settlement-dispute`)
  whose `refs` pin `verification-2041@0`. The original request stays terminal and immutable. The supplier answers
  with a new revision of the same verification record (`verification-2041@1`) or, if it upholds its evaluation,
  with a pointer to the current revision. The current verification of the claim is always the record's current
  revision.

---

## Use Case 3: 8D Analysis (request with an evolving answer)

The OEM formally demands an 8D root cause analysis. The demand is one request. The answer is one record
that matures through revisions.

```mermaid
sequenceDiagram
    participant OEM
    participant Supplier
    OEM->>Supplier: 1. POST /tasks/{taskId}/requests (8d-request)
    Supplier-->>OEM: 201 Request (RECEIVED)
    Note over Supplier: 2. Publish 8d-77@0 (report-8d seq 0)
    Supplier->>OEM: 2. POST /notifications FilesAppended (report-8d seq 0)
    OEM-->>Supplier: 204
    Supplier->>OEM: 2. POST /notifications RequestStatus (ANSWERED, 8d-77@0)
    OEM-->>Supplier: 204
    Note over Supplier: 3. Publish 8d-77@1 (seq 1), then 8d-77@2 (seq 2)
    Supplier->>OEM: 3. POST /notifications FilesAppended (report-8d seq 1)
    OEM-->>Supplier: 204
    Supplier->>OEM: 3. POST /notifications FilesAppended (report-8d seq 2)
    OEM-->>Supplier: 204
    Note over OEM: 4. Close the task with closure.refs pinning 8d-77@2
    OEM->>Supplier: 4. POST /notifications TaskStatus (CLOSED)
    Supplier-->>OEM: 204
```

### 1. OEM submits the 8D demand

```http
POST https://tier1.example.com/stc/tasks/430f56d3-…/requests
```

```json
{
  "messageId": "c2e6a0f3-4d5b-4a8c-9d0e-3f2a1b4c5d6e",
  "kind": "https://w3id.org/dsaf/quality/v1/8d-request",
  "payload": { "problem": "Connector pin corrosion, DTC C1236", "customerPartNumber": "BRK-4711" },
  "refs": [
    { "author": "did:web:oem.example.com", "type": "https://w3id.org/dsaf/quality/v1/complaint",
      "record": "b7e3d2c1-4a5f-4b6c-8d9e-0f1a2b3c4d5e", "revision": 1, "seq": 1 },
    { "author": "did:web:tier1.example.com", "type": "https://w3id.org/dsaf/quality/v1/part-analysis",
      "record": "a77f0e1d-2c3b-4a59-8e67-45d2c1b0a9f8", "revision": 0, "seq": 0 }
  ]
}
```

→ `201`, request `9d2e3f4a-…`, `status: RECEIVED`. (Request `refs` may point at either party's records, here one
of each.)

### 2. Supplier answers with the initial report

The supplier publishes `8d-77` @ revision 0 (D1–D3: team, problem, containment) in its 8D stream, file
seq 0, announces it, then answers the request pinned to that revision:

```json
"response": {
  "record": { "author": "did:web:tier1.example.com", "type": "https://w3id.org/dsaf/quality/v1/report-8d",
              "record": "d8770a1b-2c3d-4e5f-9a6b-7c8d9e0f1a2b", "revision": 0, "seq": 0 }
}
```

The request is now terminal (`ANSWERED`). The obligation is discharged by the initial report. The report
itself continues to evolve.

### 3. The report matures as ordinary revisions

- File seq 1: `8d-77` @ 1, `UPDATED`, root cause identified (D4), with `refs` to the failure-pattern evidence.
- File seq 2: `8d-77` @ 2, `UPDATED`, corrective actions verified (D5–D8).

Each is a complete snapshot, each announced by `FilesAppended`. The request stays pinned to revision 0, the
auditable answer as given. The OEM follows the current report through the unpinned record id (newest-first
lookup, head file first). Supporting lab data and photos go into the attachment stream, and their records
reference `8d-77`.

### 4. Closure cites the final report

When the OEM closes the task, `closure.refs` pins `8d-77` @ revision 2. The closure references its concluding
artifact, and the deletion clock starts for the data contributed under the task.

---

## The One Recovery Rule

Every walkthrough step that pushed an event recovers the same way. If the event is lost, the fact it
announced is recoverable by pull:

| Lost event | Recovered by |
|---|---|
| `FilesAppended` | `GET /tasks/{taskId}/index` (the gapless `seq` exposes the missing files) |
| `RequestStatus` | `GET /tasks/{taskId}/requests/{requestId}`, or `GET /tasks/{taskId}/requests?fromChangeSeq=n` for many requests |
| `TaskStatus` | `GET /tasks/{taskId}` against the owner |
| profile events (early warning) | re-sent by their author, or moot, because they reference records the index already lists |

Events cost latency when lost, never consistency.
