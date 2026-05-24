# Technical Decisions

## RAG instead of fine-tuning

Fine-tuning would make sense if the target behavior were stable and the dataset were large enough. This system needs fresh context: recent projects, updated pricing rules, team capacity, and reference architectures. RAG keeps that context editable and inspectable.

## Tool use instead of a single prompt

A single large prompt is hard to reason about and easy to break. Tool use gives the model explicit ways to retrieve pricing rules, similar projects, and architecture references. It also makes failures easier to isolate.

## Deterministic pricing engine

Pricing is business-critical and must be explainable. The LLM produces structured scope and risk signals; code calculates effort, margin, taxes, risk buffer, and price ranges.

## PostgreSQL with pgvector

At the current scale, a separate vector database would add operational overhead without enough benefit. PostgreSQL handles relational data, auditability, and vector search in one place.
