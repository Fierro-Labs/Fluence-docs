# Advanced example

## Intro

In this section, we will demonstrate how the Fluence JS Client and Aqua can be used to provide an IPFS API to Aqua and use it with the Fluence stack.

We will implement given Aqua IPFS interface to interact with IPFS p2p network through the js implementation.

## Aqua code

Let's start by defining Aqua interface. The signature will tell us about a service behavior a bit:

```aqua
-- Export signature for making our functions visible outside this file.
export exists, upload_string, remove

-- Alias for making function types closer to IPFS domain definitions.
alias CID : string

-- Service interface. It is what we're going to implement. Implementation could vary depending on client needs.
service IpfsClient("ipfs_client"):
    exists(cid: CID) -> bool
    upload_string(contents: string) -> CID
    remove(cid: CID) -> string

-- Exported functions. Export declarations defined on the first line of code snippet.
func exists(cid: CID) -> bool:
    <- IpfsClient.exists(cid)

func upload_string(contents: string) -> CID:
    <- IpfsClient.upload_string(contents)

func remove(cid: CID) -> string:
    <- IpfsClient.remove(cid)

```

## Install dependencies

Initialize an empty `npm` package:

```sh
npm init
```

We will need these three packages for the application runtime: JS Client API, JS Client implementation for Node.js, and the package containing a well-maintained list of relay nodes.

```sh
npm install @fluencelabs/js-client.api @fluencelabs/js-client.node @fluencelabs/fluence-network-environment
```

Also, we need some specific packages for implementing IPFS service

```sh
npm install helia @helia/strings @helia/dag-json multiformats blockstore-fs datastore-fs
```

We will need the Fluence CLI to use the Aqua compiler, but only as a development dependency:

```sh
npm install --save-dev @fluencelabs/cli
```

Aqua comes with the standard library, which can be accessed from "@fluencelabs/aqua-lib" package. All Aqua packages are only needed at compiler time, so we install it as a development dependency:

```sh
npm install --save-dev @fluencelabs/aqua-lib
```

And last, but no least, we will need TypeScript:

```sh
npm install --save-dev typescript @types/node ts-node
npx tsc --init
```

## Setting up Aqua compiler

Let's put Aqua described earlier in the `aqua/files.aqua` file. You probably want to keep the generated TypeScript in the same directory with other typescript files, usually `src`. Let's create the `src/_aqua` directory for that.

The overall project structure looks like this:

```
 ┣ aqua
 ┃ ┗ files.aqua
 ┣ src
 ┃ ┣ _aqua
 ┃ ┃ ┗ files.ts
 ┃ ┗ index.ts
 ┣ package-lock.json
 ┣ package.json
 ┗ tsconfig.json
```

To compile Aqua, we can use the Fluence CLI with `npm`:

```sh
npx fluence aqua -i ./aqua/ -o ./src/_aqua
```

To watch for Aqua file changes, add the `-w` flag:

```sh
npx fluence aqua -w -i ./aqua/ -o ./src/_aqua
```

We recommend to store this logic inside a script in the `packages.json` file:

```javascript
{
  ...
  "scripts": {
    ...
    "compile-aqua": "fluence aqua -i ./aqua/ -o ./src/_aqua",
    "watch-aqua": "fluence aqua -w -i ./aqua/ -o ./src/_aqua"
  },
  ...
}
```

## Using the compiled code in a TypeScript application

Using the code generated by the compiler is as easy as calling a function. The compiler generates all the boilerplate needed to send a particle into the network and wraps it into a single call. It also generates a function for service callback registration. Note that all the type information and therefore type checking and code completion facilities are there!

We will start with implementing `utils.ts` file:

