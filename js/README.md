# LangChainPlus Client SDK

This package contains the TypeScript client for interacting with the [LangChainPlus platform](https://www.langchain.plus/).

To install:

```bash
yarn add langchainplus-sdk
```

LangChainPlus helps you and your team develop and evaluate language models and intelligent agents. It is compatible with any LLM Application and provides seamless integration with [LangChain](https://github.com/hwchase17/langchainjs), a widely recognized open-source framework that simplifies the process for developers to create powerful language model applications.

> **Note**: You can enjoy the benefits of LangChainPlus without using the LangChain open-source packages! To get started with your own proprietary framework, set up your account and then skip to [Logging Traces Outside LangChain](#logging-traces-outside-langchain).

A typical workflow looks like:

1. Set up an account with LangChainPlus or host your [local server](https://docs.langchain.plus/docs/getting-started/local_installation).
2. Log traces.
3. Debug, Create Datasets, and Evaluate Runs.

We'll walk through these steps in more detail below.

## 1. Connect to LangChainPlus

Sign up for [LangChainPlus](https://www.langchain.plus/) using your GitHub, Discord accounts, or an email address and password. If you sign up with an email, make sure to verify your email address before logging in.

Then, create a unique API key on the [Settings Page](https://www.langchain.plus/settings), which is found in the menu at the top right corner of the page.

Note: Save the API Key in a secure location. It will not be shown again.

## 2. Log Traces

You can log traces natively in your LangChain application or using a LangChainPlus RunTree.

### Logging Traces with LangChain

LangChainPlus seamlessly integrates with the JavaScript LangChain library to record traces from your LLM applications.

```bash
yarn add langchain
```

1. **Copy the environment variables from the Settings Page and add them to your application.**

Tracing can be activated by setting the following environment variables or by manually specifying the LangChainTracer.

```typescript
process.env["LANGCHAIN_TRACING_V2"] = "true";
process.env["LANGCHAIN_ENDPOINT"] = "https://api.langchain.plus"; // or your own server
process.env["LANGCHAIN_API_KEY"] = "<YOUR-LANGCHAINPLUS-API-KEY>";
// process.env["LANGCHAIN_SESSION"] = "My Session Name"; // Optional: "default" is used if not set
```

> **Tip:** Sessions are groups of traces. All runs are logged to a session. If not specified, the session is set to `default`.

2. **Run an Agent, Chain, or Language Model in LangChain**

If the environment variables are correctly set, your application will automatically connect to the LangChainPlus platform.

```typescript
import { ChatOpenAI } from "langchain/chat_models/openai";

const chat = new ChatOpenAI({ temperature: 0 });
const response = await chat.predict(
  "Translate this sentence from English to French. I love programming."
);
console.log(response);
```

### Logging Traces Outside LangChain

_Note: this API is experimental and may change in the future_

You can still use the LangChainPlus development platform without depending on any
LangChain code. You can connect either by setting the appropriate environment variables,
or by directly specifying the connection information in the RunTree.

1. **Copy the environment variables from the Settings Page and add them to your application.**

```typescript
process.env["LANGCHAIN_ENDPOINT"] = "https://api.langchain.plus"; // or your own server
process.env["LANGCHAIN_API_KEY"] = "<YOUR-LANGCHAINPLUS-API-KEY>";
// process.env["LANGCHAIN_SESSION"] = "My Session Name"; // Optional: "default" is used if not set
```

2. **Log traces using a RunTree.**

A RunTree tracks your application. Each RunTree object is required to have a name and run_type. These and other important attributes are as follows:

- `name`: `string` - used to identify the component's purpose
- `run_type`: `string` - Currently one of "llm", "chain" or "tool"; more options will be added in the future
- `inputs`: `Record<string, any>` - the inputs to the component
- `outputs`: `Optional<Record<string, any>>` - the (optional) returned values from the component
- `error`: `Optional<string>` - Any error messages that may have arisen during the call

```typescript
import { RunTree, RunTreeConfig } from "langchainplus-sdk";

const parentRunConfig: RunTreeConfig = {
  name: "My Chat Bot",
  run_type: "chain",
  inputs: {
    text: "Summarize this morning's meetings.",
  },
  serialized: {}, // Serialized representation of this chain
  // session_name: "Defaults to the LANGCHAIN_SESSION env var"
  // apiUrl: "Defaults to the LANGCHAIN_ENDPOINT env var"
  // apiKey: "Defaults to the LANGCHAIN_API_KEY env var"
};

const parentRun = new RunTree(parentRunConfig);

const childLlmRun = await parentRun.createChild({
  name: "My Proprietary LLM",
  run_type: "llm",
  inputs: {
    prompts: [
      "You are an AI Assistant. The time is XYZ." +
        " Summarize this morning's meetings.",
    ],
  },
});

await childLlmRun.end({
  outputs: {
    generations: [
      "I should use the transcript_loader tool" +
        " to fetch meeting_transcripts from XYZ",
    ],
  },
});

const childToolRun = await parentRun.createChild({
  name: "transcript_loader",
  run_type: "tool",
  inputs: {
    date: "XYZ",
    content_type: "meeting_transcripts",
  },
});

await childToolRun.end({
  outputs: {
    meetings: ["Meeting1 notes.."],
  },
});

const childChainRun = await parentRun.createChild({
  name: "Unreliable Component",
  run_type: "tool",
  inputs: {
    input: "Summarize these notes...",
  },
});

try {
  // .... the component does work
  throw new Error("Something went wrong");
} catch (e) {
  await childChainRun.end({
    error: `I errored again ${e.message}`,
  });
}

await parentRun.end({
  outputs: {
    output: ["The meeting notes are as follows:..."],
  },
});

await parentRun.postRun({
  exclude_child_runs: false,
});
```

### Create a Dataset from Existing Runs

Once your runs are stored in LangChainPlus, you can convert them into a dataset.
For this example, we will do so using the Client, but you can also do this using
the web interface, as explained in the [LangChainPlus docs](https://docs.langchain.plus/docs/).

```typescript
import { LangChainPlusClient } from "langchainplus-sdk/client";
const client = new LangChainPlusClient({
  // apiUrl: "https://api.langchain.com", // Defaults to the LANGCHAIN_ENDPOINT env var
  // apiKey: "my_api_key", // Defaults to the LANGCHAIN_API_KEY env var
  /* callerOptions: {
         maxConcurrency?: Infinity; // Maximum number of concurrent requests to make
         maxRetries?: 6; // Maximum number of retries to make
    */
});
const datasetName = "Example Dataset";
// We will only use examples from the top level AgentExecutor run here,
// and exclude runs that errored.
const runs = await client.listRuns({
  sessionName: "my_session",
  executionOrder: 1,
  error: false,
});

const dataset = await client.createDataset(datasetName, {
  description: "An example dataset",
});

for (const run of runs) {
  await client.createExample(run.inputs, run.outputs ?? {}, {
    datasetId: dataset.id,
  });
}
```

# Evaluating Runs

You can run evaluations directly using the LangChainPlus client.

```ts
import { StringEvaluator } from "langchainplus-sdk/evaluation";

function jaccardChars(output: string, answer: string): number {
  const predictionChars = new Set(output.trim().toLowerCase());
  const answerChars = new Set(answer.trim().toLowerCase());
  const intersection = [...predictionChars].filter((x) => answerChars.has(x));
  const union = new Set([...predictionChars, ...answerChars]);
  return intersection.length / union.size;
}

async function grader(config: {
  input: string;
  prediction: string;
  answer?: string;
}): Promise<{ score: number; value: string }> {
  let value: string;
  let score: number;
  if (config.answer === null || config.answer === undefined) {
    value = "AMBIGUOUS";
    score = 0.5;
  } else {
    score = jaccardChars(config.prediction, config.answer);
    value = score > 0.9 ? "CORRECT" : "INCORRECT";
  }
  return { score: score, value: value };
}

const evaluator = new StringEvaluator({
  evaluationName: "Jaccard",
  gradingFunction: grader,
});

const runs = await client.listRuns({
  sessionName: "my_session",
  executionOrder: 1,
  error: false,
});

for (const run of runs) {
  client.evaluateRun(run, evaluator);
}
```

## Additional Documentation

To learn more about the LangChainPlus platform, check out the [docs](https://docs.langchain.plus/docs/).