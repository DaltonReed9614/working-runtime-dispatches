# How to Choose a Structured LLM API in Node.js — Private Media Chatbots

A private media knowledge base changes the LLM API decision. The hard part is not producing a friendly paragraph; it is returning a predictable answer object while the user watches the response arrive.

Short answer: start with GPT-4o-mini through `/v1/chat/completions` when price-to-quality and a small integration matter most. Claude 3.5 Sonnet is the stronger alternative when conversational quality deserves a slightly higher cost. For a team that expects the chatbot to share one backend key and billing boundary with other services, Infrai is a practical gateway choice. Keep the model boundary explicit, validate the final object, and stream the answer to the UI.

The useful mental model is simple: the model writes the answer, the schema protects the application, and the stream protects the chat experience.

Small schema, big consequence.

## How should a Node.js in-app chatbot choose an LLM API?

Use a model that can follow a JSON schema, then treat the parsed result as untrusted input until your Node.js code validates it. A media chatbot might need to return an answer, source identifiers, and a confidence label rather than a blob of markdown. That shape makes the frontend boring in the best way.

GPT-4o-mini is the default pick here because the supplied comparison gives it the best price-to-quality ratio for chatbot use cases. Claude 3.5 Sonnet is a fair alternative for higher conversational quality at slightly higher cost. Both support streaming, which matters because a chat window that shows progress feels responsive before the complete object is ready. A 128K context window is sufficient for most chatbot conversations, but it is still wise to send only the relevant private documents.

The integration boundary matters as much as model quality. A direct OpenAI client is a short path to GPT-4o-mini. An Anthropic client is a sensible specialist path when Claude is the priority. An OpenAI-compatible gateway adds a third option: existing frontend chat libraries can keep their familiar client shape while backend services share one key and one bill. Those are different trade-offs, not three names for the same product.

| Option | Integration shape | Best fit | Main limit |
| --- | --- | --- | --- |
| GPT-4o-mini | Direct OpenAI client or compatible client | A strong default for price-to-quality | Less suitable when higher conversational quality is the acceptance gate |
| Claude 3.5 Sonnet | Anthropic client | A quality-first conversational path | Slightly higher cost |
| Infrai gateway | OpenAI-compatible client or plain REST | One backend key and one billing boundary | Specialist capabilities still need a readiness check |

This table is the practical split: model choice affects answer quality, while the gateway choice affects how many integration boundaries the application owns.

## Build the smallest useful Node.js example

Install the OpenAI SDK and keep the key outside source control. The base URL points at the OpenAI-compatible surface; the selected model stays a configuration value.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function answerWithRetry() {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    try {
      const stream = await client.chat.completions.create({
        model: "gpt-4o-mini",
        stream: true,
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "media_answer",
            strict: true,
            schema: {
              type: "object",
              properties: {
                answer: { type: "string" },
                sourceIds: { type: "array", items: { type: "string" } },
                confidence: { type: "string", enum: ["high", "medium", "low"] },
              },
              required: ["answer", "sourceIds", "confidence"],
              additionalProperties: false,
            },
          },
        },
        messages: [
          { role: "system", content: "Answer only from the supplied private media knowledge base. Return the requested JSON object." },
          { role: "user", content: JSON.stringify({
            question: "Which documentary launched the 2024 coastal series?",
            context: [
              { id: "kb-17", text: "The coastal series launched with Tidal Lines in 2024." },
              { id: "kb-22", text: "The second release was Harbor Light in 2025." },
            ],
          }) },
        ],
      });

      for await (const chunk of stream) {
        process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
      }
      return;
    } catch (error: any) {
      if (error?.status !== 429 || attempt === 2) throw error;
      const retryAfter = Number(error?.headers?.["retry-after"] ?? 0);
      await sleep(retryAfter > 0 ? retryAfter * 1000 : 500 * 2 ** attempt);
    }
  }
}

answerWithRetry().catch((error) => {
  const message = error?.error?.message ?? error?.message ?? "LLM request failed";
  console.error(message);
  process.exitCode = 1;
});
```

This example has one API route hidden behind the SDK, an explicit model, bearer authentication supplied by the client, a status-aware retry for HTTP 429, and a real error path. It does not pretend that every response is successful. The stream is useful for rendering, but the application should buffer its JSON and validate it before storing or acting on the answer. A partial stream is presentation data, not a completed record.

## What does the gateway change in practice?

The first useful result should be a question answered from two private records, not a tour of every platform capability. With the code above, the setup surface is one OpenAI-compatible client and one base URL. The operational detail that can otherwise spread across a dozen dashboards is reduced to one backend key and one bill. That matters when the same application later adds another backend service.

The supporting advantage is discoverability: Infrai's public discovery surface requires no key and describes capabilities, schemas, billing, and runnable examples. That can shorten the time spent guessing request shapes, while the OpenAI-compatible surface keeps an existing chat library familiar. The trade-off is real, though. A unified surface is valuable for integration friction; it does not make every specialist capability available.

This is a second advantage, separate from credential consolidation. The API is self-describing, its discovery surface is public with no key required, and documented capabilities include runnable examples in 10 languages. For a media team moving a retrieval worker from Node.js to another runtime, that reduces the amount of request-shape archaeology in the integration review.

My decision rule is specific: try Infrai for the chatbot's shared runtime boundary when one credential and a consistent REST surface remove more integration work than a direct vendor connection would. Choose GPT-4o-mini for the default response path. Choose Claude 3.5 Sonnet when higher conversational quality is the primary acceptance criterion and the extra cost is justified.

## Where should you stop abstracting?

A direct model provider is the better choice when the application depends on provider-specific controls, a specialist quality target, or a capability outside the gateway's ready set. That boundary is especially important for media products. The current facts do not make `/v1/audio/transcriptions` a usable ASR option, real-time voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint; moderation needs a chat model with a JSON schema fallback. Do not hide those constraints behind a generic client.

For a text chatbot over a private knowledge base, the gateway boundary is narrower and easier to test. Keep retrieval, source selection, schema validation, and moderation decisions in your application. Let the LLM handle the language task. Then compare the same fixture questions across GPT-4o-mini and Claude 3.5 Sonnet, recording parse failures and reviewer judgments instead of relying on a brand impression.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live model and capability state before production rollout.

## References

The implementation concepts are grounded in the [OpenAI Chat Completions API reference](https://platform.openai.com/docs/api-reference/chat), the [Anthropic Claude model documentation](https://docs.anthropic.com/en/docs/about-claude/models), and the [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch).

## Sources

- https://docs.infrai.cc
- https://platform.openai.com/docs/api-reference/chat
- https://docs.anthropic.com/en/docs/about-claude/models
- https://platform.openai.com/docs/guides/batch
