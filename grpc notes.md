# gRPC — Complete Notes

---

## 1. What is RPC?

**RPC (Remote Procedure Call)** is the idea of calling a function that lives on a **different machine** as if it were a **local function call** in your own code.

Without RPC you'd write:
```java
// Manual HTTP call — you handle everything
HttpClient client = HttpClient.newHttpClient();
HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("http://patient-service/api/patients/123"))
    .build();
HttpResponse<String> res = client.send(req, BodyHandlers.ofString());
// then manually parse JSON...
```

With RPC (gRPC) you write:
```java
// Feels like a local method call — framework handles everything
PatientResponse response = patientStub.checkPatientExists(request);
```

That single line **crosses the network**, reaches the patient-service process, executes code there, and returns the result — but from your perspective it looks identical to calling any local Java method.

---

## 2. What is gRPC?

**gRPC** = Google Remote Procedure Call. It is Google's open-source implementation of RPC built on two technologies:

| Layer | Technology | Purpose |
|---|---|---|
| Transport | **HTTP/2** | Fast, multiplexed, binary network transport |
| Serialization | **Protocol Buffers (protobuf)** | Compact binary data format — much smaller than JSON |

You define your service contract in a `.proto` file. gRPC generates all the boilerplate Java code (stubs, message classes) for both server and client from that one file.

---

## 3. The Four Types of gRPC

### 3a. Unary RPC ← **this is what we used**
One request → one response. Exactly like a normal function call.
```protobuf
rpc CheckPatientExists (PatientRequest) returns (PatientResponse);
```
```
Client ──── request ────► Server
Client ◄─── response ─── Server
```

### 3b. Server Streaming RPC
Client sends one request, server sends back a **stream** of multiple responses.
```protobuf
rpc GetAppointmentHistory (PatientRequest) returns (stream AppointmentResponse);
```
```
Client ──── 1 request ──────────────────► Server
Client ◄─── response 1 ─────────────── Server
Client ◄─── response 2 ─────────────── Server
Client ◄─── response 3 ─────────────── Server
```
*Use case: live feed, paginated results pushed in chunks.*

### 3c. Client Streaming RPC
Client sends a **stream** of messages, server replies once at the end.
```protobuf
rpc UploadRecords (stream RecordRequest) returns (UploadSummary);
```
*Use case: uploading a large batch of records.*

### 3d. Bidirectional Streaming RPC
Both sides stream simultaneously, independently.
```protobuf
rpc LiveChat (stream ChatMessage) returns (stream ChatMessage);
```
*Use case: real-time chat, live telemetry dashboards.*

---

## 4. How We Implemented It — Step by Step

### Step 1 — Write the `.proto` file (contract first)

This is the **single source of truth**. Both server and client share the same contract.

**patient-service** (`src/main/proto/patient.proto`):
```protobuf
syntax = "proto3";
package patient;

option java_package = "com.healthcare.patient_service.grpc";
option java_outer_classname = "PatientProto";
option java_multiple_files = true;

service PatientGrpcService {
    rpc CheckPatientExists (PatientRequest) returns (PatientResponse);
}

message PatientRequest {
    string patient_id = 1;   // the = 1 is the field NUMBER, not a value
}

message PatientResponse {
    bool exists = 1;
    string patient_id = 2;
    string name = 3;
}
```

**appointment-service** gets a **copy** of both patient.proto and doctor.proto in its own `src/main/proto/` — so Maven can generate the `BlockingStub` classes it needs to make calls.

> **Field numbers** (the `= 1`, `= 2`) are how protobuf identifies fields in the binary wire format. They never change once deployed — if you add a new field you give it a new number (e.g. `= 4`). This is what makes protobuf **backward compatible**.

---

### Step 2 — Add Maven dependencies

Three services need **protobuf + grpc-stub + grpc-protobuf** in all of them. The difference is the starter:

```xml
<!-- patient-service and doctor-service (SERVERS) -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>

<!-- appointment-service (CLIENT) -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>

<!-- ALL THREE services need these -->
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.25.3</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.65.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.65.0</version>
</dependency>
<!-- Required because gRPC generated code uses @javax.annotation.Generated
     which was removed from JDK 9+ -->
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```

Also add the **build plugins** so Maven knows how to compile `.proto` files:
```xml
<extensions>
    <extension>
        <groupId>kr.motd.maven</groupId>
        <artifactId>os-maven-plugin</artifactId>   <!-- detects OS/arch for protoc binary -->
        <version>1.7.1</version>
    </extension>
</extensions>

<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.6.1</version>
    <!-- configured to download protoc and grpc-java plugin, then generate Java -->
</plugin>
```

