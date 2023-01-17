# sqlite-vss - SQLite as a Vector Search Database

`sqlite-vss` (SQLite <b><u>V</u></b>ector <b><u>S</u></b>imilarity <b><u>S</u></b>earch) is a SQLite extension that brings vector search capabilities to SQLite, based on [Faiss](https://faiss.ai/) and [`sqlite-vector`](https://github.com/asg017/sqlite-vector).

Still a work in progress, not meant to be widely shared!

Once complete, you'll be able to do things like:

**Semantic Search**

```sql

/*
  Here we have a table of newspaper articles, where we want to perform
  semantic search on the  headline column. sqlie-vss is "bring your own
  vectors", so we'll need some API to generate vector embeddings from
  text (OpenAI's embbedding API, sentence-transformers, etc.).
*/
create table articles(
  headline text,
  headline_embedding blob,
  authors text
);

insert into articles
  select
    headline,
    -- OpenAI, sentence-transformers, huggingface inference, etc.
    some_embeddings_api(headline),
    authors
  from data;


/*
  Here, we're creating a vss virtual table so we can search through a
  large amount of vectors efficiently. Our embeddings examples has 384
  dimensions, and the 'with "IVF4096,PQ64"' string refers to Faiss's
  index_factory function.
*/
create virtual table vss_articles using vss_index(
  headline_embedding(384) with "IVF4096,PQ64"
);

-- The IVF index we're using requires training, so we're training it
-- with a special "operation='training'" insert.
insert into vss_articles(operation, headline_embedding)
  select
    'training',
    vector_from_blob(headline_embedding)
  from articles;

-- Now that the underlying index is trained, we can insert our vectors.
insert into vss_articles(rowid, headline_embedding)
  select
    rowid,
    vector_from_blob(headline_embedding)
  from articles;


-- Now, we can query the vector database and join results back to our original table!
with similar_matches as (
  select
    rowid,
    distance
  from vss_articles
  -- "get the 10 nearest vectors"
  where vss_search(
    headline_embedding,
    vss_search_params(
      some_embeddings_api('global warming'),
      10
    )
  )
)
select
  articles.rowid,
  similar_matches.distance
  articles.headline
from similar_matches
left join articles on articles.rowid = similar_matches.rowid;
/*
┌───────┬───────────────┬──────────────────────────────────────────────────────────────┐
│ rowid │    distance   │                           headline                           │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 980   │ 0.80562582091 │ Goal Of Capping Global Warming At 1.5 Degrees Celsius Is 'On │
│       │               │  Life Support,' UN Chief Warns                               │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 906   │ 1.0909304615  │ ‘Now Or Never’: New U.N. Report Sees Narrow Path For Avertin │
│       │               │ g Climate Catastrophe                                        │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 813   │ 1.0154189589  │ This Earth Day, Biden Faces 'Headwinds' On Climate Agenda    │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 318   │ 1.1543321607  │ Democrats' Reconciliation Package The 'Biggest Climate Actio │
│       │               │ n In Human History'                                          │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 182   │ 1.1375182154  │ Gas Prices Are Falling, But Global Events Could Cause Increa │
│       │               │ se, Energy Secretary Warns                                   │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 352   │ 1.267364358   │ Biden Takes Modest Executive Action After Climate Agenda Sta │
│       │               │ lls In Congress                                              │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 146   │ 1.2807360649  │ Pakistan Flooding Deaths Pass 1,000 In 'Climate Catastrophe' │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 69    │ 1.2418689723  │ Weather Helping, But Threat From Western Fires Persists      │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 806   │ 1.2779963497  │ Twitter Bans Ads That Contradict Science On Climate Change   │
├───────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ 463   │ 1.3659076694  │ The Netherlands, Facing Energy And Climate Crises, Bets On A │
│       │               │  Nuclear Revival                                             │
└───────┴───────────────┴──────────────────────────────────────────────────────────────┘

Notice how searching "global warming" returns results about "climate change",
since they are semantically similar.

Very cool!
*/


```