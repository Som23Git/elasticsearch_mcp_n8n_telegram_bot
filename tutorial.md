# Tutorial - Expense Tracker Tool with n8n + Elasticsearch Agent Builder in Elastic Cloud!

## Step 0: Create an Inference Endpoint

Before setting up Elasticsearch, you can optionally create a custom **inference endpoint** using **AWS Bedrock** if you plan to use a specific model for embeddings.

```
# embedding task
PUT _inference/text_embedding/bedrock-embeddings
 {
    "service": "amazonbedrock",
    "service_settings": {
        "access_key": "XXXXXXXXXXXXXXXXXXXXX",
        "secret_key": "XXXXXXXXXXXXXXXXXXXXXgLdX+jjuHk1o1bLMY",
        "region": "<appropriate_region>",
        "provider": "amazontitan",
        "model": "amazon.titan-embed-text-v2:0"
    }
}
```
Doc: https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-inference-put-amazonbedrock.

---

## Step 1: Set Up Your Elasticsearch Environment

Start by creating a new [**Elasticsearch Cloud**](https://cloud.elastic.co) or [**Serverless**](https://cloud.elastic.co) account using your email address.  
👉 No credit card is required for the trial.  

For this tutorial, let’s assume we’re creating a **proof-of-concept (POC)** environment to quickly check the capability of the Agent Builder.

Once your deployment is ready, create a new index named **`expenses`** with the following mappings:

```json
PUT expenses/
{
  "mappings": {
    "properties": {
      "amount": { "type": "double" },
      "attachments": {
        "type": "nested",
        "properties": {
          "hash": { "type": "keyword" },
          "linked_expense_ids": { "type": "keyword" },
          "type": { "type": "keyword" },
          "url": { "type": "keyword" }
        }
      },
      "audio_hash": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
      },
      "category": { "type": "keyword" },
      "chat_id": { "type": "keyword" },
      "currency": { "type": "keyword" },
      "merchant": { "type": "keyword" },
      "normalized_inr": { "type": "double" },
      "note": { "type": "text", "copy_to": ["semantic_all"] },
      "payment_method": { "type": "keyword", "copy_to": ["semantic_all"] },
      "raw_transcript": { "type": "text", "copy_to": ["semantic_all"] },
      "segments": {
        "type": "nested",
        "properties": {
          "amount": { "type": "double" },
          "category": { "type": "keyword" },
          "end_ms": { "type": "integer" },
          "merchant": { "type": "keyword" },
          "payment_method": {
            "type": "text",
            "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
          },
          "start_ms": { "type": "integer" },
          "text": { "type": "text", "copy_to": ["semantic_all"] }
        }
      },
      "semantic_all": {
        "type": "semantic_text",
        "inference_id": "bedrock-embeddings",
        "model_settings": {
          "service": "amazonbedrock",
          "task_type": "text_embedding",
          "dimensions": 1024,
          "similarity": "cosine",
          "element_type": "float"
        }
      },
      "stt_confidence": { "type": "double" },
      "stt_model": { "type": "keyword" },
      "stt_provider": { "type": "keyword" },
      "ts": { "type": "date" },
      "user_id": { "type": "keyword" }
    }
  }
}
```

This creates an index with all the required fields.  
Note the `semantic_all` field — it connects your index to the **inference endpoint** you created earlier, identified as `bedrock-embeddings`:

```json
"semantic_all": {
  "type": "semantic_text",
  "inference_id": "bedrock-embeddings",
  "model_settings": {
    "service": "amazonbedrock",
    "task_type": "text_embedding",
    "dimensions": 1024,
    "similarity": "cosine",
    "element_type": "float"
  }
}
```

---

## Step 2: Ingest Sample Data

Now, let’s add a sample document to verify ingestion:

```json
POST expenses/_doc/1
{
  "normalized_inr": 4500,
  "note": "Burger purchase",
  "amount": 4500,
  "user_id": "567876545",
  "merchant": "McDonald's",
  "currency": "INR",
  "category": "food",
  "raw_transcript": "Can you add my expense on McDonald's burger yesterday which is 25th of September which was Thursday and I paid via my credit card and I spent around 4500.",
  "payment_method": "credit_card",
  "chat_id": "45678456784567-iyn678",
  "ts": "2025-09-25T12:00:00+05:30"
}
```
Documentation:
- indexing a document: https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-index
- (optional) bulk ingestion: https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk


You can now view the document in **Discover**. The `semantic_all` field automatically includes all text fields for embedding:

```json
{
  "_index": "expenses",
  "_id": "1",
  "_source": {
    "normalized_inr": 4500,
    "note": "Burger purchase",
    "amount": 4500,
    "user_id": "567876545",
    "merchant": "McDonald's",
    "currency": "INR",
    "category": "food",
    "raw_transcript": "Can you add my expense on McDonald's burger yesterday which is 25th of September which was Thursday and I paid via my credit card and I spent around 4500.",
    "payment_method": "credit_card",
    "chat_id": "45678456784567-iyn678",
    "ts": "2025-09-25T12:00:00+05:30"
  },
  "fields": {
    "semantic_all": [
      "Burger purchase",
      "credit_card",
      "Can you add my expense on McDonald's burger yesterday which is 25th of September which was Thursday and I paid via my credit card and I spent around 4500."
    ]
  }
}
```

---

## Step 3: Setting Up Agent Builder

By default, Elastic’s **Managed LLM (Elastic Inference Service(EIS))** automatically generates `ES|QL` queries for you and retrieves results from your documents.

Take a look at what is [Elastic Inference Service](https://www.elastic.co/docs/explore-analyze/elastic-inference/eis "EIS documentation for Elastic").

If you’d like more control, you can create **custom agents** — defining your own tools, `ES|QL` queries, parameters, and logic — but for now, the default `Elastic AI agent` is sufficient.

You can already start asking natural-language questions about your `expenses` index, even with just one document.

---

## Step 4: Integrate with n8n

Next, set up your **n8n workflow**.

You can import the provided JSON object and connect the nodes using your credentials:

- **STT (Speech-to-Text)** – Use *Sarvam AI* or *AWS Transcribe*.  
- **Telegram Bot** – Add your Telegram API Token.  
- **AWS Bedrock** – Add your access key and secret key for model access.  
- **Elasticsearch (MCP Server)** – Use your endpoint credentials.  
- **Elasticsearch (Basic Auth)** – Use an API key for bulk ingestion.

Once everything is connected, **save** your workflow and **execute** it directly from your Telegram bot.

> 💡 **Tip:** Restrict bot responses to your own Telegram `user_id` to prevent others from accidentally triggering or accessing your data.

---

If you encounter any issues or edge cases, feel free to open an issue in the GitHub repository — contributions and feedback are always welcome!
