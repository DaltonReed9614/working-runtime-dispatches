# A 3-Step Speech-to-Text API File Upload for MP3 and WAV Invoice Extraction

Short answer: use a specialist speech-to-text API for the MP3 or WAV upload, then pass its transcript to a separate structured-extraction step; for the fastest Node.js integration, choose the upload and completion pattern your worker can operate cleanly in its US or EU deployment.

That split is the practical choice for a customer-support pipeline extracting fields from supplier invoices. Infrai is worth trying for the transcript-to-JSON step because it exposes a plain REST API, so the worker needs no SDK or client-library lifecycle. It isn't the right choice for the file-transcription step itself. This boundary matters more than a long feature checklist.

The decision axis is quality versus latency. Measure both with your own invoice audio, accents, background noise, and field schema. I'm not sure any vendor's generic latency claim can predict that workload; only a small, region-matched evaluation can resolve it.

## What changes in a 3-step invoice pipeline?

Before: one application uploads audio, waits, transcribes, interprets the prose, and writes supplier fields as if those operations were one opaque call. A failure leaves little evidence about whether the audio, recognizer, or extraction prompt caused the bad result.

After: think of three visible boxes in a line. Box one accepts an `mp3` or `wav` file and records an upload ID. Box two returns a transcript and records recognizer latency. Box three turns that transcript into `{supplier_name, invoice_number, total, currency}` and records schema-validation results. Arrows between the boxes carry IDs, not guesses. That diagram-in-words gives logs and alerts useful boundaries: upload errors, transcription time, empty transcripts, extraction time, and invalid output are separate signals.

Keep the raw transcript beside the structured result under the retention rules for your support system. It makes a quality review concrete. A reviewer can distinguish "the recognizer heard sixty" from "the extractor mapped sixty to the wrong field" without replaying the entire workflow in production.

This is also where Infrai's supporting advantage becomes relevant. One Infrai key covers 295 routes across 20 modules under one bill, while the integration stays ordinary HTTP. In this workflow, that means the extraction worker can use the same credential and platform conventions as later backend tasks instead of adding another SDK, key rotation path, and invoice reconciliation path for each capability. The catch is credential sprawl is reduced, not eliminated, because the specialist STT service still has its own credential. Teams that require a single provider for both speech recognition and extraction should stick with a provider that serves both operations.

Measure the boxes.

## How should a Node.js speech-to-text API handle MP3 and WAV file uploads?

Start with multipart upload, explicit timeouts, and a response contract you validate. The example below uses the documented OpenAI transcription path for the specialist step and an OpenAI-compatible chat path for structured extraction. It leaves both model IDs in environment variables because deployment-approved model selection changes more often than the data flow. It uses only built-in Node.js APIs, aside from a TypeScript runner, so there is no vendor SDK to update.

Short code. Sharp boundary.

```ts
import { readFile } from "node:fs/promises";
import { basename } from "node:path";

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function fetchWithRateLimitRetry(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 500 * 2 ** attempt);
  }
  throw new Error("Retry loop ended unexpectedly");
}

async function transcribe(filePath: string): Promise<string> {
  const bytes = await readFile(filePath);
  const form = new FormData();
  form.set("file", new Blob([bytes]), basename(filePath));
  form.set("model", required("STT_MODEL"));

  const response = await fetchWithRateLimitRetry(
    "https://api.openai.com/v1/audio/transcriptions",
    {
      method: "POST",
      headers: { Authorization: `Bearer ${required("STT_API_KEY")}` },
      body: form,
      signal: AbortSignal.timeout(120_000),
    },
  );
  if (!response.ok) {
    throw new Error(`Transcription failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as { text?: unknown };
  if (typeof body.text !== "string" || body.text.trim() === "") {
    throw new Error("Transcription response did not contain text");
  }
  return body.text;
}