```typescript
import { FsBlockstore } from 'blockstore-fs'; // Import block storage
import { FsDatastore } from 'datastore-fs'; // Import app info storage
import { createHelia } from 'helia'; // Lib for interacting with IPFS

const blockstore = new FsBlockstore('./temp/store');
const datastore = new FsDatastore('./temp/data');

export async function makeHelia() { // Helper func for creating client node which will be used for reading IPFS
    return await createHelia({
        blockstore,
        datastore,
    });
}

export async function timeout<T,>(promise: Promise<T>, timeout: number, abort: AbortController): Promise<T> {
    const timerPromise = new Promise<never>((_, reject) => {
        setTimeout(() => {
            reject(new Error("Timeout"));
            abort.abort();
        }, timeout).unref();
    });

    return await Promise.race([promise, timerPromise]);
}

export function noop() {}
```

Let's see how to use the generated code in our application. The `index.ts` file looks this way:

```typescript
import '@fluencelabs/js-client.node'; // Import the JS Client implementation. Don't forget to add this import!
import { Fluence } from '@fluencelabs/js-client.api'; // Import the API for JS Client
import { exists, registerIpfsClient, remove, upload_string } from './_aqua/files.js'; // Aqua compiler provides functions which can be directly imported like any normal TypeScript function.
import { readFile } from 'node:fs/promises';
import { strings } from '@helia/strings';
import { CID } from 'multiformats/cid';
import { dagJson } from '@helia/dag-json';
import { randomTestNet } from '@fluencelabs/fluence-network-environment';
import { makeHelia, noop, timeout } from './utils.js';

async function main() {
    await Fluence.connect(randomTestNet());

    registerIpfsClient({
        async exists(cid: string) {
            const helia = await makeHelia();

            const s = strings(helia);

            const c = CID.parse(cid);

            const controller = new AbortController();
            const content = await timeout(s.get(c, { signal: controller.signal }), 5 /* 5 sec */, controller).catch(noop);

            const result = Boolean(content);
            await helia.stop();
            return result;
        },
        async remove(cid: string) {
            const helia = await makeHelia();

            const c = CID.parse(cid);

            const isPinned = await helia.pins.isPinned(c);

            if (isPinned) {
                console.log('Remove pinned entry:', c.toString());
                const pin = await helia.pins.rm(c);
                await helia.gc();
            }

            await helia.stop();
            return c.toString();
        },
        async upload_string(contents: string) {
            const helia = await makeHelia();

            const s = strings(helia);

            const cid = await s.add(contents);
            await helia.pins.add(cid);

            await helia.stop();
            return cid.toString();
        }
    });

    const cid = await upload_string('Hello world!!!', { ttl: 120000 });

    console.log('cid:', cid.toString());

    console.log('Is entry exists:', await exists(cid.toString(), { ttl: 120000 }));

    await remove(cid.toString(), { ttl: 120000 });

    console.log('Is entry exists:', await exists(cid.toString(), { ttl: 120000 }));

    await Fluence.disconnect();
}

main();
```

Let's try running the example:

```sh
./node_modules/.bin/ts-node-esm ./src/index.ts
```

If everything has been done correctly, you should see the following text in the console:

```
id: bafkreicdktp5u4gi6djzsg454pkw3s3ot4x4nqbrnurvwy5p5m4ii4nnuq
Is entry exists: true
Remove pinned entry: bafkreicdktp5u4gi6djzsg454pkw3s3ot4x4nqbrnurvwy5p5m4ii4nnuq
Is entry exists: false
```

## Conclusion:

Now we have a working service implementation which can be deployed to peer and interacted.

You learned:
- Basic aqua syntax
- How to implement a peer
- Notion of IPFS

You can find remote service call examples [here](https://github.com/fluencelabs/examples)

### Notes

- IPFS local peer stores data in your `./src/temp` folder. You can remove this folder if you want to start to manually clear written data.
- There is an almost 0% chance to pass something to another IPFS peer as we break connection too soon. You can fix this by rewriting code and omitting `helia.stop()` instructions. The longer you keep connection - the higher chances to exchange data.
- This example is not suitable for Fluence peer, because local file changes are not guaranteed to be persistent. 