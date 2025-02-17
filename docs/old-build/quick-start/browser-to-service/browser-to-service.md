# 3. Browser-to-Service

In the first section, we explored browser-to-browser messaging using local, i.e. browser-native, services and the Fluence network for message transport. In the second section, we developed a `HelloWorld` Wasm module and deployed it as a hosted service on the testnet peer `12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi` with service id `1e740ce4-81f6-4dd4-9bed-8d86e9c2fa50` . We can now extend our browser-to-browser messaging application with our hosted service.

Let's navigate to the `3-browser-to-service` directory in the VSCode terminal and install the dependencies:

```sh
npm install
```

And run the application with:

```sh
npm start
```

Which will open a new browser tab at `http://localhost:3000` . Following the instructions, we connect to any one of the displayed relay ids, open another browser tab also at `http://localhost:3000`, select a relay and copy and paste the client peer id and relay id into corresponding fields in the first tab and press the `say hello` button.

![Browser To Service Implementation](./Browser-To-Service-Implementation.png)

The result looks familiar, so what's different? Let's have a look at the Aqua file. Navigate to the `aqua/getting_started.aqua` file in your IDE or terminal:

```aqua
import "@fluencelabs/aqua-lib/builtin.aqua"

const HELLO_WORLD_NODE_PEER_ID ?= "12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi"
const HELLO_WORLD_SERVICE_ID ?= "1e740ce4-81f6-4dd4-9bed-8d86e9c2fa50"

data HelloWorld:
  msg: string
  reply: string

-- The service runs on a Fluence node
service HelloWorld:
    hello(from: PeerId) -> HelloWorld

-- The service runs inside browser
service HelloPeer("HelloPeer"):
    hello(message: string) -> string

func sayHello(targetPeerId: PeerId, targetRelayPeerId: PeerId) -> string:
    -- execute computation on a Peer in the network
    on HELLO_WORLD_NODE_PEER_ID:
        HelloWorld HELLO_WORLD_SERVICE_ID
        comp <- HelloWorld.hello(%init_peer_id%)

    -- send the result to target browser in the background
    on targetPeerId via targetRelayPeerId:
        co HelloPeer.hello(%init_peer_id%)

    -- send the result to the initiator
    <- comp.reply
```

And let's work it from the top:

- Import the Aqua standard library (1)
- Provide the hosted service peer id (3) and service id (4)
- Specify the `HelloWorld` struct interface binding (6-8) for the hosted service from the `marine aqua` export
- Specify the `HelloWorld` interface and function binding (11-12) for the hosted service from the `marine aqua` export
- Specify the `HelloPeer` interface and function binding (15-16) for the local service
- Create the Aqua workflow function `sayHello` (18-29)

Before we dive into the `sayHello` function, let's look at why we still need a local service even though we deployed a hosted service. The reason for that lies in the need for the browser client to be able to consume the message sent from the other browser through the relay peer. With that out of the way, let's dig in:

- The function signature (18) takes two arguments: `targetPeerId`, which is the client peer id of the other browser and the `targetRelayPeerId`, which is the relay id -- both parameters are the values you copy and pasted from the second browser tab into the first browser tab
- The first step is to call on the hosted service `HelloWorld` on the host peer `helloWorldPeerId`, which we specified in line 1
  - We bind the `HelloWorld` interface, on the peer `helloWorldPeerId`, i.e.,`12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi`, to the service id of the hosted service `helloWorldServiceId` , i.e. `1e740ce4-81f6-4dd4-9bed-8d86e9c2fa50`, which takes the %init\_\_peer\_\_id% parameter, i.e., the peer id of the peer that initiated the request, and pushes the result into `comp` (20-22)
  - We now want to send a result back to the target browser (peer) (25-26) using the local service via the `targetRelayPeerId` in the background as a `co` routine.
  - Finally, we send the `comp` result to the initiating browser

A little more involved than our first example but we are again getting a lot done with very little code. Of course, there could be more than one hosted service in play and we could implement, for example, hosted spell checking, text formatting and so much more without much extra effort to express additional workflow logic in our Aqua script.

This brings us to the end of this quick start tutorial. We hope you are as excited as we are to put Aqua and the Fluence stack to work. To continue your Fluence journey, have a look at the remainder of this book, take a deep dive into Aqua with the [Aqua book](../../../aqua-book/introduction.md) or dig into Marine and Aqua examples in the [repo](https://github.com/fluencelabs/examples).
