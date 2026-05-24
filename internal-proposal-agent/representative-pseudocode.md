# Representative Pseudocode

This is intentionally pseudocode. It shows the engineering shape of the solution without exposing production source code.

## Tool-based reasoning loop

```py
# FastAPI / service-layer pseudocode
def reply(conversation_id: str, message: str) -> AssistantReply:
    context = conversation_repo.load_recent_messages(conversation_id)
    tools = [get_pricing_rules, search_past_projects, search_reference_architectures]

    model_output = llm.respond(
        system=SYSTEM_PROMPT,
        messages=context + [message],
        tools=tools,
        response_schema=StructuredScope
    )

    if model_output.tool_call:
        result = execute_tool(model_output.tool_call)
        return llm.respond_with_tool_result(result)

    return model_output.final_answer
```

## Semantic retrieval over internal references

```py
def search_past_projects(query: str, stack: list[str]) -> list[ProjectReference]:
    embedding = embedding_model.embed(query)

    return repository.match_projects(
        query_embedding=embedding,
        stack_filter=stack,
        limit=5,
        min_similarity=0.72
    )
```

## Deterministic pricing boundary

```py
def calculate_price(scope: StructuredScope, rules: PricingRules) -> PricingBreakdown:
    base_hours = estimate_hours(scope.features, scope.integrations, scope.data_complexity)
    complexity = rules.multiplier_for(scope.risk_level)
    direct_cost = base_hours * rules.blended_hourly_cost * complexity

    return PricingBreakdown(
        direct_cost=direct_cost,
        overhead=direct_cost * rules.overhead_pct,
        risk_buffer=direct_cost * rules.risk_buffer_pct,
        target_margin=rules.target_margin_pct,
        price_range=range_p50_p90(direct_cost, rules)
    )
```
