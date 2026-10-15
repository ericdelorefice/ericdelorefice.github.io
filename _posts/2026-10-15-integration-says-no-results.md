---
layout: post
title: "Your Integration Says “No Results.” It Means “I Failed.”"
date: 2026-10-15
---

*How a caught exception turns into a confident wrong answer in RAG and agent pipelines, and the test that catches it*

Ask an assistant “when is the launch?” and imagine it answers, “I couldn’t find anything about that in your documents.” That sounds like a normal answer, and on the surface it is. The documents might not mention a launch. But in the run I am about to show you, the document is sitting right there in the store. The store just timed out; however, the code in the middle turned “I could not look” into “there is nothing to find.”

Nothing crashed. A warning went to a log nobody was reading, and the user got a believable answer that was wrong.

I have spent the last few weeks finding this exact bug in open-source AI and data infrastructure and sending fixes upstream. Four of those fixes are the examples in this article, plus one from another contributor, and all five are merged. The pattern is small, it is everywhere, and it gets through code review easily. So I want to walk through what it looks like, why it hides so well, and how you can write the test that catches it.

## The pattern

Here is the whole thing in six lines. This is a retriever sitting between an agent and a vector store:

```python
def retrieve(store, query):
    try:
        return store.search(query)
    except Exception as e:
        log.warning("search failed, returning no results: %s", e)
        return []
```

The intent behind it is reasonable. Someone did not want one flaky call to take the whole app down, so they caught the error and returned an empty list. The problem is that an empty list already means something. It means “I looked and there was nothing.” Now it also means “I could not look,” and the code above it has no way to tell the two apart.

To see what that does, I wrote a small fake store that can be empty, healthy, or down, and a one-line agent that answers from whatever comes back:

```python
def answer(question, docs):
    if not docs:
        return "I couldn't find anything about that in your documents."
    return "Based on your documents: " + " / ".join(docs)

store = FakeVectorStore(docs=["the Q3 roadmap moves launch to May"], down=True)
print(answer("when is the launch?", retrieve(store, "when is the launch")))
```

Running it prints:

```
search failed, returning no results: vector store did not answer within 5s
I couldn't find anything about that in your documents.
```

The answer is about a document that exists. If you put a real LLM in place of my one-liner, it gets worse, not better, because a model handed an empty context will still write a confident, fluent answer.

A crash would honestly have been better here. A crash gets noticed; a wrong answer that looks right does not.

![Two paths from the same query: a store that answers with nothing, and a store that fails, both reaching the user as "no results"]({{ '/assets/integration-no-results-diagram.png' | relative_url }})

*Image by author*

## Why it gets past review and tests

I think there are four reasons this survives, and none of them come down to anyone being careless.

**“No results” is a valid answer.** An empty list is a normal return value, so nothing downstream looks surprised by it. There is no type error, no null check that fires, nothing.

**The tests cover the happy path.** Most adapter tests check that a search finds what was inserted and that an empty store returns nothing. Both pass on the code above. Very few tests make the backend fail on purpose and check what the caller actually sees.

**There is a log line, so it feels handled.** The warning is real; however, a log line is just a note to someone who might read it later. It does not change the answer the user gets right now.

**It usually lives in one adapter, not all of them.** Libraries like these support many backends, and each adapter is written on its own. Most of them raise on a failed call. One of them catches it. If you review that one file on its own, the handler looks like a careful choice. Next to its siblings, it is the odd one out.

## Five real cases, all merged

The first four are public pull requests I opened. In every one, the maintainers reviewed the change and merged it, and I am grateful they took the time.

