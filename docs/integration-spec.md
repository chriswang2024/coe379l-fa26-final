# Portfolio Assistant Integration Spec

The Python server runs on your laptop and listens at `http://127.0.0.1:8000`. When you open that address, Python sends the HTML, CSS, and JavaScript files to your browser. The browser runs the JavaScript, sends each question back to Python via `POST /api/chat`, and displays the answer along with its supporting citations. Python runs retrieval and invokes Lean on the same laptop. Python can call Gemini or another hosted model for inference (try Gemini API? free tier seems to be enough, or we can try a TACC LLM).

## 1. Corpus format · Chris → Aashvi

Use `corpus/corpus.json`, an array with one object per citable passage. Chris prepares the corpus and Aashvi decides how to retrieve from it. Example:

```json
[
  {
    "id": "project.timer.overview",
    "source": "Project Description",
    "title": "Verilog Countdown Timer",
    "section": "Overview",
    "text": "Chris built a four-digit countdown timer in Verilog. The same button starts and pauses it."
  }
]
```
Each passage should contain enough context to be understood independently, since the original passage will also be displayed when a user expands a citation in the chat UI.

## 2. Model answer format · Aashvi’s model integration → Chris’s checker

Ask the model to write natural prose in paragraphs. Each paragraph carries the chunk IDs it cites. 

```json
{
  "status": "answer",
  "paragraphs": [
    {
      "text": "Chris built a four-digit countdown timer in Verilog. It starts and pauses with the same button.",
      "citations": ["project.timer.overview"]
    }
  ]
}
```

If the retrieved passages provide no support for an answer, return `{"status":"abstain","paragraphs":[]}`. If they support part of the answer, give that part and acknowledge the gap.

## 3. Checker handoff · Aashvi’s Python server ↔ Chris’s Lean checker

**Input (Python → Lean).** Python parses the model reply, builds the request below, and passes `json.dumps(request) + "\n"` to the Lean subprocess on stdin. If the LLM's reply is malformed JSON, Python returns the error response in section 4.

- `response` is the untrusted model reply as a JSON object in the format from section 2.
- `retrieved_ids` is the trusted list of passage IDs Python sent to the model for this question. Python builds this list from its own records for each request.

```json
{
  "response": {
    "status": "answer",
    "paragraphs": [
      {
        "text": "Chris built a four-digit countdown timer in Verilog. It starts and pauses with the same button.",
        "citations": ["project.timer.overview"]
      }
    ]
  },
  "retrieved_ids": ["project.timer.overview"]
}
```

**Validation rules.** Lean uses `Json.parse` on the stdin line. It decodes `retrieved_ids` as an array of strings and calls `verify` with those IDs and the `response` JSON value. `verify` uses `fromJson?` to decode the response into Lean types before running the Boolean checker. Reject malformed JSON, missing required fields, wrong types, or a `status` other than `answer` or `abstain`. For every paragraph, `text` must be a string and `citations` must be an array of strings. Ignore extra fields.

- `answer` requires at least one paragraph. Each paragraph must contain text that is nonempty and at least one citation. Every citation must exactly match an ID in this request’s `retrieved_ids`.
- `abstain` requires `paragraphs: []`. An abstention can pass with an empty `retrieved_ids` list.

**Output (Lean → Python).** Lean's `main` uses `toJson` to encode the Boolean decision under `allowed` and prints one compact JSON object to stdout. The example above returns `{"allowed":true}`. An answer citing an ID outside this request’s `retrieved_ids` returns `{"allowed":false}`, even if the ID exists in the corpus.

Python parses stdout with `json.loads` and proceeds only if the checker exits successfully and the decoded object has `allowed` exactly `True`. It builds the browser reply from the same `response` it submitted.

## 4. Browser handoff · Chris’s UI ↔ Aashvi’s Python server

The UI sends `POST /api/chat` with `Content-Type: application/json` and a body like `{"question":"What did Chris build in Verilog?"}`.

After Lean accepts the model reply, Python returns HTTP `200` with `Content-Type: application/json`. The response contains the validated status and paragraphs, along with the supporting passages corresponding to the cited IDs.

Example response:

```json
{
  "status": "answer",
  "paragraphs": [
    {
      "text": "Chris built a four-digit countdown timer in Verilog. It starts and pauses with the same button.",
      "citations": ["project.timer.overview"]
    }
  ],
  "sources": [
    {
      "id": "project.timer.overview",
      "source": "Project Description",
      "title": "Verilog Countdown Timer",
      "section": "Overview",
      "text": "Chris built a four-digit countdown timer in Verilog. The same button starts and pauses it."
    }
  ]
}
```

Python constructs `sources` by looking up the validated citation IDs in the retrieved corpus passages. Only cited passages are included. The source, title, section, and text come directly from the corpus, not from the model.

The UI displays the paragraph text with numbered citations such as [1] or [2]. Citation numbers are assigned in order of first appearance, and repeated references to the same passage use the same number. Each citation corresponds to a source in `sources` through its ID.

The UI allows users to expand a citation to view the source, title, section, and original supporting passage. This gives visitors additional context and allows them to inspect the evidence behind an answer without requiring links to external documents.

If the model determines that there is not enough evidence to answer the question and Lean accepts the response, Python returns: 

```json
{
  "status": "abstain",
  "paragraphs": [],
  "sources": []
}
```

The UI displays “I couldn’t find enough information to answer that.”

For an invalid question, checker rejection, or service failure, Python sends an HTTP error status and puts the message directly in the response body as `text/plain; charset=utf-8`. For example, a blank question gets HTTP `400` with the body `Please enter a question.` The UI checks response.ok to tell whether the request succeeded. It reads successful responses using response.json() and error messages using response.text(). 
