# Overview

## Introduction and Basic Flow

Anki-Android-Backend uses Rust, Java and a little Python. Since setting up a Rust environment is complex, having a separate library encourages drive-by contributions to the main app by keeping a low barrier to entry for Anki-Android.

There are two aspects of this library:

* `rsdroid.so` - A rust library which contains `anki/rslib` and a JNI bridge (`rslib-bridge`)
* rsdroid.aar - A java library with `rsdroid.so` and handles command processing plus acts as an adapter for database access

### Protocol Buffers

Serialisation over the JNI boundary happens mostly via **Protocol Buffers**.

The source files are stored in `anki/proto`. We perform RPC over JNI, rather than depending on HTTP.

`fluent.proto` is a special-case, and is generated when the Rust library is built (`anki/rslib/build.rs`)

Protobuf service generation is handled on the rust side via `anki/rslib/build.rs` and on the java side via `tools/protoc-gen/protoc-gen.py`.

### Main usages

See the components section for additional details

### Command Setup

![](https://www.planttext.com/api/plantuml/img/ZLLDZzem4BtxLupsi0Af8FNAeThjqWE7TYMazj9MaUDCmKkmhV44YNzVEmci7Q8bkIJnl3VpvYjvyYo9csCjf69Bq5tPiuV68maNS5ff9mt3jl6y9gkhhr8Tq5GHD3mJfMrC9UaC3y_ce1VFfehMHMz-o1psXpzPrtxCyElpvkZgIvXBX1GOhX-IzGc_8-zjvPTlyYHx_PaXqKM-rkMY95tjCDSJpfVa8xV5T924AKD6EQF5HK8q5MMlS4UsSP3cBuI8vOJ5bzighi0wD2-shb6njcOjMPRIuyn9tiz1t116dAn04Kf6M9UmCQ6xHY4ymWxvwcq-wYXjGQyawcvXxv8wAI833yX1WJKd98Q81RRWoBAzuIIT_23UQrQHZPaB4RlQ5VQrPAaDEAiDXvhEh45WVSHvYqa3vF5MWCOtXFthR1IVJKqdCCTdWCX8PcKCdvX7_1DoGzTSnWDaAJWahprdZ3XpQNs21dYltiGiqsupVOBopFZxxzIS1-46IMVm2fMj44AGoT1sakw1eun2NNKKSIMxvPl4j5HqErGOMpIk2aztYI47KD90YV5hMSMfbqgXzs4PwWdTxseyeUi9yAFvVjcZEi2_y1L78ajtyLlqpXgFeFrbcPVVZexFopzcsqtcwLB0E6JnBWOEsz-4U0fluN_o7m00)

Source: `docs/sources/sequence_open_collection.plantuml`. Created with [planttext](https://www.planttext.com/)

### Process command

![](https://www.planttext.com/api/plantuml/img/bPH1ZzCm48Nl_XMZFQNIjg9mAnBQ1VQ0n7802Gvq5JcnTsis6KVZSKibVZpZ90Kd1R7aK4MUzxqPFpjLLu4rSMmRfMls1CCpUGyGWoNLYSxLhjF8y346ValUcTUwVhHeacY-fYeVqMWwmiKrFhhbDPfKNOxbYudXkFXv_QxjcfFRoIWNolD1izlRMyixRyBgczxhSSn98MjFeN7LiY9d7koqhQolA2IsrmoIZDGo-9JeTGb8fR8Q9tmW7pl8jwbK2WsMhywpsa2m_DxNkhbr6Dc6BpPmiNws0ANEnAF1Fza1_JErWThZtXA3ansmXuuy-pamIMy3zhkjfS4RtxOQJT4nNSBwnILKHxPVxnOlrKIV3B88KyU_SPaiKNcE6w28vOYMYGX5ngfS-mJsUUaGBVs7nQU3ute77cNaBHfRUsCbj2xo5YNpHj9lxbTDohziXmCe3t82MoJBaH1yP16d-z5dl4NvZ2oH_9wMpYQOn3RQ53TjnySVwKBT97enJssMzN2wlNywttxtJqCqkleN0eMx1zwHF-1Pn_dD_EtHpveyzbALGCsu2wNKbGZbl-Kd)

Source: `docs/sources/sequence_method_call.plantuml`. Created with [planttext](https://www.planttext.com/)

## Components

### Anki-Android

There is a small amount of code in the consuming app Anki-Android to use this library

#### DroidBackend/RustDroidBackend

Java/Rust interface to the backend. `RustDroidBackend` wraps the implementation of the Rust.

* Allows a testable comparison between the Java and the Rust during the conversion
  * Allows a pure Java conversion afterwards (if deemed appropriate due to AGPL concerns)
* Encapsulates all access to the Rust, allowing the implementation to later be swapped out within one file.

### Anki-Android-Backend

#### RustBackend (interface)

An interface of the commands allowed to be sent to the Rust. Allows for easy mocking, and method discovery.

##### RustBackend Example

RustBackend is generated by `gen/protoc-gen/protoc-gen.py` and is not checked into source control.

```java
    Backend.RenderCardOut renderUncommittedCard(@Nullable Backend.Note note, int cardOrd, @Nullable com.google.protobuf.ByteString template, boolean fillEmpty);
```

#### RustBackendImpl

It contains a method per RPC method defined in `backend.proto`

It is responsible for:

* Converting parameters from Java types to protobufs
* Executing a command
* Ensuring that the result was not an error
* Deserializing and returning data (if applicable)

##### RustBackendImpl Example

RustBackendImpl is generated by `gen/protoc-gen/protoc-gen.py` and is not checked into source control.

```java
    public Backend.SearchCardsOut searchCards(@Nullable java.lang.String search, @Nullable Backend.SortOrder order) { 
        byte[] result = null;
        try {
            Backend.SearchCardsIn.Builder builder = Backend.SearchCardsIn.newBuilder();
            if (search != null) { builder.setSearch(search); }
            if (order != null) { builder.setOrder(order); }
            Backend.SearchCardsIn protobuf = builder.build();

            Pointer backendPointer = ensureBackend();
            result = NativeMethods.executeCommand(backendPointer.toJni(), 9, protobuf.toByteArray());
            Backend.SearchCardsOut message = Backend.SearchCardsOut.parseFrom(result);
            validateMessage(result, message);
            return message;
        } catch (InvalidProtocolBufferException ex) {
            validateResult(result);
            throw BackendException.fromException(ex);
        }
    }
```

#### BackendV1Impl

BackendV1Impl extends `RustBackendImpl`.

It is responsible for:

* Maintaining a pointer to the collection (to be passed into the Rust to identify the collection)
* Accessing methods in the Rust which are not generated from Protobufs
  * JSON serialization for database inputs
  * Collection opening/closing

#### NativeMethods

Method definitions to access `rslib-bridge`

### rslib-bridge

#### lib.rs

All public methods callable from the Java are available here.

Responsible for:

* Handling JNI specific concerns (conversions from Java to Rust primitives)
* Calling methods in `anki/rslib`
* Defining the interface callable by `NativeMethods`
* Converting a provided `backendPointer` into a Collection object
* Serialization of outputs and deserialization of inputs
* Handling panics and converting them to errors

## Database Access

We need to use the Rust for database access as:

* We need an open collection to perform most commands in rslib
* An open collection obtains a lock on the database - access can only be made through the Rust.

So, we implement `SupportSQLiteOpenHelper.Factory` and related classes.

### Memory Pressure

Anki's rust code does not stream database results, all results are currently obtained without streaming and are temporarily stored in memory in the Rust.

This is not a significant problem, as:

* Rust is not confined by the Java heap limit
* Most results are small
* In time, we will move most data processing to the Rust, removing the need to deserialize data
* Java has been converted to use protobuf serialization (vs Anki Desktop using JSON), this significantly reduces memory usage.

#### LimitOffsetSQLiteCursor

If the above is not sufficient, work could be performed to make `LimitOffsetSQLiteCursor` work.

#### Streaming over JNI

Over the JNI boundary, streaming takes place via `StreamingProtobufSQLiteCursor`

**Caveat**: Nested streamed queries cannot currently take place using this method due to the Rust implementation via a HashMap (ensuring that memory will not be leaked by non-disposed result sets). An exception will occur if nested queries are detected, and this constraint will be revisited.

## Panics

Rust panics (excluding OutOfMemory) are serialized to an `BackendError` proto, and sent to the Java.

`BackendException.fromError` will convert this to a `BackendFatalError`. ACRA cannot pick up on a native crash, by converting it into an Error, we can get a stack trace.

## Publishing

Artifacts are published to Sonatype OSS via CI. This is a Maven repository. `VERSION_NAME` inside `gradle.properties` defines the version number
