<div align="center">

# 🌤️ Weather Chat

### Mastra + Next.js 16 + AI SDK v7

A streaming AI chat agent with real-time weather tool-calling, built on the Mastra agent framework and Next.js App Router.

![Next.js](https://img.shields.io/badge/Next.js-16-000000?style=for-the-badge&logo=nextdotjs)
![Mastra](https://img.shields.io/badge/Mastra-latest-6E56CF?style=for-the-badge)
![AI SDK](https://img.shields.io/badge/AI%20SDK-v7-000000?style=for-the-badge)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?style=for-the-badge&logo=typescript&logoColor=white)

</div>

---

## 📦 Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router, Turbopack) |
| Agent runtime | Mastra |
| Streaming | AI SDK v7 (`@ai-sdk/react`) |
| Storage | LibSQL (local file or Turso) |
| Logging | Pino (`@mastra/loggers`) |
| Validation | Zod |

---

## 🚀 Quick Start

### 1. Scaffold Next.js 16

```bash
npx create-next-app@latest weather-chat --yes --ts --eslint --tailwind --src-dir --app --turbopack --no-react-compiler --no-import-alias
cd weather-chat
```

### 2. Initialize Mastra

```bash
npx mastra@latest init
```

This scaffolds a `src/mastra` folder with an example weather agent:

```
src/mastra/
├── index.ts                  # Mastra config, including memory
├── tools/
│   └── weather-tool.ts       # Fetches weather for a given location
└── agents/
    └── weather-agent.ts      # Weather agent that uses the tool
```

You'll call `weather-agent.ts` from your Next.js API routes.

> 💡 Pick your model provider when prompted, and drop your API key into the terminal — or skip it and add it to `.env` later.

### 3. Install dependencies

```bash
npm install @mastra/core@latest @mastra/loggers@latest @mastra/libsql@latest @mastra/ai-sdk@latest @ai-sdk/react@latest ai@latest zod@latest
```

### 4. Configure environment variables

Create a `.env` file in the project root:

```bash
OPENAI_API_KEY=sk-...
```

---

## 🛠️ Build the Agent

### Weather tool

**`src/mastra/tools/weather-tool.ts`**

```ts
import { createTool } from '@mastra/core/tools'
import { z } from 'zod'

export const weatherTool = createTool({
  id: 'get-weather',
  description: 'Get the current weather for a location',
  inputSchema: z.object({
    location: z.string().describe('City name'),
  }),
  outputSchema: z.object({
    location: z.string(),
    temperatureCelsius: z.number(),
    conditions: z.string(),
    humidity: z.number(),
    windKph: z.number(),
  }),
  execute: async ({ location }) => {
    // Geocode
    const geoRes = await fetch(
      `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(location)}&count=1`
    )
    const geo = await geoRes.json()

    if (!geo.results?.length) {
      throw new Error(`Location not found: ${location}`)
    }

    const { latitude, longitude, name } = geo.results[0]

    // Current weather
    const weatherRes = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code`
    )
    const weather = await weatherRes.json()
    const c = weather.current

    const conditionsMap: Record<number, string> = {
      0: 'Clear sky', 1: 'Mainly clear', 2: 'Partly cloudy', 3: 'Overcast',
      45: 'Fog', 61: 'Light rain', 63: 'Rain', 65: 'Heavy rain',
      71: 'Snow', 80: 'Rain showers', 95: 'Thunderstorm',
    }

    return {
      location: name,
      temperatureCelsius: c.temperature_2m,
      conditions: conditionsMap[c.weather_code] ?? 'Unknown',
      humidity: c.relative_humidity_2m,
      windKph: c.wind_speed_10m,
    }
  },
})
```

### Weather agent

**`src/mastra/agents/weather-agent.ts`**

```ts
import { Agent } from '@mastra/core/agent'
import { weatherTool } from '../tools/weather-tool.ts'

export const weatherAgent = new Agent({
  id: 'weather-agent',
  name: 'Weather Agent',
  instructions: `
    You are a helpful weather assistant that provides accurate weather information.
    When responding:
    - Include temperature, conditions, humidity, and wind
    - Keep responses concise and friendly
    - If no location is given, ask for one
    Use the weatherTool to fetch current weather data.
  `,
  model: 'openai/gpt-5.6-sol',
  tools: { weatherTool },
})
```

### Mastra entry point

**`src/mastra/index.ts`**

```ts
import { Mastra } from '@mastra/core/mastra'
import { PinoLogger } from '@mastra/loggers'
import { LibSQLStore } from '@mastra/libsql'
import { weatherAgent } from './agents/weather-agent.ts'

export const mastra = new Mastra({
  agents: { weatherAgent },
  storage: new LibSQLStore({
    id: 'mastra-storage',
    url: process.env.TURSO_DATABASE_URL ?? 'file:./mastra.db',
    authToken: process.env.TURSO_AUTH_TOKEN,
  }),
  logger: new PinoLogger({ name: 'Mastra', level: 'info' }),
})
```

---

## 🔌 Wire Up the API Route

**`src/app/api/chat/route.ts`**

```ts
import { handleChatStream } from '@mastra/ai-sdk'
import { createUIMessageStreamResponse } from 'ai'
import { mastra } from '@/mastra'

const THREAD_ID = 'weather-chat-thread'
const RESOURCE_ID = 'weather-chat-user'

export async function POST(req: Request) {
  const params = await req.json()

  const stream = await handleChatStream({
    mastra,
    agentId: 'weather-agent',
    version: 'v7',
    params: {
      ...params,
      memory: { ...params.memory, thread: THREAD_ID, resource: RESOURCE_ID },
    },
  })

  return createUIMessageStreamResponse({ stream })
}
```

Then build your frontend chat UI (e.g. with `useChat` from `@ai-sdk/react`) and point it at `/api/chat`.

---

## ▶️ Run It

```bash
npm run dev
```

Open **http://localhost:3000** to chat with your weather agent.

> 🧪 Optionally, run `npx mastra dev` in a second terminal to launch **Mastra Studio** — an interactive playground for your agents — at **http://localhost:4111**.

---

## 📁 Final Project Structure

```
weather-chat/
├── src/
│   ├── app/
│   │   └── api/
│   │       └── chat/
│   │           └── route.ts
│   └── mastra/
│       ├── index.ts
│       ├── tools/
│       │   └── weather-tool.ts
│       └── agents/
│           └── weather-agent.ts
├── .env
└── package.json
```

---

<div align="center">

Built with [Mastra](https://mastra.ai) · [Next.js](https://nextjs.org) · [AI SDK](https://sdk.vercel.ai)

</div>
