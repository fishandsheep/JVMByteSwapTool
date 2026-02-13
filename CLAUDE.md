# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JVM ByteSwap Tool (Swapper) is a Java agent that hot-swaps class bytecode at runtime using JVM instrumentation, ASM, Javassist, and JVMTI. It attaches to running JVM processes and provides a Web UI for debugging Java applications (especially Spring Boot) through bytecode manipulation.

## Build and Run

### Build (Maven)
```bash
mvn package
# Output: target/swapper-0.0.1-SNAPSHOT.jar
# To rename: cp target/swapper-0.0.1-SNAPSHOT.jar swapper.jar
```

### Run
```bash
java -jar swapper.jar
# Select target JVM PID from the list
# Web UI will be served at http://localhost:8000 (default)
# WebSocket at http://localhost:18000 (default)
```

### Run with Custom Ports
```bash
java -jar -Dw_http_port=9999 -Dw_ws_port=19999 swapper.jar
```

### Run Tests
```bash
mvn test
```

## Architecture

### Entry Points

- **`Attach.java`** - Main class when running from CLI. Lists JVM processes and attaches the agent to target PID. Handles custom `WClassLoader` for JDK 1.8 compatibility.
- **`App.java`** - Agent entry point (`agentmain`). Initializes instrumentation, HTTP/WebSocket servers, and exec bundles.

### Core Components

- **`Global.java`** - Central state management: instrumentation instance, loaded classes cache, transformer registry, WebSocket connections, native library loading, logging broadcast.

- **`Swapper.java`** - Central dispatch hub. Routes message types to appropriate `BaseClassTransformer` implementations:
  - `WATCH` → `WatchTransformer`
  - `OUTER_WATCH` → `OuterWatchTransformer`
  - `CHANGE_BODY` → `ChangeBodyTransformer`
  - `CHANGE_RESULT` → `ChangeResultTransformer`
  - `REPLACE_CLASS` → `ReplaceClassTransformer`
  - `TRACE` → `TraceTransformer`
  - `DECOMPILE` → `DecompileTransformer`

- **`BaseClassTransformer`** (abstract) - Base class for all transformers. Implements `ClassFileTransformer`. Handles bytecode transformation using ASM. Subclasses implement `transform(byte[] origin)`.

### Bytecode Manipulation

- **`WCompiler`** - Compiles Java code at runtime using Janino. Also decompiles bytecode using CFR.
- **ASM** (`w.core.asm`) - Used for low-level bytecode manipulation in transformers.
- **`ExecBundle`** - Dynamic code execution via hot-swapping a temporary `w.Exec` class.

### Communication

- **`Httpd`** - NanoHTTPD server serving the Web UI (static files from `/nanohttpd` resources).
- **`Websocketd`** - WebSocket server receiving JSON messages from Web UI, dispatching to `Swapper`.
- **Messages** (`w.web.message.*`) - Jackson-serialized message types corresponding to transformer operations.

### Native Support

- Native libraries (`w_amd64.dll`, `w_amd64.so`, `w_aarch64.so`, etc.) loaded via JNI for `getInstances(Class<?>)` - primarily used to extract Spring application context.

### Utilities

- **`SpringUtils`** - Spring Boot integration, detects and caches `LaunchedURLClassLoader`.
- **`WClassLoader`** / **`JarInJarClassLoader`** - Custom classloaders for handling agent JAR structure.
- **`GroovyBundle`** - Groovy script evaluation support.

## Transformer Lifecycle

1. WebSocket receives message → `Websocketd.dispatch()`
2. `Swapper.swap()` creates appropriate transformer
3. `Global.addTransformer()` registers with instrumentation
4. `Global.addActiveTransformer()` calls `instrumentation.retransformClasses()`
5. JVM calls `BaseClassTransformer.transform()` when class is loaded/retransformed
6. Transformer modifies bytecode using ASM/Janino
7. Logs broadcast to all WebSocket clients

## Key Configuration

- `w_http_port` - HTTP server port (default: 8000)
- `w_ws_port` - WebSocket port (default: 18000)
- `maxHit` - Max hit count for watch/trace before auto-delete (default: 100)
- `-Xverify:none` - Recommended JVM flag for class retransformation

## Dependencies

- ASM 9.7 (bytecode manipulation)
- Janino 3.1.12 (runtime compilation)
- CFR 0.152 (decompilation)
- NanoHTTPD 2.2.0 (embedded HTTP server)
- Groovy 4.0.22 (scripting)
- OGNL 3.2.1 (expression language for Spring access)
- Jackson 2.13.5 (JSON serialization)
- Lombok 1.18.28

## Go Client (jbs-client)

CLI client in `./jbs-client/` connects to Swapper's WebSocket. Build with:
```bash
cd jbs-client && go build
```