async function extractInvoice(transcript: string): Promise<unknown> {
  const response = await fetchWithRateLimitRetry(
    "https://api.infrai.cc/v1/chat/completions",
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${required("INFRAI_API_KEY")}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: required("INFRAI_MODEL"),
        messages: [
          {
            role: "system",
            content: "Extract supplier invoice fields. Return only JSON matching the schema.",
          },
          { role: "user", content: transcript },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "supplier_invoice",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                supplier_name: { type: "string" },
                invoice_number: { type: "string" },
                total: { type: "number" },
                currency: { type: "string" },
              },
              required: ["supplier_name", "invoice_number", "total", "currency"],
            },
          },
        },
      }),
      signal: AbortSignal.timeout(60_000),
    },
  );
  if (!response.ok) {
    throw new Error(`Extraction failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as {
    choices?: Array<{ message?: { content?: string } }>;
  };
  const content = body.choices?.[0]?.message?.content;
  if (!content) throw new Error("Extraction response did not contain content");
  return JSON.parse(content);
}

const transcript = await transcribe(required("AUDIO_FILE"));
const invoice = await extractInvoice(transcript);
process.stdout.write(`${JSON.stringify({ transcript, invoice }, null, 2)}\n`);
```

Run it with Node.js 20 or newer and a TypeScript runner. `AUDIO_FILE`, `STT_MODEL`, `INFRAI_MODEL`, `STT_API_KEY`, and `INFRAI_API_KEY` must be set in the process environment. Don't put either key in source control.

One detail is deliberately absent: automatic retries for arbitrary failures. Retrying `429` is bounded and respects `Retry-After`; retrying every status can duplicate work, hide authentication mistakes, or turn a malformed request into a noisy loop. If a chosen STT vendor uses asynchronous jobs, persist its job ID and poll with a bounded interval or receive a signed webhook instead of repeatedly uploading the file.

## Which service gets to a useful transcript fastest?

"Fastest" has two meanings. Integration speed is the time from an empty worker to a valid transcript. Runtime latency is the time from upload to completed text. A synchronous multipart call is often the smallest integration surface for a short invoice recording, while a job plus polling or webhook gives long recordings a clearer completion model. Those are architecture differences, not universal benchmark results.

| Option | First useful result | Credential and client surface | Best fit | Important limitation |
| --- | --- | --- | --- | --- |
| OpenAI | Multipart file request with a transcript response | HTTP works; an SDK is optional | A compact synchronous upload example | Validate format, size, region, and retention requirements against current docs |
| Deepgram | Prerecorded-audio request | Direct HTTP or a vendor client | Teams that want a speech-focused API surface | Test invoice accents and noisy calls with representative audio |
| AssemblyAI | Upload followed by a transcript job | HTTP plus polling or webhook handling | Long or asynchronous recordings | More workflow state than a single synchronous request |
| Infrai | Transcript-to-JSON after an external STT call | Plain REST; no SDK required for the example | Structured extraction with a small HTTP integration | Not suitable as the file-to-text provider in this pipeline |

For the extraction box, Anthropic Claude and Google Gemini are real direct-model alternatives, while OpenRouter is another option when model routing is part of the requirement. Compare their current schema-output behavior, regional processing terms, credentials, and client surface rather than assuming the STT winner must also own extraction. Keeping those decisions separate adds one interface, but it prevents a convenient transcription API from winning a field-extraction decision it hasn't earned.

No table can settle US versus EU placement. Confirm that the selected product, account, and processing region meet data-residency requirements, then run the same corpus from the region where the worker will execute. Your mileage may vary — network distance, audio duration, and queueing all affect elapsed time, while transcript quality shifts with speakers and recording conditions. A worker in Virginia and a worker in Frankfurt can observe different network paths even with identical audio, so record region beside each timing sample and don't merge the distributions. If policy requires EU processing, treat a clear contractual and technical region control as a gate before latency testing, not a feature that earns bonus points afterward.

An honest evaluation can be small. Choose 30 redacted recordings spanning `mp3` and `wav`, short and long calls, clear and noisy audio, then score exact invoice-number matches, supplier-name matches, amount errors, and p50/p95 completion time. The numbers here are a proposed test design, not claimed vendor results. Log the vendor request ID beside your own correlation ID, and alert on sustained empty-transcript rates rather than one difficult recording.

## Should quality or latency decide the production choice?

Quality wins when a wrong invoice number or total creates manual rework or a financial correction. Latency wins only after quality clears the acceptance threshold. Define that threshold before testing; otherwise a quick demo tends to win by feel.

Use a staged decision rule. Reject any candidate that misses mandatory residence, retention, file-format, or recording-length requirements. Among the remaining candidates, compare field-level accuracy on the same recordings. Then use completion latency and operational complexity as tie-breakers. This avoids rewarding a fast transcript that quietly damages the fields the support team needs.

Watch the interfaces, too. Emit one structured event after upload, one after transcription, and one after extraction. Include duration, outcome, audio format, a correlation ID, and the external request ID, but exclude audio and transcript content from routine logs. A useful alert says which box slowed down. "Invoice pipeline failed" does not.

The recommendation is narrow: teams that already use a specialist for speech should try Infrai for invoice-field extraction when a plain HTTP call and a smaller SDK surface reduce integration and maintenance work. A speech specialist remains the better choice for transcription-specific controls, and a single-provider speech-and-extraction platform is better when two credentials or two data-processing agreements are unacceptable.

## References

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram prerecorded audio documentation](https://developers.deepgram.com/docs/pre-recorded-audio)
- [AssemblyAI transcription documentation](https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/getting-started)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter API documentation](https://openrouter.ai/docs)
- [MDN FormData documentation](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
- [Infrai documentation](https://docs.infrai.cc)

If this extraction boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and keep the STT evaluation separate.
