---
title: "Use the AI Gateway with agent frameworks"
description: "Route agent framework model calls through the Logfire AI Gateway."
---

# Use the AI Gateway with agent frameworks

Agent frameworks that accept an OpenAI-compatible provider can route large language model (LLM) calls through
the Logfire AI Gateway. Use this when you want Logfire-managed provider keys, spending limits, routing, and
gateway request telemetry without changing the rest of your agent code.

These examples only configure the model provider. They do not configure framework telemetry export; for that,
see the [agent framework telemetry guides](../../../integrations/agent-frameworks/index.md).

Set `LOGFIRE_GATEWAY_API_KEY` to a gateway API key from the Gateway **API Keys** tab. The examples use the US
gateway route and an OpenAI-compatible provider with the slug `openai`. For EU projects, use
`https://gateway-eu.pydantic.dev/proxy/openai`. If your provider or routing group uses a different slug, replace
`openai` with that slug from the **Providers** or **Routing** tab.

Install the packages used by both examples:

=== "npm"

    ```bash
    npm install ai @ai-sdk/openai @mastra/core zod
    ```

=== "pnpm"

    ```bash
    pnpm add ai @ai-sdk/openai @mastra/core zod
    ```

=== "yarn"

    ```bash
    yarn add ai @ai-sdk/openai @mastra/core zod
    ```

=== "bun"

    ```bash
    bun add ai @ai-sdk/openai @mastra/core zod
    ```

## Vercel AI SDK

The [Vercel AI SDK](https://ai-sdk.dev/) can use the gateway through its OpenAI-compatible provider:

```typescript title="vercel-ai-sdk-gateway.mts"
import { generateText, stepCountIs, tool } from 'ai';
import { createOpenAI } from '@ai-sdk/openai';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const gateway = createOpenAI({
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
});

const weatherTool = tool({
  description: 'Get the weather in a location',
  inputSchema: z.object({ location: z.string().describe('City name') }),
  outputSchema: z.object({ output: z.string() }),
  execute: async ({ location }) => ({ output: `The weather in ${location} is sunny` }),
});

const { text } = await generateText({
  model: gateway('gpt-5.4-mini'),
  prompt: 'What is the weather in London?',
  tools: { weatherTool },
  stopWhen: stepCountIs(5),
});

console.log(text);
```

## Mastra

[Mastra](https://mastra.ai/) can use the same OpenAI-compatible gateway provider as the model for an agent:

```typescript title="mastra-gateway.mts"
import { createOpenAI } from '@ai-sdk/openai';
import { Mastra } from '@mastra/core';
import { Agent } from '@mastra/core/agent';
import { createTool } from '@mastra/core/tools';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const gateway = createOpenAI({
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
});

const weatherTool = createTool({
  id: 'get-weather',
  description: 'Get current weather for a location',
  inputSchema: z.object({ location: z.string().describe('City name') }),
  outputSchema: z.object({ output: z.string() }),
  execute: async ({ location }) => ({ output: `The weather in ${location} is sunny` }),
});

const weatherAgent = new Agent({
  id: 'weather-agent',
  name: 'Weather Agent',
  instructions:
    'You are a concise weather assistant. Ask for a location if none is provided. Use the weatherTool to fetch current weather data.',
  model: gateway('gpt-5.4-mini'),
  tools: { weatherTool },
});

const mastra = new Mastra({
  agents: { weatherAgent },
});

const result = await mastra.getAgent('weatherAgent').generate('What is the weather in London?');
console.log(result.text);
```
