# League Voice Coach Prototype

Build a working prototype of a Discord voice bot that acts as a live League of Legends coach using OpenAI Realtime.

## Goal

A user joins a Discord voice channel with the bot and can ask questions such as:

- "What item should I build now?"
- "Who should I focus?"
- "Can we contest the next objective?"

The bot should answer by voice with low latency and base its answer on the player's current League game state.

## Scope

Prototype first. Optimize for one user, one Discord guild, one active voice session, and one League game at a time. Do not build a polished SaaS or multi-tenant system yet.

## Architecture

Use TypeScript throughout.

### Components

1. `apps/discord-bot`
   - Discord bot using `discord.js` and `@discordjs/voice`.
   - Supports slash commands `/join` and `/leave`.
   - `/join` joins the invoking user's current voice channel.
   - Receive that user's voice audio only for the prototype.
   - Decode Discord Opus audio to PCM.
   - Resample/convert audio to the format expected by OpenAI Realtime.
   - Maintain a WebSocket session to OpenAI `gpt-realtime-2.1`.
   - Forward user speech to OpenAI.
   - Receive Realtime audio output, convert/resample/encode as needed, and play it back in Discord.
   - Support interruption/barge-in reasonably well.

2. `apps/league-companion`
   - Small local Node.js process that runs on the same machine as League of Legends.
   - Poll Riot's Live Client Data API at `https://127.0.0.1:2999/liveclientdata/...`.
   - Ignore the local self-signed TLS certificate only for this localhost API.
   - Detect whether a live game is running.
   - Read the active player Riot ID automatically.
   - Build a compact normalized game state.
   - Send that state to the cloud backend over an outbound WebSocket.
   - Reconnect automatically.

3. `apps/control-plane`
   - Cloudflare Worker, deployed via SST.
   - Accept WebSocket connections from the local League companion.
   - Keep the latest normalized game state for the prototype.
   - Expose an authenticated way for the Discord bot to retrieve the latest state.
   - Prefer a Cloudflare Durable Object for the WebSocket/session state if it materially simplifies this.
   - Do not unnecessarily proxy Discord voice audio through Cloudflare.

4. `packages/shared`
   - Shared TypeScript types/schemas.
   - Use Zod where useful.
   - Define `GameState`, WebSocket messages, and internal protocol types here.

## Important hosting constraint

Do not try to run the persistent Discord voice connection inside a normal Cloudflare Worker. Discord voice uses a persistent gateway/voice connection and UDP/Opus media flow. The Cloudflare part should be the control plane/session bridge, not the Discord voice host.

For the prototype, run `apps/discord-bot` as a normal long-lived Node process locally. Structure it so it can later be moved to a container/Fargate-style environment if desired.

Use SST for the Cloudflare infrastructure. Keep infra intentionally small.

## OpenAI Realtime

Use the current OpenAI Realtime WebSocket API with `gpt-realtime-2.1`.

Secrets must never be committed. Read the OpenAI API key from `OPENAI_API_KEY`.

The Discord bot should establish one Realtime session per active Discord coaching session.

Configure the model as a concise League coach:

- spoken, natural responses
- short answers suitable while playing
- explain only what matters immediately
- ask for live game state via tool calls when needed
- do not hallucinate unknown cooldowns, timers, enemy information, or player intent

Provide a Realtime function/tool named approximately:

`get_current_game_state`

The tool has no user-supplied parameters. When invoked, the Discord bot fetches the latest normalized state from the control plane and returns it to the model.

Do not continuously dump full game state into the model context. Fetch it on demand through the tool.

## Riot Live Client Data

Use official localhost Live Client Data endpoints, including as useful:

- `/liveclientdata/activeplayer`
- `/liveclientdata/playerlist`
- `/liveclientdata/eventdata`
- `/liveclientdata/gamestats`

Create a normalized state roughly like:

