# Discord Opus Audio Hook - Main 2025 Guide

## Overview

This is a full walk-through on how to hook Discord’s internal Opus encoder module (`discord_voice.node`) so you can mess with the audio — mute it, distort it, force bitrate to 0, or whatever. We're gonna locate the encode function, patch it in memory with Frida, and test it live.

## What It Can Do

- Hook `WebRtcOpus_Encode` to mute audio
- Hook `WebRtcOpus_SetBitrate` to force 0kbps (can cause glitchy or silent mic)
- Modify Opus configuration directly via memory patching
- Works with regular Discord client (Electron app)
- Frida-based (no compiling DLLs needed)

## What You Need

- Node.js (>=18)
- `npm install frida ps-list`
- Frida ([https://frida.re](https://frida.re))
- Discord installed (not web)
- IDA Pro or Ghidra (for reverse engineering)

---

## IDA: Finding Function Offsets (Detailed Walkthrough)

1. Open `discord_voice.node` in IDA (ensure it is recognized as 64-bit PE file)
2. Press `Shift+F12` to access the **Strings** window
3. Search for keywords like `opus`, `bitrate`, or `encode`
4. Double-click one that seems relevant (e.g., `WebRtcOpus_SetBitrate`)
5. Press `X` to view all cross-references to that string or symbol
6. Go to the code reference (not data)
7. You will likely see a `call` to a function — scroll up to identify the start of that function
8. Look for a `push rbp` or `sub rsp,` instruction near the top — that's your function start
9. Note the address of that instruction
10. Subtract the module base (from IDA or Frida) to calculate the relative offset

For example:

```
Function start:   0x000000018000A750
Module base:      0x0000000180000000
Relative offset:  0xA750
```

Use this offset in your script.

---

## Editing Opus Internals via Reverse Engineering

### Understanding the Goal:

Opus operates on frame configuration objects passed into `WebRtcOpus_Encode`. This may include fields for:

- Bitrate
- Frame size
- Signal type
- Application mode (VoIP, audio)

Reverse engineering these parameters allows you to not just hook the encoder — but modify how it behaves mid-stream.

### Reverse Engineering the Encoder Struct:

1. Inside IDA, locate the call site for `WebRtcOpus_Encode`
2. Observe the first argument (`args[0]`) — typically a pointer to an encoder state struct
3. Use cross-references to this struct to find where fields are being written
4. Watch for constant values being moved to offsets like `[rcx+0x14]` or `[rdi+0x20]`
5. These are potential config parameters being set

### Strategy for Mapping Opus Config:

- Look for static values like 12000, 24000, 48000 (bitrate)
- Look for switches on integer constants (e.g., `cmp eax, 2`) — these might switch `application` modes
- Match struct offset sizes with known Opus encoder fields (from open-source `libopus` if needed)

Once you identify a field and its offset, you can overwrite it dynamically.

### Patching in Frida:

```js
Interceptor.attach(Module.findBaseAddress("discord_voice.node").add(0xA750), {
  onEnter(args) {
    const config = args[0];
    config.add(0x14).writeU32(6000); // sets the opus bitrate to 6kbps
    config.add(0x1C).writeU32(1);    // forces mode (e.g., OPUS_APPLICATION_VOIP)
  }
});
```

This approach lets you bend the encoder into entirely nonstandard behavior (silence, distortion, or extreme compression)

---

## Example (Frida Script That Works)

This script finds the correct Discord.exe process **with** `discord_voice.node` loaded, and hooks the encode function.

```js
const frida = require('frida');
const ps = require('ps-list');

async function grabDiscordProc() {
  let found = null;
  const list = await ps();
  for (let p of list.filter(x => x.name.toLowerCase().includes('discord'))) {
    try {
      let s = await frida.attach(p.pid);
      const test = await s.createScript(`
        send(Process.enumerateModulesSync().some(x => x.name === 'discord_voice.node'));
      `);
      test.message.connect(msg => {
        if (msg.payload) found = s;
      });
      await test.load();
      if (found) return found;
      await s.detach();
    } catch (e) {}
  }
  throw new Error("No discord PIDs found with discord_voice.node");
}

(async () => {
  const session = await grabDiscordProc();
  const script = await session.createScript(`
    const base = Module.findBaseAddress("discord_voice.node");
    const offset = 0x1A750; // use the offset you found in IDA (this offset may be outdated or wrong)
    Interceptor.attach(base.add(offset), {
      onEnter(args) {
        const samples = args[1];
        const size = parseInt(args[2]);
        for (let i = 0; i < size * 2; i++) {
          samples.add(i * 2).writeS16(0); // silence
        }
      }
    });
    send("hooked encode");
  `);
  script.message.connect(msg => console.log('[>]', msg.payload));
  await script.load();
})();
```

---

## Option: Change Opus Gain Instead (Alternative Hook)

If you want to make your mic sound louder/quieter instead of muting it, you can find and hook the gain adjustment function (usually right before Opus encoding).

1. Look in `WebRtcOpus_Encode`
2. Set breakpoints in IDA on values being multiplied or passed into PCM buffer
3. Hook and modify those args in Frida

Example (not actual offset):

```js
Interceptor.attach(Module.findBaseAddress("discord_voice.node").add(0x11C90), {
  onEnter(args) {
    const gainValue = args[3].toInt32();
    args[3] = ptr(gainValue * 2); // boost gain
  }
});
```

---

---

1. Launch Discord
2. Join a voice channel (important — voice module won’t load otherwise)
3. Run the Frida script with Node.js
4. Use the "Mic Test" under Voice & Video settings

You should hear **no audio**, **weird effects**, or **suppression of others** if the experimental hook works.

---

## Reminder

This is all educational. Don't troll randoms in vc or get banned. Test in private calls or with friends.

---

## Credit

Idea from Cypher, geniii, cancel, abaddon owner


