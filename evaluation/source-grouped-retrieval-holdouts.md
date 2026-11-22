# Keep the same source out of both sides of a retrieval evaluation

Two different questions can point to the same learning resource. If one question lands in a development set and the other lands in a holdout set, the two sets can look independent even though they test the same source. Group labeled questions by source before assigning either split.

## What does the split measure?

Start by deciding what you want to hold out. If the goal is to test new questions about familiar sources, reuse of a source can be appropriate. If the goal is to test retrieval behavior for sources that did not influence tuning, keep every labeled question for each source in one split. Both evaluations can use the same searchable corpus. The split controls which labeled examples are used to tune and score.

This distinction matters when testing query rewrites, reranking weights, chunk sizes, or thresholds. A question about the required inputs of a prompt and a question about its output example may share an answer page. Putting one on each side can reward a setting tuned specifically for that page.

## How do you assign a source key?

Use a stable resource ID from your catalog when you have one. Otherwise, start with the URL and a small alias table of observed equivalences. Review each alias with the source maintainer or inspect both returned pages. Preserve path case and query parameters by default. A filter query can change the result set, and a redirect can lead to a different document from the one you expected.

The example below intentionally keeps query values and anchors in the key. If your grading treats all sections of one document as one source, use its resource ID for those rows. Keep the original URL beside the group key so you can explain why a row moved.

## Can you reproduce the failure with a small fixture?

This example uses Python's standard library. The URLs use the reserved example domain. The redirect alias is supplied as fixture data, so the code makes no network requests.

```python
from urllib.parse import urlsplit, urlunsplit

aliases = {
    "https://example.org/prompt-brief-old": "https://example.org/prompt-brief",
}


def source_key(raw_url):
    url = urlsplit(raw_url)
    key = urlunsplit(
        (url.scheme.lower(), url.netloc.lower(), url.path, url.query, url.fragment)
    )
    return aliases.get(key, key)


rows = [
    ("What inputs does the brief require?", "https://example.org/prompt-brief"),
    ("Where are the expected outputs?", "https://example.org/prompt-brief-old"),
    ("How is relevance scored?", "https://example.org/scoring"),
    ("Where is the reranking checklist?", "https://example.org/reranking"),
    ("Which labels are examples?", "https://example.org/scoring?view=examples"),
]


def assert_disjoint_sources(dev, holdout):
    dev_keys = {source_key(url) for _, url in dev}
    holdout_keys = {source_key(url) for _, url in holdout}
    overlap = sorted(dev_keys & holdout_keys)
    if overlap:
        raise ValueError("Sources appear in both splits: " + ", ".join(overlap))


try:
    assert_disjoint_sources(rows[::2], rows[1::2])
except ValueError as error:
    print(error)
else:
    raise AssertionError("The fixture must detect an overlapping source")

holdout_keys = {"https://example.org/prompt-brief", "https://example.org/reranking"}
dev, holdout = [], []
for question, url in rows:
    split = holdout if source_key(url) in holdout_keys else dev
    split.append((question, url))

assert_disjoint_sources(dev, holdout)
assert len(dev) == 2 and len(holdout) == 3
print(f"Disjoint source groups: {len(dev)} dev rows, {len(holdout)} holdout rows")
```

The first check prints the overlapping source. The second completes with two development rows and three holdout rows. The scoring page and its example view remain separate because the fixture gives them separate labels. If a later review finds both URLs serve the same evidence, add an explicit alias and rebuild the split.

## What should you save for the next evaluation?

Save the source key, original URL, assigned split, and reason for any alias. For a larger dataset, group first, assign groups using a fixed random seed, then inspect source types and difficulty across the groups. Run the overlap check before each scoring job. If the types are badly imbalanced, revise the split before looking at holdout scores and record the reason.

Direct URL matching cannot detect copies hosted at different addresses. If mirrored material is common in the corpus, include the reviewed mirror relationships in the grouping manifest. When a score changes, check whether coverage or split membership changed before attributing the difference to the retrieval configuration.

## References

- [Python documentation for URL parsing](https://docs.python.org/3/library/urllib.parse.html).
- [Group-based splitting in scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupShuffleSplit.html).

Contributor note: Jake Tao operates [Resource Planet's AI and programming catalog](https://resources.packscena.com/categories/ai/), where resources can be browsed by topic. The example above is synthetic. AI assisted in drafting and checking this tutorial.