**semantic-router: a failed read deleted the user’s routes.** `QdrantIndex._get_all` caught a `ValueError` from its scroll loop, logged `Index likely empty`, and returned no records. The message is the code admitting it is guessing. An empty index never actually reached that handler, because the method checks that the collection exists first, and a scroll over an empty collection returns nothing without raising. So the handler only ran when the read really failed. The empty answer did not stay put either. The remote sync step took “the remote has nothing” at face value and deleted the user’s local routes to match. The fix removed the handler so the error propagates (#706, merged by jamescalam).

**Feast: failed writes were counted as written.** The Couchbase online store wrapped each write in `except Exception`, logged it, and kept going, and the progress counter still ticked for the row that failed. So a materialization job could finish normally, report every row as written, and leave the online store missing rows that nobody would go looking for. Of the other 22 online stores in Feast, 21 have no handler around their writes, and the MySQL store catches and re-raises, so a failed write reaches the caller everywhere else. The fix keeps the log line and re-raises (#6911, merged by ntkathole).

**LangChain’s Azure integrations: “these documents don’t exist.”** In the SQL Server vector store, `get_by_ids` caught a database error, logged it, and returned an empty list. Under the vector store contract, an ID missing from the result does not exist, so a dropped connection read as “none of these documents exist.” The delete path had a related problem: if the commit failed, it could still return `True` and log that rows were affected. The async delete in the same class already handled this correctly, which made it a good model for the fix (#1119, merged by beccadaniel).

**langroid: documents reported as ingested that were never stored.** `LanceDB.add_documents` wrapped the whole write in `except Exception` and only logged the error. This one did not even need a broken backend to show up. LanceDB fixes a table’s schema on the first write, so a later document with an extra metadata field gets rejected. The ingest call still returned the chunk count, and a later session never found those documents. Every other adapter in langroid let a write error through, and so did another method in the same file (#1201, merged by pchalasani).

**langroid again, a week later, from someone else.** On 13 October another contributor fixed the same pattern in langroid’s Pinecone adapter, with almost the same title (#1241). I did not write that one, and I like that I did not. It means the pattern is easy to recognize once you have seen it named.

## How to test the failure path

The fix is usually small. The test is the part that is extremely important, because the test is what keeps it fixed. Here is the approach I use every time.

**Make the backend fail on purpose.** You do not need a real outage. A fake that raises the same kind of error the real client raises is enough, and it keeps the test fast and offline. In my demo, `FakeVectorStore(down=True)` raises a `TimeoutError`, which is what a slow store looks like from the client side. For real projects I prefer the library’s own exception classes, or a small local server that returns the actual 429 or 500 response, so the error goes through the same code path it would in production.

**Assert that the call raises, not that it returns.** This is the line that catches the bug:

```python
def test_a_store_that_is_down_raises():
    store = FakeVectorStore(docs=["the Q3 roadmap moves launch to May"], down=True)
    with pytest.raises(RetrievalError):
        retrieve(store, "when is the launch")
```

**Keep controls that pass on both versions.** A failure test on its own can pass for the wrong reason. So next to it I keep two controls: an empty store still returns an empty list, and a healthy store still finds the document. Those prove the fix did not break the normal answers.

```python
def test_an_empty_store_still_returns_nothing():
    assert retrieve(FakeVectorStore(), "when is the launch") == []

def test_a_healthy_store_still_finds_documents():
    store = FakeVectorStore(docs=["the Q3 roadmap moves launch to May"])
    assert retrieve(store, "when is the launch") == ["the Q3 roadmap moves launch to May"]
```

**Run the test against the old code and watch it fail.** This is the step people skip, and it is the one that matters most. In the demo, the three tests pass with the fix. Run against the original retriever, the failure test fails and the two controls still pass:

```
FAILED test_retriever.py::test_a_store_that_is_down_raises - Failed: DID NOT RAISE
1 failed, 2 passed
```

If a new test passes on the old code too, it is not testing the bug. A check that has never gone red proves nothing.

## Fixing it

Once the test is in place, the change itself is usually a few lines. These are the rules I follow.

**Separate “not found” from “could not check.”** Catch the specific error that really means “not there,” like a missing collection or a 404, and let everything else through. A catch-all `except Exception` is what turns a timeout into an empty answer.

**Log, then raise.** Keeping the log line is fine. In a lot of cases, ending the handler with `raise` instead of `return []` is the whole fix. If you want a cleaner error for callers, wrap it, as long as the original is attached:

```python
except TimeoutError as e:
    log.warning("search failed: %s", e)
    raise RetrievalError(f"could not search the vector store: {e}") from e
```

**Check the siblings.** Look at how the other adapters, the async version, and the other methods in the same file handle the same failure. If everything else raises, the one that does not is usually the bug, and the siblings show you what the fix should look like.

**Make fallbacks visible.** Sometimes a fallback really is the right call, like trying a second search mode when the first one fails. That is fine, as long as the caller can see it happened, through a flag on the result or a different return type. A fallback the caller cannot see is just this bug with better intentions.

## A short checklist

Before you trust an integration’s error handling, I would ask these five questions:

1. What does this method return when the backend times out?
2. Is that the same value it returns when there is genuinely nothing there?
3. Does any test make the backend fail and check what the caller gets?
4. Did that test fail before the fix?
5. Do the sibling adapters and the async version agree?

If the answer to the second question is yes, you have probably found one.

The lesson I keep relearning from these fixes is the same every time, and it applies to integrations as a whole. The scary failures are not the ones that crash. They are the ones that report success.

---

*The examples come from public, merged pull requests, and the code is from a small companion repository you can run yourself: [github.com/ericdelorefice/integration-no-results-demo](https://github.com/ericdelorefice/integration-no-results-demo).*
