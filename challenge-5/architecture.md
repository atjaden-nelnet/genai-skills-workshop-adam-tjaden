# Challenge 5 — ADS Online Agent: Architecture

Solution architecture for the Alaska Department of Snow (ADS) online agent. A citizen asks a
question through a web form; the agent screens it, answers it from the ADS knowledge base (RAG)
or live weather data, screens the answer, logs everything, and returns a grounded reply.

## Diagram

```mermaid
flowchart TD
    user([Citizen / Web browser])

    subgraph agent["ADS Online Agent (Jupyter notebook)"]
        ui["ipywidgets web form<br/>(text box + Submit)"]
        turn["secure_agent_turn()"]
        armor_in["Model Armor<br/>prompt filter"]
        router{"Intent router<br/>weather or knowledge?"}
        rag_call["answer_knowledge()<br/>Gemini + RAG grounding"]
        wx_call["answer_weather()<br/>Gemini summarizes forecast"]
        armor_out["Model Armor<br/>response filter"]
    end

    subgraph gcp["Google Cloud"]
        datastore[("Vertex AI Search<br/>data store — ADS FAQs")]
        gcs[("Cloud Storage<br/>gs://labs.roitraining.com/<br/>alaska-dept-of-snow")]
        gemini["Gemini on Vertex AI<br/>(google-genai SDK)"]
        logging[("Cloud Logging<br/>ads-agent audit log")]
    end

    nws["NWS API<br/>api.weather.gov"]

    user -->|question| ui --> turn
    turn --> armor_in -->|safe| router
    armor_in -->|blocked| armor_out
    router -->|knowledge| rag_call
    router -->|weather| wx_call
    rag_call -. grounded retrieval .-> datastore
    rag_call --> gemini
    wx_call -->|lat/long| nws
    wx_call --> gemini
    rag_call --> armor_out
    wx_call --> armor_out
    armor_out -->|safe answer| ui --> user

    gcs -->|import docs| datastore
    turn -. logs prompt + verdicts + response .-> logging
```

## Components

| Component | Service | Role |
|---|---|---|
| Web form | `ipywidgets` (in notebook) | "Deployed" UI — text box + Submit button |
| Orchestration | `secure_agent_turn()` | Filter → answer → filter → log pipeline |
| Prompt / response filtering | **Model Armor** (2 templates) | Blocks injection, jailbreaks, unsafe content |
| Knowledge answers | **Vertex AI Search** + Gemini | RAG grounded on ADS FAQ documents |
| Weather answers | **NWS API** + Gemini | Live forecasts via `api.weather.gov` |
| Knowledge source | **Cloud Storage** → data store | 50 ADS FAQ `.txt` docs imported in code |
| Audit log | **Cloud Logging** (`ads-agent`) | Records every prompt, filter verdict, and response |
| Generation | **Gemini** on Vertex AI | `gemini-2.5-flash` via the `google-genai` SDK |
| Quality | **Gen AI Evaluation Service** | Scores grounded answers (`EvalTask`) |

## Request flow

1. Citizen submits a question in the web form.
2. The prompt is **logged**, then screened by **Model Armor** (prompt template). If blocked, a safe
   refusal is returned.
3. An **intent router** decides whether the question is about weather or ADS knowledge.
   - **Knowledge** → Gemini answers, **grounded on the Vertex AI Search data store**.
   - **Weather** → the **NWS API** is called for the location, and Gemini summarizes the forecast.
4. The answer is screened by **Model Armor** (response template). If blocked, a safe refusal is
   returned.
5. The response is **logged** and shown to the citizen.
