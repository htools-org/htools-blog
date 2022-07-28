---
title: 'A look into FxWallet'
date: 2022-07-28T00:00:00+05:30
draft: false
cover:
  image: 'images/cover.jpeg'
  alt: 'FxWallet'
  relative: false
---

[FxWallet](https://www.fxfi.io/) is a wallet by DxPool for
[Android](https://play.google.com/store/apps/details?id=com.pundix.functionx)
and [iOS](https://apps.apple.com/us/app/fxwallet/id1560943983) that's not talked
about enough.

It's a multi-currency self-custody wallet that supports a bunch of chains
including Handshake. This post focuses on the Handshake part.


## Features

1. Create and switch between multiple wallets (seed phrases)
2. Participate in Handshake auctions
3. A trustless marketplace for names
4. Name monitoring (for auction and expiry)


## Process

>**This post is based on this particular apk:** App Version: 2.10.0 (SHA256:
>`778260febdbf192c66d2d853a0c7b8dbec3d160f776135d360ed42f87935339e`) APK:
>`https://fxwallet.s3.ap-east-1.amazonaws.com/fxwallet_2.10.0.apk`

FxWallet does not seem to be open source, so we inspect the compiled app. There
are a few ways to go about this, and in such cases the easiest is to monitor
network traffic - see what the app sends to its servers.

On Android, even if not rooted, it's possible to inspect SSL traffic with apps
like [Packet
Capture](https://play.google.com/store/apps/details?id=app.greyshirts.sslcapture).
If you set it up, it doesn't work for FxWallet. It could be SSL Pinning or
something else. So we start with [Frida](https://frida.re/) that can hook into
internal methods and bypass SSL Pinning. Still doesn't show up in any capture.
Turns out, Flutter apps (that FxWallet is) compile dart into a separate library.
And that dart code does not use the OS certificate store and proxy/VPN settings.

Now we can use a flutter-specific script in Frida, or use a tool like
[ReFlutter](https://github.com/Impact-I/reFlutter) that makes the whole process
a bit easier (ofc we go with ReFlutter now). ReFlutter extracts the apk, finds
the correct location to hook in the dart binary, disables SSL pinning, and also
sets a proxy.

We finally start seeing requests in [BurpSuite](https://portswigger.net/burp)
(with proxy set up)! Now we go through the app and try all actions.

>Decompiling dart libraries doesn't give any useful code and can't be analyzed
>easily, network requests is the next best way.
>
>But it's important to note here that just because we don't see any malicious
>activity _right now_ doesn't mean the app won't do anything in the future
>(delayed activation or in newer versions).
>
>**It is not a guarantee that the app is safe and will stay safe forever**.


## Wallet

As you'd expect from a multi-currency wallet, it does not run a node on your
phone - it connects to remote servers to get balance, transaction history, etc.

When a new wallet is created, the seed phrase is not sent anywhere and is
locally stored encrypted with a password.

To identify wallets, the [extended public
key](https://learnmeabitcoin.com/technical/extended-keys) `xpub` (or the address
in case of account-based chains) is sent to the server in all requests. There is
no concept of login/register (usernames, email, tokens, etc.)

All transactions (auctions, marketplace actions) are requested from the server,
and signed locally, then sent back to be broadcasted.


## Handshake auctions

Not much to say here. Like any wallet and similar to
[Namebase](https://www.namebase.io/), you can view on-chain auctions and
participate (bid, reveal, redeem). The auction page also shows unconfirmed bids
and reveals which helps detect sniping.


## Trustless Marketplace

This is pretty interesting, and as far as I can tell, is the first (and only, as
of this post!) decentralized marketplace for Handshake names. It is based on
[HIP-1](https://hsd-dev.org/HIPs/proposals/0001/) and
[HIP-6](https://hsd-dev.org/HIPs/proposals/0006/).

Since they don't custody names, to put a name up for sale, the name has to be
transferred to a script that restricts the way the name can be used. Like any
transfer, it takes at least 2 days so listing for sale is not immediate.

The script roughly says:
```js
// A transfer can only be initated with the current owner's blessing
if (transfer) {
  return verify(owner)
} else {
  // And once a transfer has started, anyone can finalize it
  if (finalize) {
    return true
  } else {
    // A transfer once started cannot be cancelled with an UPDATE/REVOKE
    return false
  }
}
```

This is similar to [Shakedex](https://github.com/kurumiimari/shakedex), which
also builds on HIP-1, but uses locktime to create Dutch-style auctions.

Instead, in this marketplace, the owner creates a single presign for a "BIN"
(buy it now) price and can simultaneously accept bids for any price (HIP-6).

Once listed, interested parties can:
- Buy it immediately at the BIN price
- Send a bid to the owner for any price to negotiate
- Delete or update bids they've sent already

All bids are public and can be seen by everyone.

>**Note:** Buyers can update bids, and the app does show the new value, but the
>old bids are still technically spendable on chain. I'm not sure if the owner
>can update the BIN price, but the same might apply there.
>
>_Example:_ You place a bid of 100 HNS, then update it to 50 HNS. If the owner
>has been watching, they can accept and sell it for 100 HNS _after_ you've
>reduced it to 50 HNS in some cases.


## Privacy

**Information sent to FxWallet or a 3rd party service they use:**
- Wallet xpub (can see all transaction history, funds, names in that wallet)
- Favorites (getting push notifications on name state changes, ...)
- OneSignal collects: Device model, Android version, Language, Timezone, IP
  Address

**Information NOT sent anywhere:**
- Seed phrase or any key material
- Addresses/description stored in Address Book

(These are non-exhaustive lists.)


## Updates

With the app published on App Store and Play Store, they will be automatically
updated in the background.

Even if auto-update is disabled, FxWallet checks for updates on every start and
can force users to update if required (this is a rather common "feature" in apps
of which I am not personally a fan).


## API

The app primarily connects to 2 hosts:
- `https://api.fxfi.io:8765`
  - All API calls are made to this domain over HTTPS
  - Request and Response data are always JSON
  - Common headers: `user-agent: dio; api: 1.0.0; x-xpubkey: xpub...`
  - All examples in this post share this base url and headers
- `http://8.212.8.124:3000/price?base=USD`
  - The only endpoint used here is for getting the price of all coins/tokens


## Conclusion

I'm very excited to see what DxPool is building here and appreciate their
decision to go the self-custody route instead of taking the easy way with a
centralized marketplace! It's nice to see alternatives to Namebase.

However, this post is not an endorsement in any way, it just provides
information for those who want to learn about how the wallet works.

Also a reminder that FxWallet is not open source, may stop working (if their
servers shut down), and forced updates may include changes you may not want
including malicious code (ex: they are compromised).


## References

- [Root detection and SSL pinning bypass](https://buaq.net/go-101771.html)
- [Intercepting Flutter traffic on Android
  (ARMv8)](https://blog.nviso.eu/2020/05/20/intercepting-flutter-traffic-on-android-x64/)
- [reFlutter](https://github.com/Impact-I/reFlutter)
- [HIP-0001: Non-Interactive Name Atomic
  Swaps](https://hsd-dev.org/HIPs/proposals/0001/)
- [HIP-0006: Trustless Live Auctions for
  Handshake](https://hsd-dev.org/HIPs/proposals/0006/)
- [Bitcoin Script](https://en.bitcoin.it/wiki/Script)
