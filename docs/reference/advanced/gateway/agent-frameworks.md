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

Each snippet assumes you have installed the packages imported in that snippet. If you are starting from one of
the telemetry guides, keep that guide's installation step and only swap the model/provider configuration.

## OpenAI SDK (TypeScript)

Use this as the baseline shape for any framework that accepts a standard OpenAI-compatible client:

```typescript title="openai-gateway.mts"
import OpenAI from 'openai';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const client = new OpenAI({
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
});

const response = await client.chat.completions.create({
  model: 'gpt-5.4-mini',
  messages: [{ role: 'user', content: 'What is the weather in London?' }],
});

console.log(response.choices[0]?.message.content);
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

[Mastra](https://mastra.ai/) can use the same AI SDK OpenAI-compatible provider as the model for an agent:

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

## LangChain.js / LangGraph.js

[LangChain.js](https://js.langchain.com/) and [LangGraph.js](https://langchain-ai.github.io/langgraphjs/) can
use the gateway by configuring `ChatOpenAI` with the gateway API key and OpenAI-compatible base URL:

```typescript title="langchain-js-gateway.mts"
import { ChatOpenAI } from '@langchain/openai';
import { HumanMessage } from '@langchain/core/messages';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const model = new ChatOpenAI({
  model: 'gpt-5.4-mini',
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  temperature: 0,
  configuration: {
    baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
  },
});

const response = await model.invoke([new HumanMessage('What is the weather in London?')]);
console.log(response.content);
```

For a LangGraph agent, pass the same `model` into `createReactAgent({ llm: model, tools: [...] })`.

## OpenAI Agents SDK (TypeScript)

The [OpenAI Agents SDK for TypeScript](https://openai.github.io/openai-agents-js/) can use an
`OpenAIProvider` configured with the gateway base URL. The example below uses Chat Completions mode so the
provider calls the gateway's OpenAI-compatible chat route.

```typescript title="openai-agents-ts-gateway.mts"
import { Agent, OpenAIProvider, Runner } from '@openai/agents';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const provider = new OpenAIProvider({
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
  useResponses: false,
});

const runner = new Runner({ modelProvider: provider });
const agent = new Agent({
  name: 'Weather Agent',
  instructions: 'You are a concise weather assistant.',
  model: 'gpt-5.4-mini',
});

const result = await runner.run(agent, 'What is the weather in London?');
console.log(result.finalOutput);
```

## VoltAgent

[VoltAgent](https://voltagent.dev/) accepts the AI SDK OpenAI-compatible provider as the agent model:

```typescript title="voltagent-gateway.mts"
import { createOpenAI } from '@ai-sdk/openai';
import { Agent } from '@voltagent/core';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const gateway = createOpenAI({
  apiKey: env.LOGFIRE_GATEWAY_API_KEY,
  baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
});

const agent = new Agent({
  name: 'weather-agent',
  instructions: 'A helpful assistant that answers questions.',
  model: gateway('gpt-5.4-mini'),
});

const response = await agent.generateText('What is the weather in London?');
console.log(response.text);
```

## LlamaIndex.TS

[LlamaIndex.TS](https://developers.llamaindex.ai/typescript) can route its OpenAI LLM through the gateway by
setting `apiKey` and `baseURL` on the `openai()` provider:

```typescript title="llamaindex-ts-gateway.mts"
import { openai } from '@llamaindex/openai';
import { agent } from '@llamaindex/workflow';
import { z } from 'zod';

const envSchema = z.object({
  LOGFIRE_GATEWAY_API_KEY: z.string(),
});

const env = envSchema.parse(process.env);

const myAgent = agent({
  llm: openai({
    model: 'gpt-5.4-mini',
    apiKey: env.LOGFIRE_GATEWAY_API_KEY,
    baseURL: 'https://gateway-us.pydantic.dev/proxy/openai',
  }),
  tools: [],
});

const result = await myAgent.run('What is the weather in London?');
console.log(result.data.result);
```

## Agno

[Agno](https://docs.agno.com/) can use the gateway through its OpenAI chat model:

```python title="agno-gateway.py"
import os

