# TPM Rewrite

A Node.js-based helper/macro for Hypixel SkyBlock flipping that integrates with CoflNet and the TPM backend. It logs Minecraft accounts in via Mineflayer, talks to CoflNet over WebSocket, and sends status/flip info to your Discord via webhook.


> **Security note**: This README only describes what the code does and how to run it. You should never share your `config.json5`, Cofl session, Minecraft tokens, or Discord webhooks with anyone.

---

## 1. Requirements

- **OS**: Windows 10/11 (project is Node-based and should also work on Linux/macOS)
- **Node.js**: 18.x LTS recommended (package targets `node18-*` in `package.json`)
- **npm**: Comes with Node
- **Discord**: Access to the TPM Discord server to get your Discord ID and any official instructions

---

## 2. Installation

1. **Download / extract the project**
   - Place the folder somewhere like `C:/Users/you/TPM-rewrite-main/TPM-rewrite-main`.

2. **Install dependencies**
   - Open a terminal in the project folder (where `package.json` and `index.js` are).
   - Run:
     ```bash
     npm install
     ```

3. **First run will create a config file**
   - Run once to generate `config.json5` from the default config:
     ```bash
     npm start
     ```
   - On first run it will:
     - Prompt you for your **Minecraft IGN** if `igns[0]` is empty.
     - Create `config.json5` in the project root, based on `config.js` defaults.

4. **Edit `config.json5`**
   - Open `config.json5` in a text editor.
   - At minimum, set:
     - `igns`: Your Minecraft IGNs, e.g. `["YourIGN"]`.
     - `discordID`: The Discord user ID you get from the TPM Discord `/get_discord_id` command.
     - `webhook`: Your Discord webhook URL (channel where TPM will send messages).
   - Optionally adjust other settings:
     - `visitFriend`: Island to flip on.
     - `useCookie` / `autoCookie`: Whether to use/buy booster cookies.
     - `relist`: Enable auto-relisting.
     - `percentOfTarget`, `listHours`: Price and duration rules.
     - `skip` / `doNotRelist` / `autoRotate`: Fine-tune behaviour.

---

## 3. Running TPM

From the project folder:

```bash
npm start
```

or equivalently:

```bash
node index.js
```

On startup the program will:

- Ensure `config.session` exists (a random UUID is set if missing).
- Log in Mineflayer bots for each IGN in `config.igns`.
- Connect to the TPM backend WebSocket at `ws://104.245.104.234:1241`.
- Connect each bot to CoflNet via WebSocket:
  - `wss://sky.coflnet.com/modsocket?...&SId=<your session>`.
- Send a "Started TPM" embed to your configured Discord webhook.
- Keep a terminal prompt open so you can send commands (e.g. `/stats`, `/ping`, etc.) to the running bots.

Most actual control flows through the TPM Discord integration and CoflNet; this repo just implements the local client that those services drive.

---

## 4. Files and Folders

- `index.js`
  - Main entrypoint. Starts bots, hooks into TPM WebSocket, and exposes a small CLI for sending commands.
- `config.js`
  - Manages defaults and reading/writing `config.json5` using `json5` and `golden-fleece`.
- `TpmSocket.js`
  - Connects to the TPM backend at `ws://104.245.104.234:1241`.
  - Sends your `discordID`, `webhook`, IGNs, and some settings to that server.
  - Receives commands like `startBot`, `killBot`, `buyFlip`, `sendTerminal`, etc., and forwards them to your local bots.
- `TPM-bot/`
  - `bot.js`: Creates a Mineflayer bot, can log in with IGN or an access token.
  - `AhBot.js`: Wrapper that wires together all handlers for a single account.
  - `AutoBuy.js`: Handles buying flips from CoflNet and the auction house.
  - `RelistHandler.js`: Handles relisting auctions, cookies, drill part removal, expired auctions, etc.
  - `AutoIsland.js`: Ensures the bot is on the correct island (private / hub / friend).
  - `BankHandler.js`: Handles coin deposit/withdraw interactions with the bank.
  - `CoflWs.js`: WebSocket client for `wss://sky.coflnet.com/modsocket` using your `session` value.
  - `MessageHandler.js`: Parses in-game chat, updates internal state, and sends Discord webhooks.
  - `StateManager.js`: Tracks current state and queue, persists listing-related data in `SavedData/`.
  - `TokenHandler.js`: Exchanges a Minecraft access token for UUID and username via the official Mojang services.
  - `Utils.js`: Shared helpers: number/time formatting, Discord webhook sending, price helpers, stats, etc.
