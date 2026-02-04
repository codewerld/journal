# Next.js ConnectRPC Proxy Pattern

This repository demonstrates a strictly typed gRPC implementation in Next.js using **ConnectRPC** and **TanStack Query**.

The core architectural goal is to unify the Server and Client definitions. It achieves this by implementing a **Proxy Pattern**: the Next.js API route exposes a local `GreetService` that acts as a typed gateway, forwarding requests to a remote `ElizaService` while reusing the exact same Proto definitions.

## 1. Dependencies

Required packages for ConnectRPC v2, Next.js 15, and TanStack Query.

**`package.json`**

```json
{
  "name": "airborneo-grpc-proxy",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "generate": "buf generate proto"
  },
  "dependencies": {
    "@bufbuild/protobuf": "^2.2.3",
    "@connectrpc/connect": "^2.0.1",
    "@connectrpc/connect-next": "^2.0.1",
    "@connectrpc/connect-node": "^2.1.1",
    "@connectrpc/connect-query": "^2.0.1",
    "@connectrpc/connect-web": "^2.1.1",
    "@tanstack/react-query": "^5.0.0",
    "next": "15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@bufbuild/buf": "^1.47.0",
    "@bufbuild/protoc-gen-es": "^2.2.3",
    "@connectrpc/protoc-gen-connect-query": "^2.0.1",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "typescript": "^5"
  }
}

```

## 2. Proto Definition & Generation

We define two proto files. `GreetService` imports `ElizaService` messages to ensure strict type parity between the proxy and the upstream backend.

**`proto/connectrpc/eliza/v1/eliza.proto`**

```protobuf
syntax = "proto3";

package connectrpc.eliza.v1;

message SayRequest {
  string sentence = 1;
}

message SayResponse {
  string sentence = 1;
}

service ElizaService {
  rpc Say(SayRequest) returns (SayResponse) {}
}

```

**`proto/greet/v1/greet.proto`**

```protobuf
syntax = "proto3";

package greet.v1;

import "connectrpc/eliza/v1/eliza.proto"; 

// This service mimics the remote service but is exposed locally
service GreetService {
  rpc Say(connectrpc.eliza.v1.SayRequest) returns (connectrpc.eliza.v1.SayResponse) {}
}

```

**`buf.gen.yaml`**

```yaml
version: v1
plugins:
  # 1. Generates Types AND Service Definitions (v2 Standard)
  - plugin: es
    path: ./node_modules/.bin/protoc-gen-es
    out: gen
    opt: target=ts

  # 2. Generates React Hooks (TanStack Query)
  - plugin: connect-query
    path: ./node_modules/.bin/protoc-gen-connect-query
    out: gen
    opt: target=ts

```

Run the generator:

```bash
npm run generate

```

## 3. The Proxy Utility

This utility dynamically implements a service definition by forwarding method calls to a target client. This allows us to "mount" the remote backend onto our local API route without manually rewriting every resolver.

**`src/utils/proxy.ts`**

```typescript
import { type DescService } from "@bufbuild/protobuf";
import { type Client } from "@connectrpc/connect";

export function createProxy<T extends DescService>(
  service: T,
  client: Client<T>,
) {
  const implementation: any = {};

  for (const method of service.methods) {
    const fnName = method.localName;

    implementation[fnName] = async (req: any) => {
      console.log(`[Proxy] Forwarding ${fnName} for ${service.typeName}`);

      // Call the client dynamically
      return await (client as any)[fnName](req);
    };
  }

  console.log(
    `[Proxy] Built methods for ${service.typeName}:`,
    Object.keys(implementation),
  );

  return implementation;
}

```

## 4. API Route Implementation

We create a backend connection to the demo Eliza service, then route our local `GreetService` to proxy that connection.

**`pages/api/[...connect].ts`**

```typescript
import { nextJsApiRouter } from "@connectrpc/connect-next";
import { ConnectRouter } from "@connectrpc/connect";
import { createClient } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-node";

// Generated Code
import { GreetService } from "@/gen/greet/v1/greet_pb";
import { ElizaService } from "@/gen/connectrpc/eliza/v1/eliza_pb";
import { createProxy } from "@/src/utils/proxy";

// 1. The Backend Connection (Target)
const elizaBackend = createClient(
  ElizaService,
  createConnectTransport({
    baseUrl: "https://demo.connectrpc.com",
    httpVersion: "1.1",
  }),
);

function routes(router: ConnectRouter) {
  // ---------------------------------------------------------
  // ROUTE 1: GreetService (Proxy -> ElizaService)
  // URL: /api/greet.v1.GreetService/Say
  // ---------------------------------------------------------
  router.service(GreetService, createProxy(GreetService, elizaBackend));
}

export const { handler, config } = nextJsApiRouter({ routes });
export default handler;

```

## 5. Client Consumption

The frontend uses `connect-web` to talk to the local Next.js API. Note that we are using the `GreetService` definition here, but the data types (`sentence`) are strictly typed to the original Eliza definition.

**`app/page.tsx`**

```typescript
"use client";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { createClient } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web"; 
import { GreetService } from "@/gen/greet/v1/greet_pb";

// Point to YOUR local API
const transport = createConnectTransport({ baseUrl: "/api" });
const client = createClient(GreetService, transport);

export default function ChatPage() {
  const [input, setInput] = useState("");
  const [history, setHistory] = useState<string[]>([]);

  const mutation = useMutation({
    mutationFn: async (msg: string) => {
      // Input is typed as SayRequest (from Eliza)
      const res = await client.say({ sentence: msg });
      
      // Output is typed as SayResponse (from Eliza)
      return res.sentence;
    },
    onSuccess: (answer) => {
      setHistory((prev) => [...prev, `Eliza (via Proxy): ${answer}`]);
    },
  });

  return (
    <div style={{ padding: "50px", fontFamily: "sans-serif" }}>
      <h1>Proxy Chat</h1>
      <div style={{ display: "flex", gap: "10px" }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Say something..."
          style={{ padding: "8px", flexGrow: 1 }}
        />
        <button onClick={() => mutation.mutate(input)} disabled={mutation.isPending}>
          {mutation.isPending ? "Sending..." : "Send"}
        </button>
      </div>

      <div style={{ marginTop: "20px", background: "#f5f5f5", padding: "20px", borderRadius: "8px" }}>
        {history.map((line, i) => (
          <p key={i} style={{ borderBottom: "1px solid #ddd", padding: "8px 0" }}>
            {line}
          </p>
        ))}
      </div>
    </div>
  );
}

```