from agno.agent import Agent
from agno.models.openai import OpenAIChat

agent = Agent(
    name='Weather Agent',
    model=OpenAIChat(
        id='gpt-5.4-mini',
        api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
        base_url='https://gateway-us.pydantic.dev/proxy/openai',
    ),
)

agent.print_response('What is the weather in London?')
```

## AutoGen

[AutoGen](https://microsoft.github.io/autogen/) can use the gateway by configuring its OpenAI chat completion
client with the gateway API key and base URL:

```python title="autogen-gateway.py"
import asyncio
import os

from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient


async def main() -> None:
    model_client = OpenAIChatCompletionClient(
        model='gpt-5.4-mini',
        api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
        base_url='https://gateway-us.pydantic.dev/proxy/openai',
    )
    try:
        agent = AssistantAgent(
            'weather_agent',
            model_client=model_client,
            system_message='You are a concise weather assistant.',
        )
        response = await agent.run(task='What is the weather in London?')
        print(response.messages[-1].content)
    finally:
        await model_client.close()


asyncio.run(main())
```

## CrewAI

[CrewAI](https://docs.crewai.com/) uses LiteLLM-style provider strings. Point the OpenAI provider at the
gateway before creating the crew:

```python title="crewai-gateway.py"
import os

from crewai import Agent, Crew, Task

os.environ['OPENAI_API_KEY'] = os.environ['LOGFIRE_GATEWAY_API_KEY']
os.environ['OPENAI_API_BASE'] = 'https://gateway-us.pydantic.dev/proxy/openai'

researcher = Agent(
    role='Weather assistant',
    goal='Answer weather questions concisely.',
    backstory='You give short, direct answers.',
    llm='openai/gpt-5.4-mini',
)

task = Task(
    description='What is the weather in London?',
    expected_output='A concise answer.',
    agent=researcher,
)

crew = Crew(agents=[researcher], tasks=[task])
print(crew.kickoff())
```

## Haystack

[Haystack](https://haystack.deepset.ai/) can use the gateway with `OpenAIChatGenerator`:

```python title="haystack-gateway.py"
import os

from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage
from haystack.utils import Secret

generator = OpenAIChatGenerator(
    model='gpt-5.4-mini',
    api_key=Secret.from_token(os.environ['LOGFIRE_GATEWAY_API_KEY']),
    api_base_url='https://gateway-us.pydantic.dev/proxy/openai',
)

response = generator.run(messages=[ChatMessage.from_user('What is the weather in London?')])
print(response['replies'][0].text)
```

## Instructor

[Instructor](https://python.useinstructor.com/) wraps a standard OpenAI client, so configure that client for the
gateway first:

```python title="instructor-gateway.py"
import os

import instructor
from openai import OpenAI
from pydantic import BaseModel


class WeatherAnswer(BaseModel):
    location: str
    answer: str


client = instructor.from_openai(
    OpenAI(
        api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
        base_url='https://gateway-us.pydantic.dev/proxy/openai',
    )
)

answer = client.chat.completions.create(
    model='gpt-5.4-mini',
    response_model=WeatherAnswer,
    messages=[{'role': 'user', 'content': 'What is the weather in London?'}],
)
print(answer)
```

## LangGraph

[LangGraph](https://langchain-ai.github.io/langgraph/) uses LangChain chat models. Configure `ChatOpenAI` with
the gateway base URL, then pass it to your graph nodes:

```python title="langgraph-gateway.py"
import os

from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model='gpt-5.4-mini',
    api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
    base_url='https://gateway-us.pydantic.dev/proxy/openai',
)

response = llm.invoke('What is the weather in London?')
print(response.content)
```

## Semantic Kernel (Python)

[Semantic Kernel for Python](https://learn.microsoft.com/en-us/semantic-kernel/) can use an `AsyncOpenAI`
client configured for the gateway:

```python title="semantic-kernel-python-gateway.py"
import asyncio
import os

from openai import AsyncOpenAI
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion


