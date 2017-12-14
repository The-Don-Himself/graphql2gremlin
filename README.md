# graphql2gremlin

![Image](graphql2gremlin.png?raw=true "graphql2gremlin")

## The What?

GraphQL2Gremlin to an attempt create a standard way to traverse a Gremlin interfaced Graph database using GraphQL.

## The How?

Its ridiculously simple really, all we do is tie down certain notions in GraphQL to mean something in Gremlin. Let's start with GraphQL arguments. In the GraphQL2Gremlin standard, each level of an argument **should** be either an edge or a vertex and if so **MUST** match the label. Thereafter, each argument level preceding it **MUST** be the opposite. Furthermore, all argument properties that represent traversal steps **MUST** begin with an underscore. Lastly, all argument properties that do not represent a traversal step must be a search predicate and thus match GraphQL input type STRING. It's best explained using a practical example: Twitter.


At it's utmost simplest form, Twitter can be graphed as a set of only 2 nodes; users and tweets where the edges signify actions between them. Let's say we have vertexes with label 'users' and 'tweets' and edges between them called 'tweeted', 'liked' and edges between 'users' called 'followedby', 'following' and finally edges between 'tweets' called 'retweeted'

![Image](twittergraph.png?raw=true "simple twitter graph")

Let's fetch a users friends for user_id 5 and get the id, username and bio fields

````
{
  users(
    following: {
      users: {
        user_id: "eq(5)"
      }
    }
  ) {
    user_id
    username
    bio
  }
}
````

Notice users matches vertex label 'users', same to argument following matches edge label 'following', also notice how user_id is a search predicate, meaning it could have between gt(5), or lte(1000) or between(1000, 2000) or [any other](http://tinkerpop.apache.org/docs/current/reference/#a-note-on-predicates). Predicates may also depend on the Graph Database in use, I'm currently using [JanusGraph](https://github.com/JanusGraph/janusgraph) which accepts a few other [predicates](http://docs.janusgraph.org/latest/search-predicates.html) as well such as [text search](http://docs.janusgraph.org/latest/search-predicates.html#_text_predicate) and [geo](http://docs.janusgraph.org/latest/search-predicates.html#_geo_predicate).

Let's get a bit more complex and fetch only 10 friends of a user_id with user_id 5, with a last_login date within a day, who've retweeted tweets with the word 'graph' liked by the user (i.e user_id 5), we'll also sort these users by username in decreasing order and get the id, username and bio fields

````
{
  users(
    last_login: "gte(123456789)", # Depends on how you store you're datetime function
    retweeted: {
      tweets: {
        text: "textContains('Graph')", # Depends on you Graph Database
        liked: {
          users: {
            user_id: "eq(5)"
          }
        }
      }
    },
    _sort: "username",
    _order: "decr",
    _limit: 10
  ) {
    user_id
    username
    bio
  }
}
````

Still not satisfied, well, I actually use this in production for my startup Campus Discounts, a social network where students find and recommend discounts on offer by businesses near campus.

You can play around with a live GraphiQL portal https://campus-discounts.com/graphql/explorer 

````
{
  discounts(
    posted: "lte(2017-1-01T00:00:00+0300)", # A Valid PHP DateTime::ISO8601 format
    circle: "geoWithin(-1.291536, 36.845736, 10)", # Lat,Lon Coordinates for Nairobi, Kenya
    page: {
      pages: {
        averagepricerating: "lte(4.5)", # Out of 5 starts, use less than because we dont have a lot of data yet
        country: {
          countries: {
            name: "textContains('Kenya')" # Again Only Kenya has a decent amount of data to query as of right now
          }
        }
      }
    },
    _sort: "posted",
    _order: "decr",
    _limit: 10
  ) {
    id
    title
    description
    oldprice
    newprice
    posted
    discountcategories {
      taggedon
      category {
        id
        name
      }
    }
    page{
      id
      name
      about
      averagepricerating
      country {
        id
        name
        currencycode
        currencysymbol
        exchangerate
      }
    }
  }
}
````

You can play around with our GraphiQL portal but please be mindful of resources, avoid deeply nested traversals which I haven't even fully implemented yet so you're bound to get incorrect data if you try. Otherwise enjoy! But back to the GraphQL2Gremlin standard.


## The Why?

Several reasons

1.) Simplicity

I wanted to expose my Graph Database through an API. Graph DBs aren't that hard to understand -> you can connect a friend to friends to other friends - simple right? However, they can be a bit tricky to use -> I started from user Barack Obama traversed to his friends found Joe Biden traversed to his Friends and landed back at Obama. What? Solution, filter out the start traversal. Ok, I want to get all universities with geniuses, i.e students taking Rocket Science and under the age of 17, I got 20,000 students but also 20,000 universities, how? Either aggregate the traversal after getting all the students or better yet de-duplicate your universities results at the end.

GraphQL allows me to hide this complexity from my API since i can both control and predict some of the queries being made.

2.) Security

Gremlin is a very powerful, and constantly evolving API. Exposing it directly to end-users can be very dangerous. I needed to control exactly what features I can give them, if I don’t want a to allow them to do a profile() or explain() step (which I don't by the way), simply leave it out of a GraphQL schema, the rest have to be either a predicate or a traversal so controlling and sanitizing that input is much easier.

3.) Flexibility

GraphQL is very flexible. Take the Twitter Graph for example, in reality there might not be a 'following' and 'followedby' edges but just a single 'follows' edges with 'in' and 'out' directions. With GraphQL you can expose virtual edges such as the 'following' and 'followedby' edges but in actuality what happens is that one is traversed using out('follows') and the other in('follows').

4.) Portability

GraphQL doesn't care whether it is JanusGraph, CosmosDB or Neo4j, whether it's Gremlin or Cypher. In this rapidly evolving space of Graph DBs, that could be vital



All said and done, this standard is still very early and certainly doesn't and probably can't cater for every use case or traversal but for most. Some of the limitations I can think of from the top of my head are:-

1) Only handles queries not mutations
2) Deeply nested queries would be a real pain to handles
3) Grouping data to form related objects is natively hard in graphs. Traversals by default go from one element to another, you'd have to do extra work to store the elements touched along the way and save them as an object(s). It's manageable when you control the queries less so when you don't. Personally I get the ids from the parent label and get the objects from Redis.
