# How to Use Django & PostgreSQL for Full Text Search

## Introduction

For any project there may be a need to use a database full-text search. We
expect high speed and relevant results from this search. When we face such
problem, we usually think about Solr, ElasticSearch, Sphinx, AWS CloudSearch,
etc. But in this article we will talk about PostgreSQL. Starting from version
8.3, a full-text search support in PostgreSQL is available. Let's look at how
it is implemented in the DBMS itself.

The basic idea of a full-text search in PostgreSQL is that you create an index
using a document, which contains the words you would like to use for search.
For storing search data in PostgreSQL, new data type has been added -
tsvector. Tsvector is a storage for the tokens from a document.

    
    
    postgres=# SELECT to_tsvector('All cats love fish but hate to get their paws wet'); to_tsvector
    -------------------------------------------------------------- 'cat':2 'fish':4 'get':8 'hate':6 'love':3 'paw':10 'wet':11 (1 row)
    

_to_tsvector_ \- is a support function that has been implemented to convert a
document into the tsvector data type.

We can use logical operators `&` (AND), `|` (OR), and `!` (NOT) for our
queries. We need to use `tsquery` for this:

    
    
    postgres=# SELECT to_tsquery('fishes | dog); to_tsquery
    ---------------- 'fish' | 'dog' (1 row)
    

When we want to check the search for a match, we have to use a full-text
operator `@@`:

    
    
    postgres=# SELECT to_tsvector('All cats love fish but hate to get their paws wet') @@ to_tsquery('fishes | dog'); ?column?
    ---------- t (1 row)
    

Excellent! The match is found. Let's write a query that uses a logical
operator `AND`:

    
    
    postgres=# SELECT to_tsvector('All cats love fish but hate to get their paws wet') @@ to_tsquery('fishes & cat'); ?column?
    ---------- t (1 row)
    

Match is found again. Now let's modify the query so that it could return a
negative result:

    
    
    postgres=# SELECT to_tsvector('All cats love fish but hate to get their paws wet') @@ to_tsquery('cat & dog'); ?column?
    ---------- f (1 row)
    

More information can be found in the PostgreSQL documentation. So at this
point we will stop talking about full-text search in PostgreSQL and move on to
using this in Django.

## Using a full text search django

A full-text search with PostgreSQL support in Django appeared with version
1.10. In previous Django versions the full-text search in PostgreSQL was
possible with djorm-ext-pgfulltext library.

Let's look at what kind of interface gives for us django postgres full text
search. The easiest way to do a full-text search using `search`. Let's
consider the following example:

    
    
    Post.objects.filter(body__search='programming') # [<Post: Computer programming>, <Post: Neural networks and deep learning>]
    

In our `Post` model 2 matches with the word `Programming` have been found.
This record creates `to_tsvector` in the database for the `body` field and
`plainto_tsquery` for `Programming`.

### SearchVector

`search` lookup is great when we need to search by one field. If we want to
search by two or more fields, SearchVector should be used, for example:

    
    
    from django.contrib.postgres.search import SearchVector Post.objects.annotate(search=SearchVector('body', 'category__title')).filter(search='programming') # [<Post: Computer programming>, <Post: Neural networks and deep learning>, <Post: Mastering Basic Algorithms in the Python>]
    

As you can see, this time we were searching by `body` and `category__title`
fields, and as a result, found 3 matches. `SearchVector` objects can be
grouped together for the further reuse (the DRY principles are followed):

    
    
    from django.contrib.postgres.search import SearchVector body_vector = SearchVector('body')
    category_title_vector = SearchVector('category__title')
    Post.objects.annotate(search=body_search + category_title_search).filter(search='programming')
    

### SearchQuery

If we need to convert the word into the search query object, we can use
SearchQuery:

    
    
    from django.contrib.postgres.search import SearchQuery SearchQuery('python') & SearchQuery('java') # python AND java
    SearchQuery('python') | SearchQuery('java') # python OR java
    ~SearchQuery('java') # NOT java
    

### SearchRank

Now let's assume that we need to sort the search results by relevance.
PostgreSQL provides a ranging function for these purposes, which determines
how often the words are found in the document, how these words are close to
each other in the document and how important that part of the document, where
these words are situated, is. Ranking interface in Django full text search is
provided by `SearchRank`. Let's look at an example:

    
    
    from django.contrib.postgres.search import SearchQuery, SearchRank, SearchVector body_vector = SearchVector('body')
    term_query = SearchQuery('programming')
    Post.objects.annotate(rank=SearchRank(body_vector, term_query)).order_by('-rank')
    

The most relevant records are at the top.

## Search configuration

SearchQuery and SearchVector accept the config as a parameter. It is necessary
for using different search settings, for example, if you want to change the
language parser or dictionary.

    
    
    from django.contrib.postgres.search import SearchQuery, SearchVector Post.objects.annotate(search=SearchVector('body', config='german')).filter(search=SearchQuery('programmierung', config='german'))
    

We can also control the relevance, by setting different weight to those
fields, which we plan to search by. For this `SearchVector` accepts a `weight`
parameter. As the `weight` value there can be one of these letters: D, C, B,
A. The default values for these letters are 0.1, 0.2, 0.4 and 1.0
respectively.

    
    
    from django.contrib.postgres.search import SearchQuery, SearchRank, SearchVector search_vector = SearchVector('body', weight='A') + SearchVector('category__title', weight='B')
    term_query = SearchQuery('programming')
    Post.objects.annotate(rank=SearchRank(search_vector, term_query)).filter(rank__gte=0.3).order_by('rank')
    

## Optimization

Search in your project may be one of the most resource-intensive operations,
especially if it is a search in a few hundred or even thousands of records.
And this is the point when the optimization problem arises. Let's look at the
tools for optimization which PostgreSQL gives us and how it is implemented in
Django.

### Indexes

To speed up the search, PostgreSQL offers the use of indexes. It is worth
noting that the index is not required for a full-text search. But if the
search takes place in specific (constant) columns, the presence of the index
is desirable.

There are 2 types of indexes:

GiST (Generalized Search Tree) - a bit signature is assigned for each
document, which contains information about all the tokens that are in this
document. It is created with a command:

    
    
    CREATE INDEX name ON table USING GIST (column);
    

GIN (Generalized Inverted Index) - in this type of index the key is a token
and the value is an organized list of document identifiers which contain the
token. It is created with a command:

    
    
    CREATE INDEX name ON table USING GIN (column);
    

There is a big difference in performance between these two types of indexes.
Before you choose which one to use, I recommend you to study the documentation
on them in detail. I would like to add that:

  * the creation of GIN index is 3 times faster than GiST
  * GIN index is 2-3 times bigger than GiST-index
  * Search for GIN index is 3 times faster than GiST index
  * GiN index updates 10 times slower

It is best to use GiST index for updated data, and GIN index suits well for
static data.

### SearchVectorField

Another way to speed up the search is to use SearchVectorField in the model.
You will need to use triggers to fill in this field, for example, as described
in the [Triggers for Automatic
Updates](https://www.postgresql.org/docs/current/static/textsearch-
features.html#TEXTSEARCH-UPDATE-TRIGGERS) documentation. After that, you can
query the field, so if it is annotated SearchVector.

    
    
    Post.objects.update(search_vector=SearchVector('body_text'))
    Post.objects.filter(search_vector='programming') # [<Post: Computer programming>, <Post: Neural networks and deep learning>]
    

## Conclusion

When you need a full-text search, and you do not want to connect third-party
libraries to the project, you can easily manage database resources using a
full-text search in PostgreSQL and the api, which provides Django for work
with the search.