async def main() -> None:
    kernel = Kernel()
    kernel.add_service(
        OpenAIChatCompletion(
            ai_model_id='gpt-5.4-mini',
            async_client=AsyncOpenAI(
                api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
                base_url='https://gateway-us.pydantic.dev/proxy/openai',
            ),
        )
    )

    response = await kernel.invoke_prompt('What is the weather in London?')
    print(response)


asyncio.run(main())
```

## smolagents

[smolagents](https://huggingface.co/docs/smolagents/) exposes the OpenAI-compatible base URL directly:

```python title="smolagents-gateway.py"
import os

from smolagents import CodeAgent, OpenAIServerModel

model = OpenAIServerModel(
    model_id='gpt-5.4-mini',
    api_base='https://gateway-us.pydantic.dev/proxy/openai',
    api_key=os.environ['LOGFIRE_GATEWAY_API_KEY'],
)
agent = CodeAgent(tools=[], model=model)

print(agent.run('What is the weather in London?'))
```

## Eino (Go)

[Eino](https://www.cloudwego.io/docs/eino/) can route its OpenAI chat model through the gateway by setting the
API key and base URL on `ChatModelConfig`:

```go title="eino-gateway.go"
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	model, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:  os.Getenv("LOGFIRE_GATEWAY_API_KEY"),
		BaseURL: "https://gateway-us.pydantic.dev/proxy/openai",
		Model:   "gpt-5.4-mini",
	})
	if err != nil {
		panic(err)
	}

	response, err := model.Generate(ctx, []*schema.Message{
		schema.UserMessage("What is the weather in London?"),
	})
	if err != nil {
		panic(err)
	}

	fmt.Println(response.Content)
}
```

## Semantic Kernel (.NET)

[Microsoft Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/) can use an OpenAI client
configured with the gateway endpoint:

```csharp title="SemanticKernelGateway.cs"
using System.ClientModel;
using Microsoft.SemanticKernel;
using OpenAI;

var gatewayClient = new OpenAIClient(
    new ApiKeyCredential(Environment.GetEnvironmentVariable("LOGFIRE_GATEWAY_API_KEY")!),
    new OpenAIClientOptions
    {
        Endpoint = new Uri("https://gateway-us.pydantic.dev/proxy/openai"),
    });

var kernel = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion("gpt-5.4-mini", gatewayClient)
    .Build();

var response = await kernel.InvokePromptAsync("What is the weather in London?");
Console.WriteLine(response);
```

## Microsoft Agent Framework (.NET)

[Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/) builds on
`Microsoft.Extensions.AI`, so the same gateway-configured OpenAI client can become the agent's chat client:

```csharp title="AgentFrameworkGateway.cs"
using System.ClientModel;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;

var gatewayClient = new OpenAIClient(
    new ApiKeyCredential(Environment.GetEnvironmentVariable("LOGFIRE_GATEWAY_API_KEY")!),
    new OpenAIClientOptions
    {
        Endpoint = new Uri("https://gateway-us.pydantic.dev/proxy/openai"),
    });

IChatClient chatClient = gatewayClient
    .GetChatClient("gpt-5.4-mini")
    .AsIChatClient();

AIAgent agent = new ChatClientAgent(
    chatClient,
    name: "WeatherAgent",
    instructions: "You are a concise, helpful assistant.");

var response = await agent.RunAsync("What is the weather in London?");
Console.WriteLine(response);
```

## Other frameworks from the agent framework PR

The original agent framework PR also added examples that are not shown as gateway snippets here:

| Framework | Why it is not shown here |
| --------- | ------------------------ |
| Genkit (Go) | The PR example uses Google's Genkit plugin, so there is no direct OpenAI-compatible gateway swap in that snippet. |
| Google ADK | The PR example uses Google's Gemini-native ADK model configuration, not an OpenAI-compatible model client. |
| Letta | The PR example configures models through the Letta server, so the gateway setting belongs in server/provider configuration rather than a small client snippet. |
| Rig (Rust) | The PR example uses Rig's OpenAI provider; add a gateway snippet only after confirming the current Rig client supports a custom OpenAI base URL. |
| Strands Agents | The PR example notes OpenAI can be used instead of Amazon Bedrock, but the exact OpenAI-compatible base URL hook should be verified against the current Strands model API before documenting it. |
