# Implementation

## Writes and provenance

A fact records its category, project, source kind, evidence date, and version. A replacement names the current fact and supplies its expected version. PostgreSQL locks the candidate row while the service checks that it is visible to the caller, still current, and eligible for replacement. Closing the old fact, inserting its successor, and linking the chain take place in one transaction. Competing writes against the same version cannot both succeed. A conflict lets callers refresh the fact before deciding whether their update still applies.

This schematic transaction omits SQL and supporting index maintenance. The operations represent the sequence in the private implementation.

```python
async def replace_fact(store, fact_id, expected_version, change, actor):
    async with store.transaction():
        previous = await store.lock_visible_fact(fact_id, actor)
        if previous.is_closed or previous.version != expected_version:
            raise ValueError("Refresh the fact before replacing it")
        await store.check_replacement_allowed(previous, change, actor)

        await store.close(previous, change.effective_at)
        successor = await store.insert_successor(previous, change)
        await store.link_replacement(previous, successor)
        return successor
```

## Two time axes

- **Effective time** asks what was true at a point in the world.
- **Recorded time** asks what the service understood at a point in its own history.

For example, suppose a synthetic project changes its review cadence from weekly to fortnightly on May 3 and an agent records the change on May 6. An effective-time query for May 4 can show the new cadence after the update arrives. A recorded-time query for May 4 still shows the weekly cadence. Routine expiry can remove an item from a current task packet while keeping its history available to authorized queries.

## Retrieval and access

Nenrin combines ranked candidates from PostgreSQL full-text search and, when enabled, pgvector similarity. Full-text retrieval uses PostgreSQL's ts_rank_cd over normalized tokens, with segmentation for Chinese text. Reciprocal rank fusion combines ordinal ranks from the two channels. Project aliases contribute routing signals and a configurable reranker can refine the leading results. Visibility and temporal predicates constrain the candidate set. The context builder filters expired items for current queries and assembles summaries and selected facts within an estimated token budget.

The snippet shows the rank fusion term shared by both candidate streams. Routing bonuses and reranking are separate steps.

```python
def reciprocal_rank_score(ranks, k=60):
    return sum(1 / (k + rank) for rank in ranks)


assert reciprocal_rank_score([2, 3]) > reciprocal_rank_score([1])
```

The MCP layer resolves client credentials and applies scope and visibility policy. Private projects stay outside default retrieval. Explicit reads depend on the caller's permissions and the kind of record. Linked document bodies use a storage adapter, while the service keeps metadata and fact links in PostgreSQL. The storage path includes staging and retry logic so a temporary upload failure can be recovered.