---

### Step 3 — Run `mvn compile` → Java files are generated

```
mvn compile
```

Maven runs `protoc` (the protobuf compiler) on your `.proto` file and generates Java classes into `target/generated-sources/protobuf/`:

```
target/generated-sources/protobuf/
├── java/
│   ├── PatientRequest.java          ← message class
│   ├── PatientResponse.java         ← message class
│   └── PatientProto.java            ← descriptor
└── grpc-java/
    └── PatientGrpcServiceGrpc.java  ← contains ImplBase + BlockingStub
```

**You never write or edit these files.** They are regenerated every compile.

`PatientGrpcServiceGrpc.java` contains two important inner classes:
- `PatientGrpcServiceImplBase` — abstract class you **extend** on the server
- `PatientGrpcServiceBlockingStub` — class you **inject** on the client

---

### Step 4 — Write the server implementation

Override the method declared in the proto file:

```java
@GrpcService          // registers this with the embedded Netty gRPC server
@AllArgsConstructor
public class PatientGrpcServiceImpl
        extends PatientGrpcServiceGrpc.PatientGrpcServiceImplBase {

    private final PatientRepository patientRepository;

    @Override
    public void checkPatientExists(PatientRequest request,
                                   StreamObserver<PatientResponse> responseObserver) {
        try {
            UUID id = UUID.fromString(request.getPatientId());

            PatientResponse response = patientRepository.findById(id)
                .map(p -> PatientResponse.newBuilder()
                    .setExists(true)
                    .setPatientId(p.getId().toString())
                    .setName(p.getFirstName() + " " + p.getLastName())
                    .build())
                .orElseGet(() -> PatientResponse.newBuilder()
                    .setExists(false)
                    .setPatientId(request.getPatientId())
                    .setName("")
                    .build());

            responseObserver.onNext(response);      // send the result
            responseObserver.onCompleted();         // close the stream
        } catch (IllegalArgumentException e) {
            responseObserver.onError(
                io.grpc.Status.INVALID_ARGUMENT
                    .withDescription("Invalid UUID: " + request.getPatientId())
                    .asRuntimeException()
            );
        }
    }
}
```

> **`StreamObserver`** is gRPC's callback object. For unary RPC: call `onNext()` once with your result, then `onCompleted()`. For errors: call `onError()`. It exists (instead of a simple `return`) because the same API works for streaming RPCs where you'd call `onNext()` multiple times.

> **Protobuf objects are immutable** — you can never do `response.setExists(true)`. You always use the builder pattern: `PatientResponse.newBuilder().setExists(true).build()`.

---

### Step 5 — Configure application.yaml

**Server side** (patient-service, doctor-service):
```yaml
grpc:
  server:
    port: 9091   # separate port from HTTP (8481) — gRPC runs on its own Netty server
```

**Client side** (appointment-service):
```yaml
grpc:
  client:
    patient-service:               # must match @GrpcClient("patient-service")
      address: static://localhost:9091
      negotiation-type: plaintext  # no TLS (fine for internal service-to-service)
    doctor-service:
      address: static://localhost:9092
      negotiation-type: plaintext
```

---

### Step 6 — Implement the client

```java
@Service
@RequiredArgsConstructor   // NOT @AllArgsConstructor — stubs use field injection
public class AppointmentService {

    private final AppointmentRepository repository;  // constructor injected

    @GrpcClient("patient-service")  // name matches application.yaml key
    private PatientGrpcServiceGrpc.PatientGrpcServiceBlockingStub patientStub;

    @GrpcClient("doctor-service")
    private DoctorGrpcServiceGrpc.DoctorGrpcServiceBlockingStub doctorStub;

    public AppointmentResponse createAppointment(CreateAppointmentRequest request) {

        // gRPC call — looks like a local method call
        PatientResponse pr = patientStub.checkPatientExists(
            PatientRequest.newBuilder()
                .setPatientId(request.patientId().toString())
                .build());

        if (!pr.getExists()) {
            throw new IllegalArgumentException("Patient not found");
        }

        // ... same for doctor, then save appointment
    }
}
```

> **Why `@RequiredArgsConstructor` not `@AllArgsConstructor`?**
> `@AllArgsConstructor` generates a constructor for every field including the stubs. But gRPC stubs are injected **after** construction via field injection (like `@Autowired`). If you put them in the constructor, Spring tries to inject them at construction time and fails. `@RequiredArgsConstructor` only generates a constructor for `final` fields (your repository), leaving the stubs for field injection.

---

## 5. Why gRPC? How is it faster than REST?

