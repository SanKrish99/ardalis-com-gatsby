---
templateKey: blog-post
title: Creating a WHERE NOT EXISTS or WHERE NOT IN Query Using LLBLGen
path: blog-post
date: 2011-01-27T11:43:00.000Z
description: Consider a scenario where you have a many-to-many relationship, but
  it’s nullable. For instance, maybe you have Articles and Categories (or Tags),
  and an Article can have 0 to N Categories/Tags.
featuredpost: false
featuredimage: /img/default-post-image.jpg
tags:
  - llblgen
  - query
  - SQL
category:
  - Software Development
comments: true
share: true
---
Consider a scenario where you have a many-to-many relationship, but it’s nullable. For instance, maybe you have Articles and Categories (or Tags), and an Article can have 0 to N Categories/Tags. Now, you want to pull in a set of Articles that are uncategorized. Using SQL, you might write a query like this:

```
<span style="color: #0000ff">SELECT</span> a.ID, a.Name
<span style="color: #0000ff">FROM</span> Article a
<span style="color: #0000ff">WHERE</span> <span style="color: #0000ff">NOT</span> <span style="color: #0000ff">EXISTS</span> (
<span style="color: #0000ff">SELECT</span> <span style="color: #0000ff">NULL</span>
<span style="color: #0000ff">FROM</span> Article_Category_Xref acx
<span style="color: #0000ff">WHERE</span> acx.ArticleID = a.ID

)
```

If you needed to do this using LLBLGen, and you didn’t know a better way, you could achieve this by first getting all of the tagged messages (easily done using an existing relationship), and then getting all messages except for these. Something like this:

**Wrong Way To Exclude Non-Related Set – Don’t Do This**

```
var bucket = <span style="color: #0000ff">new</span> RelationPredicateBucket();
var categorizedArticles = GetAllCategorizedArticles();
<span style="color: #0000ff">if</span> (autoGeneratedMessages.<span style="color: #0000ff">Count</span> &gt; 0)
{

   bucket.PredicateExpression.<span style="color: #0000ff">Add</span>(MessageFields.Id != autoGeneratedMessages.<span style="color: #0000ff">Select</span>(m =&gt; m.Id).ToArray());
}

var articleEntities = <span style="color: #0000ff">new</span> EntityCollection&lt;ArticleEntity&gt;();
<span style="color: #0000ff">using</span> (var myAdapter = PersistenceLayer.GetDataAccessAdapter())

{
  myAdapter.FetchEntityCollection(articleEntities , bucket);
}
```

What’s wrong with this approach? Well, for one thing it makes an extra database call that’s completely unnecessary, pulling back all of the data we *don’t*want in order to filter these out of the data we *do*want. For another, this will eventually lead to a SQL error when the number of rows we don’t want exceeds 2100 or so. If you see an error like this:

> **The incoming tabular data stream (TDS) remote procedure call (RPC) protocol stream is incorrect. Too many parameters were provided in this RPC request. The maximum is 2100.**

it’s likely that you have run into this problem.

Fortunately, LLBLGen supports this operation natively. This requirement isn’t the most common thing in the world, so finding the actual way to do it can be a bit challenging, hence my writing about it for my own and others’ reference. The key to this is the [FieldCompareSetPredicate (link to documentation)](http://www.llblgen.com/documentation/2.6/hh_start.htm#Using%20the%20generated%20code/Adapter/Filtering%20and%20Sorting/gencode_filteringpredicateclasses_adapter.htm#FieldCompareSetPredicate). Using the FieldCompareSetPredicate to find all rows in the Article table that do not have their ID in the Article_Category_Xref table is simply a matter of doing the following:

**Correct Way To Perform WHERE NOT EXISTS Using LLBLGen**

```
var bucket = RelationPredicateBucket();
bucket.PredicateExpression.Add(
  <span style="color: #0000ff">new</span> FieldCompareSetPredicate(
    ArticleFields.Id, 
    <span style="color: #0000ff">null</span>, 
    ArticleCategoryXrefFields.ArticleId, 
    <span style="color: #0000ff">null</span>,
    SetOperator.Exist,  
    (ArticleFields.Id == ArticleCategoryXrefFields.ArticleId), 
    <span style="color: #0000ff">true</span>));
    var articleEntities = <span style="color: #0000ff">new</span> EntityCollection&lt;ArticleEntity&gt;();
<span style="color: #0000ff">using</span> (var myAdapter = PersistenceLayer.GetDataAccessAdapter())
{
   myAdapter.FetchEntityCollection(articleEntities, bucket);
}
```

There are many overloads for the FieldCompareSetPredicate’s constructor. This one takes in the field to compare on (ArticleFields.Id), the field in the set to compare against (ArticleCategoryXrefFields.ArticleId), the set operation to use (SetOperator.Exist). The last two parameters are the comparison to perform, and whether or not the set operation should be negated (true == negate it). Thus, combining SetOperator.Exist with true is equivalent to a WHERE NOT EXISTS in SQL.