- `logger.js`
  - Wraps `winston` logging, writes log files to `./logs` and exposes helpers used throughout the project.
- `SavedData/`
  - Created/used at runtime to store per-bot queue and bid data.

---

## 5. Security / Privacy Notes

This section is based on a static review of the source code in this repo. It is **not** a formal security audit.

### External endpoints used

The code talks to the following external services:

- **Minecraft / Mojang services**
  - `https://api.minecraftservices.com/minecraft/profile`
    - Used in `TPM-bot/TokenHandler.js` to turn a Bearer access token into a UUID + username.
  - `https://api.mojang.com/users/profiles/minecraft/<ign>`
    - Used in `TPM-bot/bot.js` to resolve UUIDs.
- **CoflNet**
  - WebSocket: `wss://sky.coflnet.com/modsocket?...&SId=<session>`
  - HTTP:
    - `https://sky.coflnet.com/api/auction/<auctionId>`
    - `https://sky.coflnet.com/api/price/nbt`
  - Used for flip data, inventory pricing, and account status.
- **TPM backend**
  - WebSocket: `ws://104.245.104.234:1241`
  - On connect, it sends:
    - `discordID`
    - Your `webhook` URL (or array of URLs)
    - Current IGNs and some `settings`
    - `allowedIDs`
  - It can send commands to your local client (start/stop bots, queue flips, send terminal commands, etc.).
- **Discord**
  - Your own configured webhook(s): `config.webhook` and `config.sendAllFlips`.
  - The code posts embeds and optional files (logs) to those URLs using `axios.post`.
- **Static assets**
  - Various image URLs from `cdn.discordapp.com`, `media.discordapp.net`, and `https://mc-heads.net/head/<uuid>.png`.

### Sensitive values and where they go

- **Minecraft access token / `ign` used as token**
  - If `ign.length > 16`, it is treated as an access token.
  - The token is:
    - Sent to `https://api.minecraftservices.com/minecraft/profile` as `Authorization: Bearer <token>`.
    - Used locally as the Mineflayer `session.accessToken`.
  - **From the source code, it is not forwarded to any other third-party servers** besides Mojang.
- **Cofl session (`config.session`)**
  - Stored in `config.json5`.
  - Sent as `SId` query parameter to `wss://sky.coflnet.com/modsocket`.
  - Not sent to the TPM backend IP in this code.
- **Discord webhook URL(s)**
  - Stored in `config.json5` as `webhook` / `sendAllFlips`.
  - Used:
    - Locally to send Discord messages from this client.
    - Sent once to the TPM backend over `ws://104.245.104.234:1241` as part of the `loggedIn` payload.
- **Logs**
  - Written to `./logs` via `winston`.
  - `getLatestLog()` returns a `FormData` stream.
  - `sendLatestLog` (in `Utils.js`) can post that file to your Discord webhook on crash.j

---

## 6. Uninstall / Cleanup

To remove TPM from your system:

1. Delete the project folder.
2. Optionally delete any created folders:
   - `./logs/`
   - `./SavedData/`
   - `config.json5`
3. Revoke any:
   - Discord webhooks you shared with TPM.
   - Cofl sessions you no longer want to use.
   - Minecraft access tokens that were ever typed into the tool.

---

## 7. Troubleshooting

- **`npm install` fails**
  - Make sure you have Node.js 18.x installed (`node -v`).
  - Delete `node_modules` and `package-lock.json` and run `npm install` again if needed.

- **Bot doesn't log in**
  - Check that `igns` in `config.json5` is correct and that the account can join `play.hypixel.net` manually.

- **No Discord messages**
  - Verify that `webhook` in `config.json5` is a valid Discord webhook URL.
  - Ensure the channel still exists and the webhook wasn't deleted.

- **Worried about security**
  - Regenerate/revoke:
    - Minecraft tokens used with the tool.
    - Cofl session tokens.
    - Discord webhooks.
  - Run the project only on accounts / machines you are comfortable experimenting with.
