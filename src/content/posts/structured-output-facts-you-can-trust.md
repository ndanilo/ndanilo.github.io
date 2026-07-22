---
title: "Structured output in LangChain.js: designing facts the rest of the pipeline can trust"
description: "How Zod schemas and providerStrategy turn LLM responses into a typed contract for a LangGraph quiz pipeline — starting with fact extraction."
publishDate: 2026-07-21
tags: ["langchain", "langgraph", "structured-output", "typescript", "zod"]
draft: false
---

I'm building a quiz generator with LangGraph.js. Before the graph can invent questions or distractors, it needs something more basic: a short list of atomic facts pulled from a source text.

This post is about that first stage only. The quiz product is not finished. What *is* finished enough to teach is the pattern underneath it — **structured output** — and why schema design is part of the product, not a boring type detail.

The code lives in [`02-langchain-quiz-generator`](https://github.com/ndanilo/langchain-ai/tree/master/02-langchain-quiz-generator) in my repo.

## What structured output is

With a normal chat call, the model returns prose. You hope it looks like JSON. You parse it. Sometimes it works. Sometimes it wraps the array in an extra object, invents a field, or returns markdown fences around the payload.

**Structured output** means you give the model a schema and ask the runtime to return data that matches it. In LangChain.js that shows up as a typed `structuredResponse` on the agent result — not a string you have to second-guess.

In this project the schema is a Zod description of “one to ten quiz-worthy facts.” The service asks the model for that shape. Downstream code can treat the result as data.

## Why it exists here

A quiz pipeline is a chain of consumers:

1. Extract facts from long text
2. (Later) turn facts into questions
3. (Later) generate options, score, and so on

Each step only works if the previous step produced a **stable shape**. Free-form paragraphs are fine for a chatbot. They are a poor input for the next graph node.

Structured output exists so the LLM and the rest of your TypeScript agree on a contract before you write the second node.

## When to use it — and when not to

Use structured output when:

- Another piece of code will read the result (a LangGraph node, a database write, an API response)
- You care about field names, enums, lengths, or required fields
- Failures should be detectable (validation failed) rather than silent (“looks mostly right”)

Skip it — or keep it light — when:

- The user is reading the answer directly and open-ended prose is the product
- You are exploring prompts and the shape is still changing every hour
- A single free-text field is enough and a heavy schema adds noise

Opinionated take: if a later function will `map` over the result, give it a schema. If a human will skim it once, prose is fine.

## The schema is the product decision

Here is the fact model from the project:

```ts
export const fact = z.object({
  fact: z.string().min(1),
  importance: z.enum(["low", "medium", "high"]),
});

/** Root-level fact list used by structured-output extraction. */
export const factsSchema = z
  .array(fact)
  .min(1)
  .max(10)
  .describe("All distinct atomic facts extracted from the text");
```

A few choices are doing real work:

- **Root-level array** — the model must return `[{ fact, importance }, ...]`, not `{ facts: [...] }`. That is stricter, and deliberate. Wrapping objects are a common LLM habit; the schema rejects them so the contract stays simple for the next node.
- **`min(1).max(10)`** — empty lists are useless for a quiz. Huge lists drown the later stages. The bounds encode product limits in types.
- **`importance` enum** — not required for extraction alone, but useful later when you prioritize which facts become questions. The schema is already thinking one step ahead.

> **Common mistake:** Treating the schema as documentation for humans while still parsing `JSON.parse(content)` by hand. If the schema does not drive the call (`providerStrategy(schema)` or equivalent), you do not have structured output — you have hope.

There is a unit test that rejects the wrapped shape on purpose:

```ts
it("rejects a wrapped object shape", () => {
  assert.throws(() =>
    factsSchema.parse({
      facts: [{ fact: "Only one", importance: "high" }],
    }),
  );
});
```

That test is not pedantry. It locks the decision so a “helpful” model change does not silently break the pipeline.

## Wiring it through the service

The service builds a no-tools agent and passes the schema via `providerStrategy`:

```ts
async generateStructuredOutputAsync<T>(
  systemPrompt: string,
  userPrompt: string,
  schema: z.ZodSchema<T>,
) {
  try {
    const agent = createAgent({
      model: this.llmClient,
      tools: [],
      responseFormat: providerStrategy(schema),
    });

    const messages = [
      new SystemMessage(systemPrompt),
      new HumanMessage(userPrompt),
    ];

    const result = await agent.invoke({ messages });
    return {
      success: true,
      data: result.structuredResponse,
      error: null,
    };
  } catch (error) {
    // ... return { success: false, data: null, error }
  }
}
```

What matters in that block:

- **`responseFormat: providerStrategy(schema)`** — this is the structured-output hook. The provider-facing strategy uses the Zod schema so the model is steered toward a valid payload and the result lands on `structuredResponse`.
- **Generic `T`** — callers pass `factsSchema` and get `Facts` back. The same method can serve other schemas later without a new service class.
- **Error as data** — failures return `{ success: false, error }` instead of throwing. Graph nodes can turn that into an `AIMessage` and keep control flow predictable. That is a trade-off: you must check `success` every time, but you avoid try/catch sprawl at every node.

> **Tip:** Inject a `BaseChatModel` in the constructor for tests. Production uses `ChatOpenAI` (here via OpenRouter). Tests use LangChain’s `fakeModel` and never hit the network.

## The LangGraph node that consumes the contract

`extractFact` is a factory: pass an `LLMService` (or take the default), get a node function.

```ts
export function extractFact(llmService: LLMService = defaultLlmService) {
  return async (state: GraphAnnotation): Promise<GraphAnnotation> => {
    const systemPrompt = generateSystemPrompt();
    const userPrompt = state.messages.at(-1)!.text;

    const result = await llmService.generateStructuredOutputAsync<Facts>(
      systemPrompt,
      userPrompt,
      factsSchema,
    );

    if (!result.success) {
      return {
        ...state,
        messages: [new AIMessage("error: " + result.error)],
      };
    }

    return {
      ...state,
      messages: [new AIMessage(JSON.stringify(result.data))],
    };
  };
}
```

Step by step:

1. Read the latest user message as the source text.
2. Call the service with the shared system prompt and `factsSchema`.
3. On failure, put an error string on the message list so the graph still returns something inspectable.
4. On success, stringify the typed facts into an `AIMessage`.

Stringifying into a message is a temporary bridge while the graph is still message-centric. A later version may keep `facts` as a first-class state field. For now the important part is that **validation already happened** before that stringify — the JSON is a serialization of trusted data, not the validation step itself.

The graph wiring is intentionally small: `START → extractFact → END`. One node, one responsibility.

## The prompt matches the schema

The system prompt is a JSON object of instructions, not a long essay:

```ts
export const generateSystemPrompt = () => {
  return JSON.stringify({
    role: "you extract atomic quiz-worthy facts from user text.",
    instructions: [
      "Extract as many distinct atomic facts as possible, between 5 and 10",
      "Each fact must be a separate, standalone statement",
      "Return every fact you find as a separate item in the list — do not summarize into a single fact.",
    ],
  });
};
```

Prompt and schema should agree. If the prompt says “return a markdown table” while the schema wants an array of objects, you are fighting yourself. Here both say: several atomic facts, list-shaped.

## Common mistakes

- **Wrapped arrays** — `{ "facts": [...] }` feels natural in English; this schema rejects it. Align the prompt and the Zod root type.
- **Schema as comments only** — defining Zod types but parsing raw message content by hand.
- **Unbounded lists** — without `max`, one verbose model run can flood every later node.
- **Throwing through the graph** — fine in scripts; awkward in multi-node workflows. This service prefers an explicit failure object so the node can decide how to surface it.
- **Testing only the happy path with a live key** — slow, flaky, and expensive. The contract is exactly what unit tests are for.

## How I verify the contract without calling an LLM

Structured output is only useful if you trust it. The project’s unit tests inject `fakeModel` responses whose JSON matches the schema, then assert on `LLMService` and `extractFact`.

```ts
it("appends extracted facts as an AI message", async () => {
  const facts: Facts = [
    { fact: "48 teams compete.", importance: "high" },
    { fact: "Three host nations.", importance: "high" },
  ];
  const model = createFactsModel(facts);
  const node = extractFact(new LLMService(model));

  const next = await node({
    messages: [new HumanMessage("World Cup article text")],
  });

  assert.equal(next.messages[0]?.content, JSON.stringify(facts));
});
```

You are not testing “is the model smart?” You are testing “does my node honor the structured contract when the model returns valid data — and when it fails?”

That is enough for this article: testing supports the lesson; it is not the whole lesson.

## Key takeaway

Structured output is a **contract between the model and the next engineering step**. In this quiz pipeline, the contract is a root-level array of facts with importance labels. Zod encodes the product rules; `providerStrategy` enforces them at call time; the LangGraph node consumes the result as data.

Design the schema for the consumer you have not written yet. That is usually more valuable than polishing the prompt alone.

## What’s next

When the next stage lands — turning these facts into questions — that will be a separate design problem, and a separate post.
