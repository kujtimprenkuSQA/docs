---
id: donation
title: Donation
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import {CodeTabs, Language, Github} from "@site/src/components/codetabs"

Our donation example enables to forward money to an account while keeping track of it. It is one of the simplest examples on making a contract handle tranfers.

![img](/docs/assets/examples/donation.png)
_Frontend of the Donation App_

---

## Obtaining the Donation Example

You have two options to start the Donation Example.

1. You can use the app through `Github Codespaces`, which will open a web-based interactive environment.
2. Clone the repository locally and use it from your computer.

| Codespaces                                                                                                                      | Clone locally                                               |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/near-examples/donation-examples) | 🌐 `https://github.com/near-examples/donation-examples.git` |

---

## Structure of the Example

The example is divided in two main components:

1. The smart contract, available in two flavors: rust and javascript
2. The frontend, that interacts with an already deployed contract.

<Tabs>

  <TabItem value="🌐 JavaScript">

```bash
┌── sandbox-ts # sandbox testing
│    ├── src
│    │    └── main.ava.ts
│    ├── ava.config.cjs
│    └── package.json
├── package.json
├── src # contract's code
│    ├── contract.ts
│    ├── model.ts
│    └── utils.ts
├── package.json # package manager
├── README.md
└── tsconfig.json # test script
```

  </TabItem>

  <TabItem value="🦀 Rust">

```bash
┌── sandbox-ts # sandbox testing
│    ├── src
│    │    └── main.ava.ts
│    ├── ava.config.cjs
│    └── package.json
├── package.json
├── src # contract's code
│    ├── donation.rs
│    └── lib.rs
├── build.sh # build script
├── Cargo.toml # package manager
├── README.md
├── rust-toolchain.toml
└── test.sh # test script
```

  </TabItem>

</Tabs>

---

## Frontend

The donation example includes a frontend that interacts with an already deployed smart contract, allowing user to donate NEAR tokens to a faucet service.

<hr class="subsection" />

### Running the Frontend

To start the frontend you will need to install the dependencies and start the server.

```bash
cd frontend
yarn
yarn start
```

Go ahead and login with your NEAR account. If you don't have one, you will be able to create one in the moment. Once logged in, input the amount of NEAR you want to donate and press the donate button. You will be redirected to the NEAR Wallet to confirm the transaction. After confirming it, the donation will be listed in the "Latest Donations".

<hr class="subsection" />

### Understanding the Frontend

The frontend is composed by a single HTML file (`/index.html`), while the logic lives in `/assets/js/index.js`, which communicates with the contract through `/assets/js/near/utils.js`.

<Language value="" language="js">
  <Github fname="index.js"
          url="https://github.com/near-examples/donation-examples/blob/main/frontend/index.js"
          start="71" end="93" />
  <Github fname="near-interface.js"
          url="https://github.com/near-examples/donation-examples/blob/main/frontend/near-interface.js"
          start="29" end="32" />
  <Github fname="near-wallet.js"
          url="https://github.com/near-examples/donation-examples/blob/main/frontend/near-wallet.js"
          start="105" end="113" />
</Language>

An interesting aspect of the donation example is that it showcases how to retrieve a result after being redirected to the
NEAR wallet to accept a transaction.

---

## Smart Contract

The contract exposes methods to donate money (`donate`), and methods to retrieve the recorded donations (e.g. `get_donation_by_number`).

<CodeTabs>
  <Language value="🌐 JavaScript" language="ts">
    <Github fname="contract.ts"
            url="https://github.com/near-examples/donation-examples/blob/main/contract-ts/src/contract.ts"
            start="16" end="44" />
  </Language>
  <Language value="🦀 Rust" language="rust">
    <Github fname="lib.rs"
            url="https://github.com/near-examples/donation-examples/blob/main/contract-rs/src/donation.rs"
            start="20" end="49" />
  </Language>
</CodeTabs>

<hr class="subsection" />

### Testing the Contract

The contract readily includes a set of unit and sandbox testing to validate its functionality. To execute the tests, run the following commands:

<Tabs>
  <TabItem value="🌐 JavaScript">

  ```bash
  cd contract-ts
  yarn
  yarn test
  ```

  </TabItem>
  <TabItem value="🦀 Rust">
  
  ```bash
  cd contract-rs
  ./test.sh
  ```

  </TabItem>

</Tabs>

:::tip
The `integration tests` use a sandbox to create NEAR users and simulate interactions with the contract.
:::

<hr class="subsection" />

### Deploying the Contract to the NEAR network

In order to deploy the contract you will need to [create a NEAR account](/develop/contracts/quickstart#create-a-testnet-account).

<Tabs>
  <TabItem value="🌐 JavaScript">

```bash
cd contract-ts
yarn build
near deploy <accountId> ./build/contract.wasm
```

  </TabItem>
  <TabItem value="🦀 Rust">

```bash
cd contract-rs
./build.sh
near deploy <accountId> ./target/wasm32-unknown-unknown/release/contract.wasm
```

  </TabItem>
</Tabs>

:::tip
To interact with your contract from the [frontend](#frontend), simply replace the variable `CONTRACT_NAME` in the `index.js` file.
:::

<hr class="subsection" />

### CLI: Interacting with the Contract

```bash
near view donation.near-examples.testnet get_donations
near call donation.near-examples.testnet donate --accountId <accountId> --deposit 0.1 -->
```

---

## Moving Forward

A nice way to learn is by trying to expand a contract. Modify the donation example so it accumulates the money in the contract
instead of sending it immediately. Then, make a method that only the `beneficiary` can call to retrieve the money.
