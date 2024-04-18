To see Universal Profile Browser Extension in action with EIP-6963 support:

1. Ensure you have the [Universal Profile Browser Extension](https://docs.lukso.tech/install-up-browser-extension) and at least one more wallet extension installed in your browser
2. Run `npm install`
3. Run `npm run build` (if you make any changes or would like to ensure there are no build errors)
4. Run `npm run dev` to run the app locally

Previous version of this repo built by the Metamask team can be found [here](https://github.com/MetaMask/vite-react-ts-eip-6963)

NextJS Version of this same App by the Wallet Connect team can be found [here](https://github.com/WalletConnect/EIP6963)

# A Guide to EIP-6963 for Developers

This article will fast track your understanding of EIP-6963, answer questions about how to implement, and provide example code for those who'd like to skip the chit chat.

The Example code repo above will help you create a simple implementation of EIP-6963 for detecting multiple injected providers (browser installed wallets (Externally Owned Accounts)) using ViteJS React + TypeScript.

The crux of what we will be doing is implementing the interfaces outlined in the EIP and listening to `eip6963:announceProvider` events in our dapp and storing the returned providers so they can be used anywhere and at anytime in our dApp giving us the ability to distinguish between providers.

![EIP-6963 React Demo](https://imgur.com/eWCd1S0.gif)

## The EIP-6963 Abstract

> An alternative discovery mechanism to `window.ethereum` for [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) providers which **_supports discovering multiple injected Wallet Providers_** in a web page using Javascript’s window events.

Before trying to understand any EIP, read and fully understand the [abstract](https://eips.ethereum.org/EIPS/eip-6963#abstract) (Quoted above), [motivation](https://eips.ethereum.org/EIPS/eip-6963#motivation), and do a full read through even if there are aspects that you don't understand.

> Note: If you don't understand any part, you can get further information through sites like Ethereum Magicians ([EIP-6963 MIPD](https://ethereum-magicians.org/t/eip-6963-multi-injected-provider-discovery/14076)) where you can get additional context from the authors and the community on most EIPs.

It would also be helpful to understand [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193).

Important line from the [EIP-6963 abstract](https://eips.ethereum.org/EIPS/eip-6963#abstract):

**_"An alternative discovery mechanism to `window.ethereum` for EIP-1193 providers"_**

Immediately this tells us that EIP-6963's Multi Injected Provider Discovery proposal aims to introduce a different approach or method for **discovering and interacting with EIP-1193 providers (in this case, Ethereum wallet providers)** in contrast to the existing method relying on the `window.ethereum` object.

Hopefully this info has helped us to frame our thinking as we progress through the EIP and start to implement this much better way of detecting and connecting to wallets.

## Issues Predating EIP-6963

In Ethereum decentralized applications (dApps), wallets traditionally expose their APIs using a JavaScript object known as 'the Provider.' The initial EIP created to standardize this interface was EIP-1193, and conflicts have risen among different wallet implementations. While EIP-1193 aimed to establish a common convention, the user experience suffered due to race conditions caused by wallets injecting their providers (clobbering) window's (Ethereum Object). This has been a huge criticism of Web3 UX and its shortcomings around wallet onboarding and connection. These race conditions resulted in multiple wallet extensions enabled in the same browser having conflicts, where the last injected provider typically took precedence.

Before EIP-6963, we had an unofficial standard through EIP-1193. Before it, a common convention in Ethereum dApps was for wallets to expose their API via a JavaScript object in the web page called 'the Provider.' Historically, Provider implementations had exhibited conflicting interfaces and behaviors between wallets. EIP-1193 formalized an Ethereum Provider API to promote wallet interoperability. The API was designed to be minimal, event-driven, and agnostic of transport and RPC protocols, and easily extended by defining new RPC methods and message event types. Although it provided a common convention for wallets to expose their API, user experience still suffered as a wallet would inject its provider into your browser window (Ethereum Object), causing race conditions. Having multiple wallet extensions installed meant that the last one to inject its provider typically was the one that would be detected and available for use.

EIP-5749 attempted to solve this issue with a window.evmprovider solution, and was a step in the right direction but ultimately was not adopted by enough wallets to solve our UX issue. This is where we end up with EIP 6963 bringing a standard to propose a way of doing Multi Injected Provider Discovery.

## What you need to know

As a developer, the first thing you should do in order to get familiar with [EIP-6963](https://eips.ethereum.org/EIPS/eip-6963), is to understand the initial motive, basic description, and more importantly the (TypeScript) interfaces and objects needed to implement this new approach.

## Implementing EIP-6963 in a ViteJS React + TS Application

The Provider MUST implement and expose the interfaces defined in the EIP. Let's review them real quick, i have also provided links to each section of the original EIP for reference.

[Provider Info](https://eips.ethereum.org/EIPS/eip-1193#request)

> Each Wallet Provider will be announced with the following interface:

```ts=
/**
 * Represents the assets needed to display a wallet
 */
interface EIP6963ProviderInfo {
  uuid: string;
  name: string;
  icon: string;
  rdns: string;
}
```

[Provider Detail](https://eips.ethereum.org/EIPS/eip-1193#provider-detail)

> used as a composition interface to announce a Wallet Provider and related metadata about the Wallet Provider

```ts=
interface EIP6963ProviderDetail {
  info: EIP6963ProviderInfo;
  provider: EIP1193Provider;
}
```

[Announce](https://eips.ethereum.org/EIPS/eip-1193#announce-and-request-events)

> The `EIP6963AnnounceProviderEvent` interface MUST be a `CustomEvent` object with a `type` property containing a string value of `eip6963:announceProvider` and a detail property with an object value of type `EIP6963ProviderDetail`.

```ts=
// Announce Event dispatched by a Wallet
interface EIP6963AnnounceProviderEvent extends CustomEvent {
  type: "eip6963:announceProvider";
  detail: EIP6963ProviderDetail;
}
```

[Request Events](https://eips.ethereum.org/EIPS/eip-1193#announce-and-request-events)

> The `EIP6963RequestProviderEvent` interface MUST be an Event object with a type property containing a string value of `eip6963:requestProvider`.

```ts=
// Request Event dispatched by a DApp
interface EIP6963RequestProviderEvent extends Event {
  type: "eip6963:requestProvider";
}
```

With these interfaces defined for us, we can spin up a ViteJS React + TypeScript application and update the `src/vite-env.d.ts` with those interfaces:

### vite-env.d.ts

```ts=
/// <reference types="vite/client" />

interface EIP6963ProviderDetail {
  info: EIP6963ProviderInfo;
  provider: EIP1193Provider;
}

interface EIP6963ProviderInfo {
  walletId: string;
  uuid: string;
  name: string;
  icon: string;
}

interface EIP1193Provider {
  isStatus?: boolean;
  host?: string;
  path?: string;
  sendAsync?: (request: { method: string, params?: Array<unknown> }, callback: (error: Error | null, response: unknown) => void) => void
  send?: (request: { method: string, params?: Array<unknown> }, callback: (error: Error | null, response: unknown) => void) => void
  request: (request: { method: string, params?: Array<unknown> }) => Promise<unknown>
}

type EIP6963AnnounceProviderEvent = {
  detail:{
    info: EIP6963ProviderInfo,
    provider: EIP1193Provider
  }
}
```

We can then create a `hooks` directory and add the two following files:

### useSyncProviders.tsx

```ts
import { useSyncExternalStore } from "react";
import { store } from "./store";

export const useSyncProviders = () =>
  useSyncExternalStore(store.subscribe, store.value, store.value);
```

### store.tsx

```ts
declare global {
  interface WindowEventMap {
    "eip6963:announceProvider": CustomEvent;
  }
}

let providers: EIP6963ProviderDetail[] = [];

export const store = {
  value: () => providers,
  subscribe: (callback: () => void) => {
    function onAnnouncement(event: EIP6963AnnounceProviderEvent) {
      if (providers.map((p) => p.info.uuid).includes(event.detail.info.uuid))
        return;
      providers = [...providers, event.detail];
      callback();
    }
    window.addEventListener("eip6963:announceProvider", onAnnouncement);
    window.dispatchEvent(new Event("eip6963:requestProvider"));

    return () =>
      window.removeEventListener("eip6963:announceProvider", onAnnouncement);
  },
};
```

With this hook and store in place, we can now create a basic compnentat the following location: `src/components/DiscoverWalletProviders.tsx` this component will map over those discovered providers and list them using a button that handles connections to those wallets:

### DiscoverWalletProviders.tsx

```ts
import { useState } from "react";
import { useSyncProviders } from "~/hooks/useSyncProviders";
import { formatAddress } from "~/utils";

export const DiscoverWalletProviders = () => {
  const [selectedWallet, setSelectedWallet] = useState<EIP6963ProviderDetail>();
  const [userAccount, setUserAccount] = useState<string>("");
  const providers = useSyncProviders();

  const handleConnect = async (providerWithInfo: EIP6963ProviderDetail) => {
    const accounts = await providerWithInfo.provider
      .request({ method: "eth_requestAccounts" })
      .catch(console.error);

    if (accounts?.[0]) {
      setSelectedWallet(providerWithInfo);
      setUserAccount(accounts?.[0]);
    }
  };

  return (
    <>
      <h2>Wallets Detected:</h2>
      <div>
        {providers.length > 0 ? (
          providers?.map((provider: EIP6963ProviderDetail) => (
            <button
              key={provider.info.uuid}
              onClick={() => handleConnect(provider)}
            >
              <img src={provider.info.icon} alt={provider.info.name} />
              <div>{provider.info.name}</div>
            </button>
          ))
        ) : (
          <div>there are no Announced Providers</div>
        )}
      </div>
      <hr />
      <h2>{userAccount ? "" : "No "}Wallet Selected</h2>
      {userAccount && (
        <div>
          <div>
            <img
              src={selectedWallet.info.icon}
              alt={selectedWallet.info.name}
            />
            <div>{selectedWallet.info.name}</div>
            <div>({formatAddress(userAccount)})</div>
          </div>
        </div>
      )}
    </>
  );
};
```

In the code above, as soon as the page is rendered we are logging the providers that we have detected. See lline 11. We can loop over these providers and create a button for each one, this button will be used to call `eth_requestAccounts`.

Finally we can link to this coomponent from `src/App.tsx`

### App.tsx

```ts
import "./App.css";
import { DiscoverWalletProviders } from "~/components/DiscoverWalletProviders";

function App() {
  return (
    <>
      <DiscoverWalletProviders />
    </>
  );
}

export default App;
```

With these few steps we have implemented EIP-6963 into a React application utilizing the TypeScript interfaces outlined in the EIP. At a basic level, that's it.

#### Commitment to Standards

Incorporating EIP-6963 demonstrates our ongoing commitment to align with industry standards and best practices. This ensures that our tools and services remain at the forefront of technological advancements in the blockchain domain.

## Community Support

The alternative discovery mechanism works for wallets that have implemented support for EIP-6963.
This includes Universal Profile Browser Extension, MetaMask, Coinbase, Trust Wallet, OKX, and other major wallets.

See the [list of wallets that support EIP-6963](https://github.com/WalletConnect/EIP6963/blob/master/src/utils/constants.ts).

## Backwards compatibility

Dapps that do not support EIP-6963 can still detect the Universal Profile Browser Extension using the `window.lukso` provider.
However, we recommend adding support to improve the user experience for multiple installed wallets.
Read more about [EIP-6963 backwards compatibility](https://eips.ethereum.org/EIPS/eip-6963#backwards-compatibility).

## Resources for Reading on EIP-6963

[Overview of EIP-6963: A Possible Solution for Multiple Wallet Conflict](https://mundus.dev/tpost/76iu0k1ot1-overview-of-eip-6963-a-possible-solution)
[EIP-6963](https://eips.ethereum.org/EIPS/eip-6963)
[EIP-6963 Standardizes Your Browser Wallet Experience](https://www.youtube.com/watch?v=SWmknCUwr3Y&t=281s)