```ts
export interface GameState {
  updatedAt: string;
  gameTimeSeconds: number;
  activePlayer: {
    riotId: string;
    champion: string;
    level: number;
    currentGold: number;
    team: "ORDER" | "CHAOS" | string;
    position?: string;
    scores?: {
      kills: number;
      deaths: number;
      assists: number;
      creepScore: number;
    };
    items: Array<{
      itemId: number;
      name: string;
      count?: number;
    }>;
    runes?: unknown;
  };
  players: Array<{
    riotId: string;
    champion: string;
    team: string;
    level: number;
    position?: string;
    scores?: {
      kills: number;
      deaths: number;
      assists: number;
      creepScore: number;
    };
    items: Array<{
      itemId: number;
      name: string;
      count?: number;
    }>;
  }>;
  recentEvents: Array<{
    eventName: string;
    eventTime: number;
    data?: Record<string, unknown>;
  }>;
}
```

Trim events to a useful recent window so state stays compact.

## Discord behavior

Implement:

- `/join`
- `/leave`
- status/logging when OpenAI is connected/disconnected
- graceful cleanup of Discord voice and OpenAI WebSocket sessions

For the first prototype, only listen to the Discord user who invoked `/join`. Avoid mixing every participant's audio.

Use current Discord voice libraries with DAVE support. Do not hand-roll Discord's raw voice protocol.

## Audio pipeline

Implement a pragmatic working pipeline. It is fine to use `prism-media`, FFmpeg, `@discordjs/opus`, or equivalent current packages.

Conceptually:

Discord Opus 48 kHz -> PCM -> OpenAI Realtime input format

OpenAI Realtime PCM -> Discord-compatible PCM/Opus -> Discord playback

Keep transcoding isolated behind small adapter classes so audio implementation can be swapped later.

## Configuration

Provide `.env.example` with at least:

```bash
OPENAI_API_KEY=
DISCORD_BOT_TOKEN=
DISCORD_CLIENT_ID=
DISCORD_GUILD_ID=
CONTROL_PLANE_SHARED_SECRET=
CONTROL_PLANE_URL=
```

If SST/Cloudflare requires additional environment variables, document them.

## SST / Cloudflare

Use the current SST version and current Cloudflare provider/components rather than copying old SST v2 examples.

The infrastructure should deploy the Worker/control plane and any Durable Object needed for session state.

Prefer the simplest viable setup. No database unless clearly required.

## Repository structure

Prefer a pnpm workspace:

```text
league-voice-coach/
  apps/
    discord-bot/
    league-companion/
    control-plane/
  packages/
    shared/
  sst.config.ts
  package.json
  pnpm-workspace.yaml
  .env.example
  README.md
```

## Developer experience

I want to be able to roughly do:

```bash
pnpm install
pnpm dev
```

and have clear instructions for:

1. creating a Discord application/bot
2. inviting it with the required permissions
3. setting secrets
4. starting SST/Cloudflare control plane
5. starting the local League companion
6. starting the Discord bot
7. joining a voice channel and running `/join`

Include useful scripts for each app.

## Implementation quality

- Keep it small and understandable.
- Use strict TypeScript.
- No `any` unless interacting with an unavoidable third-party boundary.
- Add lightweight tests for normalization/protocol code where useful.
- Add structured logs around Discord, Riot polling, companion WebSocket, OpenAI Realtime events, and tool calls.
- Never log secret values.
- Handle reconnects for OpenAI and the companion WebSocket.
- Fail gracefully when League is not currently running.

## Non-goals

Do not build yet:

- accounts/auth UI
- payments
- matchmaking
- multi-user voice mixing
- production observability stack
- long-term game history
- vector database/RAG
- web dashboard

## Riot policy note

This is an experimental personal prototype. Add a clear note to the README that real-time prescriptive coaching may conflict with Riot's developer/product policies and must be reviewed before public/commercial distribution.

Do not attempt to bypass Riot restrictions or anti-cheat. Only use documented/local game data already exposed by the client.

## Definition of done

The prototype is done when:

1. League is running in a live game.
2. The local companion automatically discovers the player's Riot ID and publishes current game state.
3. I join a Discord voice channel and execute `/join`.
4. The bot joins the channel.
5. I can say, "What item should I build now?"
6. The Realtime model calls `get_current_game_state`.
7. The tool returns live Riot state.
8. The bot answers me aloud in Discord with a concise recommendation.
9. `/leave` cleanly disconnects everything.

Start by implementing the full vertical slice. Prefer a working simple prototype over abstractions or premature production hardening.
