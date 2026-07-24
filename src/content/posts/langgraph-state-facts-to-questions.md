---
title: "LangGraph state between stages: turning facts into questions"
description: "How typed graph state carries facts and source text into a second structured-output call — without stuffing JSON through chat messages."
publishDate: 2026-07-24
tags: ["langchain", "langgraph", "typescript", "zod"]
draft: false
demoVideo: "/posts/langgraph-state-facts-to-questions/facts-to-questions-pipeline.mp4"
demoVideoTitle: "LangGraph Studio run: initial message, extracted facts, then generated questions"
---

In the [first stage of this quiz pipeline](/posts/structured-output-facts-you-can-trust/), structured output gave me a list of facts the rest of the code could trust. That solved the model contract. It did not solve the *graph* contract.

The next problem is quieter: once `extractFact` succeeds, how does `generateQuestions` receive those facts — plus the original source text — without reading the last chat message and hoping it is still JSON?

This post is about that wire: **LangGraph state as the typed handoff between stages**. The quiz product is still unfinished. What is finished enough to teach is a two-node pipeline that reuses one LLM service, two Zod contracts, and shared state fields instead of message archaeology.

The code lives in [`02-langchain-quiz-generator`](https://github.com/ndanilo/langchain-ai/tree/master/02-langchain-quiz-generator).

## Why messages alone were not enough

Chat-style `messages` are useful for prompts and for tooling that expects a transcript. They are a weak place to store intermediate *data*.

If stage one appends `JSON.stringify(facts)` as an `AIMessage`, stage two has to:

1. Find the right message
2. Parse it again
3. Hope nothing else overwrote or interleaved the payload

That works in a demo. It breaks the moment you care about grounding (`sourceText`), errors (`success` / `errorMessage`), or a second structured list (`questions`) that should not compete with chat history for the same channel.

Opinionated take: use `messages` for conversation. Use named state fields for anything another node will `map` over.

## State is the wire

The graph schema is not a changelog of fields I added this week. It is the product shape of the pipeline right now:

```ts
export const graphAnnotation = z.object({
  messages: withLangGraph(z.custom<BaseMessage[]>(), MessagesZodMeta),
  success: z.boolean().default(false),
  errorMessage: z.string().default(""),
  sourceText: z.string().default(""),
  facts: z.array(fact).default(() => []),
  questions: z.array(question).default(() => []),
});
```

A few fields do the teaching:

- **`sourceText`** — the original article, kept immutable for grounding later. Question generation can prefer facts for *what* to test and source text for *how* names and numbers were phrased.
- **`facts` / `questions`** — typed lists nodes read and write directly.
- **`success` / `errorMessage`** — a cheap signal so the next stage can refuse to spend another LLM call when extraction already failed.

There are two different Zod jobs in the same file, and mixing them up hurts:

| Role | Examples | Rules |
| --- | --- | --- |
| **LLM contract** | `factsSchema`, `questionsSchema` | Strict `.min(1).max(10)` — empty output is a failed generation |
| **Graph channel** | `facts`, `questions` on `graphAnnotation` | `.default(() => [])` — empty is valid *before* a node runs |

The model must return a non-empty list. The graph must be allowed to start with none. Same item shape (`fact`, `question`); different tolerance at the edges.

## Two nodes, one linear graph

Composition is deliberately boring:

```ts
const workflow = new StateGraph({
  stateSchema: graphAnnotation,
})
  .addNode("extractFact", extractFact())
  .addNode("generateQuestions", generateQuestions())
  .addEdge(START, "extractFact")
  .addEdge("extractFact", "generateQuestions")
  .addEdge("generateQuestions", END);
```

```text
START → extractFact → generateQuestions → END
```

Nodes return `Partial<GraphAnnotation>`. LangGraph merges the update. You do not pass a tuple from one function to the next; you write fields the next reader already knows how to find.

`extractFact` still takes the user article from the last message (that is the invoke input), then writes `facts`, `sourceText`, and `success: true`. From there, data travels on state — not as another stringify round-trip.

## The second structured call

`generateQuestions` reuses the same `LLMService.generateStructuredOutputAsync` helper from the first post. What changes is the **schema and the prompt payload**.

The user prompt is explicit JSON: facts as the primary input, source text as grounding context only:

```ts
export const generateQuestionsUserPrompt = (
  facts: { fact: string; importance: string }[],
  sourceText: string,
) =>
  JSON.stringify({
    task: "Generate multiple-choice questions from the facts below.",
    facts,
    sourceText,
  });
```

The node reads those fields from state, not from `messages.at(-1)`:

```ts
const systemPrompt = generateQuestionsSystemPrompt();
const userPrompt = generateQuestionsUserPrompt(state.facts, state.sourceText);

const result = await llmService.generateStructuredOutputAsync<Questions>(
  systemPrompt,
  userPrompt,
  questionsSchema,
);
```

On success it writes `questions` (and marks `success: true`). Same gateway pattern as fact extraction; different contract: a root-level array of question objects with a stem and options.

Here is that handoff in LangGraph Studio — initial message in, `facts` filled by the first node, then `questions` written by the second:

<!-- demo-video -->

## Fail fast between stages

Before any second LLM call, the node checks whether stage one actually produced something usable:

```ts
if (!state.success || !state.facts.length) {
  const errorMessage =
    state.errorMessage || "No facts to generate questions from";
  return {
    ...state,
    success: false,
    errorMessage: errorMessage,
    messages: [new AIMessage(errorMessage)],
  };
}
```

That is not clever routing. It is cost control and debuggability. If extraction failed, question generation should not invent a quiz from an empty list and call it a success.

Unit tests cover the happy path, LLM failure, and these short-circuits — still with `fakeModel`, still without burning API tokens.

## Prompts are not contracts

Here is the honest gap in the current design.

The system prompt says each question has **exactly four options**, one correct and three distractors. The Zod `question` schema says:

```ts
export const question = z.object({
  question: z.string().min(1),
  options: z.array(z.string()).min(1).max(4),
});
```

No `correctIndex`. Options may be one to four strings. The prompt wishes for a graded multiple-choice item; the schema only guarantees a stem and some options.

That drift is useful as a lesson: **structured output only enforces what you put in the schema**. Instructions outside the schema are hope. Until the contract can name the correct answer, the pipeline can *generate* quiz-shaped JSON, but it cannot *score* a learner.

I am leaving that gap visible on purpose. Fixing it is the next product decision, not a prompt tweak.

## Key takeaway

Between LangGraph stages, the valuable artifact is not another chat message. It is **named state**: `sourceText` and `facts` in, `questions` out, with `success` as a gate.

Reuse one structured-output gateway. Give each stage its own schema and prompt. Keep LLM contracts strict and graph defaults permissive. And when the prompt promises a correct answer the schema cannot store, treat that as unfinished design — not as something the model will “just know.”

## What’s next

Answer keys and grading need a field the runtime can validate — something like `correctIndex` — plus a consumer that checks a learner’s choice against it. That is a separate schema decision, and a separate post.
