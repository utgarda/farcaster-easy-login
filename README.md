## Farcaster (farcaster.xyz) Easy Login (browser extension)

This extension allows you to sign in to farcaster.xyz without having to use a mobile device and instead use your browser wallet.

When you sign to login you must sign with your custody account( the account that owns your farcaster account) in order to be able to generate an auth token and sign in.

Download from Chrome-Store:

[https://chromewebstore.google.com/detail/farcaster-easy-login/bbjcdcdpmhmenjadgdkkkabhajfeefak?authuser=0&hl=en](https://chromewebstore.google.com/detail/farcaster-easy-login/bbjcdcdpmhmenjadgdkkkabhajfeefak?authuser=0&hl=en)

### Screenshot

![screenshot](/screen_1_A1F7.png)

### Demo Video Clip

https://github.com/user-attachments/assets/1719a813-6454-4475-9f40-bc68908580a9

### Build & install locally

Requires [Bun](https://bun.sh) (this project does not use npm/pnpm).

```sh
bun install
bun run build
```

Then load the `dist/` directory as an unpacked extension in `chrome://extensions` (toggle **Developer mode** on, click **Load unpacked**, pick `dist/`).

After code changes, re-run `bun run build` and click the reload icon on the extension card.

### Notes

- This repo is a continuation of this deprecated repo: [https://github.com/andrei0x309/warp-easy-login-browser-extension](https://github.com/andrei0x309/warp-easy-login-browser-extension)

### License

MIT
