# vespa-colbert-tricks

While working with the ColBERT embeddings and Vespa some tricks might be helpful.
This repo presents 2 such tricks:
1. Limit the length of docs to be embedded
2. Feed precomputed ColBERT tensors

Why? Because [colbert-embedder](https://docs.vespa.ai/en/embedding.html#colbert-embedder) is heavy
and without GPU and with large docs you might have to wait a long time until embeddings are calculated.

Disclaimer: most of the code in this repo is shamelessly copied from the [Vespa sample app](https://github.com/vespa-engine/sample-apps/tree/master/colbert-long).

## Setup

Vespa(>=8.314.57) running locally, `vespa-cli`, and `jq`.

## Limit the length of documents to be embedded

The problem: say you have many documents most of which are short Slack messages but some are long documents,
e.g. CSV file with 1000+ lines, and each line of these CSV are split into a chunk.
In general, you want all docs to be searchable, and to use ColBERT as a reranker.
But reranking on these long CSVs might not be the most useful thing.
Also, they can clog the embedder.
It might be wise not to embed those anomalously long documents.

To do that we can do some trickery with the [indexing language](https://docs.vespa.ai/en/reference/indexing-language-reference.html).
First, let's see the entire field definition here:

```text
field colbert type tensor<int8>(context{}, token{}, v[16]) {
    indexing {
        0 | set_var chunk_cnt;
        input text | for_each {
            if (true == true) {
                get_var chunk_cnt + 1 | set_var chunk_cnt;
            } else { "" }
        };
        if (get_var chunk_cnt < 3) {
            input text | embed colbert context | attribute;
        } else { "" };
    }
}
```

In the indexing block we:
1. calculate amount of chunks by iterating over the `text` array. (The silly `if (true == true)` in the `for_each` is just a trick for Vespa to not complain about type mismatches)
2. Compare the `chunk_cnt` with our magic constant of documents that are too long e.g. 3 chunks are too much to handle.
3. Call the `colbert` embedder.

Feed the sample doc:
```shell
echo '{
  "fields": {
    "text": ["one", "two"]
  },
  "put": "id:doc:doc::1"
}' | vespa feed -
```

Check if embeddings are calculated:
```shell
vespa visit --field-set 'doc:[document],colbert'
```

```json
{"id":"id:doc:doc::1","fields":{"colbert":{"type":"tensor<int8>(context{},token{},v[16])","blocks":[{"address":{"context":"0","token":"0"},"values":[-54,-71,85,123,-80,-27,64,-19,-18,-31,-88,71,-8,94,-111,-24]},{"address":{"context":"0","token":"1"},"values":[92,-72,-51,59,-101,0,64,-19,8,112,-116,1,-29,71,-128,-63]},{"address":{"context":"0","token":"2"},"values":[24,48,77,57,-101,32,66,-55,72,96,12,5,-29,71,-108,-56]},{"address":{"context":"0","token":"3"},"values":[-52,-71,85,123,-71,-91,64,-91,-23,-19,-116,5,-21,94,-112,104]},{"address":{"context":"1","token":"0"},"values":[-62,-79,85,123,80,-27,88,-67,-90,-32,-84,71,-8,-33,-87,48]},{"address":{"context":"1","token":"1"},"values":[-37,-65,76,-7,75,-27,81,-67,-126,-16,-56,0,50,63,-96,113]},{"address":{"context":"1","token":"2"},"values":[-127,61,92,-71,67,-27,81,55,-122,-4,-120,6,50,62,-96,-77]},{"address":{"context":"1","token":"3"},"values":[-55,-69,87,91,56,-27,80,-75,-88,-28,-84,71,-14,126,-104,57]}]},"text":["one","two"]}}
```

See that `colbert` field contains a tensor.
The chunk count is under our threshold of 2.

Overwrite the doc:
```shell
echo '{
  "fields": {
    "text": ["one", "two", "three"]
  },
  "put": "id:doc:doc::1"
}' | vespa feed -
```

Check if embeddings are calculated:
```shell
vespa visit --field-set 'doc:[document],colbert'
```

The output should be:
```json
{"id":"id:doc:doc::1","fields":{"text":["one","two","three"]}}
```
See that there is no `colbert` field.
It is so because there were too many chunks.

NOTE: until the [issue](https://github.com/vespa-engine/vespa/issues/29690) with 
`if`s in the `for_each` is solved it is not possible 
(or at least I can't come up with a solution) to 
limit the amount of chunks to be embedded per doc in the indexing language, e.g.
embed only the first 10 chunks. Of course, a custom document processor could do this job.

NOTE: the same trick works for other Vespa embedders.

UPDATE: as of Vespa 8.307.19 the above trick fails when an empty list is provided, e.g.:
```shell
echo '{
  "fields": {
    "text": []
  },
  "put": "id:doc:doc::1"
}' | vespa feed -
```
The progress on this bug is tracked [here](https://github.com/vespa-engine/vespa/issues/30512).

## Feed precomputed ColBERT tensors

If you want to transfer a dataset that contains embeddings between Vespa deployments, e.g. from production to your local laptop.
The trick is:
1. Define a **document** field of the same type as the ColBERT **synthetic** field, e.g. `colbert2`.
2. While visiting export also the colbert tensor.
3. The main `colbert2` field makes an attribute either from the document field if provided
or calculates the embeddings.

Simplified schema with all the other details removed:
```text
schema doc {
    document doc {
        field text type array<string> {}
        field colbert_doc_field type tensor<int8>(context{}, token{}, v[16]) {}
    }
    field colbert2 type tensor<int8>(context{}, token{}, v[16]) {
       indexing: ((input colbert_doc_field) || (input text | embed colbert context)) | attribute
    }
}
```

Add a sample document to Vespa:
```shell
echo '{
  "fields": {
    "text": ["one", "two", "three"]
  },
  "put": "id:doc:doc::1"
}' | vespa feed -
```

Check if embedding done its job.

```shell
vespa visit --field-set 'doc:[document],colbert2'
```
The field set deciphering is:
- `doc` is for the document type
- `[document]` is one of the standard field sets
- `,colbert` also asks to return the synthetic field named `colbert`

This should return a bunch of numbers under `colbert2`:
```json
{"id":"id:doc:doc::1","fields":{"colbert2":{"type":"tensor<int8>(context{},token{},v[16])","blocks":[{"address":{"context":"0","token":"0"},"values":[-54,-71,85,123,-80,-27,64,-19,-18,-31,-88,71,-8,94,-111,-24]},{"address":{"context":"0","token":"1"},"values":[92,-72,-51,59,-101,0,64,-19,8,112,-116,1,-29,71,-128,-63]},{"address":{"context":"0","token":"2"},"values":[24,48,77,57,-101,32,66,-55,72,96,12,5,-29,71,-108,-56]},{"address":{"context":"0","token":"3"},"values":[-52,-71,85,123,-71,-91,64,-91,-23,-19,-116,5,-21,94,-112,104]},{"address":{"context":"1","token":"0"},"values":[-62,-79,85,123,80,-27,88,-67,-90,-32,-84,71,-8,-33,-87,48]},{"address":{"context":"1","token":"1"},"values":[-37,-65,76,-7,75,-27,81,-67,-126,-16,-56,0,50,63,-96,113]},{"address":{"context":"1","token":"2"},"values":[-127,61,92,-71,67,-27,81,55,-122,-4,-120,6,50,62,-96,-77]},{"address":{"context":"1","token":"3"},"values":[-55,-69,87,91,56,-27,80,-75,-88,-28,-84,71,-14,126,-104,57]},{"address":{"context":"2","token":"0"},"values":[-30,-72,69,123,-19,-92,96,-65,-11,-128,-84,79,-39,-35,-85,48]},{"address":{"context":"2","token":"1"},"values":[-6,112,-60,123,79,33,65,-69,-106,-36,-116,8,-7,109,-77,96]},{"address":{"context":"2","token":"2"},"values":[-8,114,-44,-23,-49,33,65,-85,-73,-36,12,9,-37,-19,-109,0]},{"address":{"context":"2","token":"3"},"values":[-24,-70,71,91,-3,37,64,-65,-15,-124,-84,15,-7,-35,-101,48]}]},"text":["one","two","three"]}}
```

Visit the document, transform the JSON, and store it to a file:
```shell
vespa visit --field-set 'doc:[document],colbert2' \
| jq -c '
  .fields.colbert_doc_field = .fields.colbert2
  | del(.fields.colbert2)
  | del(.fields.text)
' > doc1.jsonl
```
Transformations here are 2:
1. Rename field from to
2. Remove `text` so that no inference should happen.

Delete the document
```shell
vespa document remove "id:doc:doc::1"
```

There should be 0 docs in the index:
```shell
vespa query 'select * from sources * where true'
```

Feed it back:
```shell
vespa feed doc1.jsonl 
```

Check if the ColBERT tensor is in place.
```shell
vespa visit --field-set 'doc:[document],colbert2'
```

See that the `colbert2` field contains a tensor from our `doc1.jsonl` file.

Voila!

NOTE: the ColBERT tensors are compressed by the embedder.
Uncompressed also works the same way. 

UPDATE: there is even a cleaner way to express the above idea: the [`select_input`]() statement, e.g.:

```text
field colbert3 type tensor<int8>(context{}, token{}, v[16]) {
    indexing {
        select_input {
            colbert_doc_field: input colbert_doc_field | attribute;
            text: input text | embed colbert context | attribute;
        }
    }
}
```

One downside it has that only one indexing expression can be specified in each branch.

## Bonus

The 2 tricks described above can be combined,
i.e. embedding short docs unless the tensor is provided in the document field.
The actual implementation is left as an exercise for the dear reader.