### The two reasons gRPC is fast

**Reason 1 — Protocol Buffers (binary serialization)**

REST uses JSON:
```json
{"patient_id": "a1b2c3d4-...", "exists": true, "name": "John Smith"}
```
This is human-readable text. The sender must **serialize** it to text, send it, and the receiver must **parse** it back into objects. Text is verbose — field names are repeated in every message.

Protobuf encodes the same data as binary:
```
[field 1][length][a1b2c3d4...][field 2][1][field 3][length][John Smith]
```
- Field names are **not sent** — only the field number (1, 2, 3)
- Numbers are encoded in a variable-length format (small numbers = fewer bytes)
- No quotes, no braces, no colons
- Result: **3–10x smaller payload** than JSON, and **5–10x faster to serialize/deserialize**

**Reason 2 — HTTP/2 transport**

REST typically uses HTTP/1.1 which has a fundamental limitation: **one request at a time per connection**. If you make 5 calls you either wait for each to finish or open 5 separate TCP connections (expensive).

HTTP/2 has **multiplexing** — multiple requests and responses flow simultaneously over a **single TCP connection** with no head-of-line blocking. It also uses header compression (HPACK), so repeated headers (like `Content-Type`, `Authorization`) are sent only once and referenced by index after that.

| Feature | HTTP/1.1 (REST) | HTTP/2 (gRPC) |
|---|---|---|
| Requests per connection | 1 at a time | Many simultaneous |
| Headers | Repeated every request (text) | Compressed, sent once |
| Data format | Text (JSON/XML) | Binary (protobuf) |
| Streaming | No | Yes (bidirectional) |

---

## 6. gRPC vs REST — Honest Comparison

### Where gRPC wins

| Aspect | gRPC | REST |
|---|---|---|
| **Speed** | 5–10x faster (binary + HTTP/2) | Slower (JSON + HTTP/1.1) |
| **Type safety** | Strongly typed — contract enforced at compile time | No enforcement — field name typo compiles fine |
| **Code generation** | Stubs auto-generated from `.proto` | You write DTOs and clients manually |
| **Streaming** | Built-in (server/client/bidirectional) | Requires SSE or WebSocket workarounds |
| **Inter-service calls** | Ideal — tight contract between known services | More overhead for internal calls |

### Where REST wins

| Aspect | REST | gRPC |
|---|---|---|
| **Browser support** | Native | Not supported directly (needs grpc-web proxy) |
| **Human readability** | JSON is easy to read/debug in browser | Binary — need tooling to inspect |
| **Ecosystem/tooling** | Massive — Postman, Swagger, curl | Smaller — need BloomRPC/grpcurl |
| **Simplicity** | Easy to get started | More setup (proto files, codegen, plugins) |
| **Flexibility** | Any client can call it | Both sides must have the proto contract |
| **Caching** | HTTP caching built-in (GET + CDN) | No standard HTTP caching |

### The rule of thumb
- **gRPC** → internal microservice-to-microservice communication (what we did)
- **REST** → public-facing APIs consumed by browsers, mobile apps, third parties

That's exactly our architecture: appointment-service exposes REST to the outside world, but talks to patient-service and doctor-service internally via gRPC.

---

## 7. How Does gRPC Feel Like a Local Function Call?

This is the key insight. When you write:

```java
PatientResponse response = patientStub.checkPatientExists(request);
```

What actually happens behind the scenes:

```
Your code calls patientStub.checkPatientExists(request)
        │
        ▼
[gRPC framework serializes `request` to protobuf binary bytes]
        │
        ▼
[HTTP/2 frame sent over TCP to localhost:9091]
        │
        ▼  (network)
        │
        ▼
[patient-service receives the HTTP/2 frame]
        │
        ▼
[gRPC framework deserializes bytes back into PatientRequest object]
        │
        ▼
[your checkPatientExists() method runs, queries the DB]
        │
        ▼
[result serialized to protobuf binary bytes]
        │
        ▼
[HTTP/2 response frame sent back]
        │
        ▼  (network)
        │
        ▼
[gRPC framework deserializes bytes into PatientResponse object]
        │
        ▼
Your code receives `response` ← looks like it just returned
```

All of that networking, serialization, deserialization, and error handling is **hidden inside the generated stub**. From your perspective you called a method and got a return value — just like calling `repository.findById()`. This abstraction is the entire point of RPC.

The `BlockingStub` name tells you it **blocks** your thread until the response arrives (synchronous). There's also `AsyncStub` (non-blocking with callbacks) and `FutureStub` (returns a `Future`) for async use cases.
