# Procyon
<img align="right" src="https://res.cloudinary.com/jochemstoel/image/upload/v1622723144/procyon_h7wuaa.png">

Procyon is a genus of nocturnal mammals, comprising three species commonly known as raccoons. This is a _rewrite_ of recommendationRaccoon to better support multiple instances, cleaner code and better use of modern JavaScript language features.

The reason for this is that I wanted to run multiple instances of Raccoon for different things (Books, Movies, Music, Porn, ...) but that was not very practical. Each instance of Procyon can use it's own configuration options. See below.

### From RecommendationRaccoon
> An easy-to-use collaborative filtering based recommendation engine and NPM module built on top of Node.js and Redis. The engine uses the Jaccard coefficient to determine the similarity between users and k-nearest-neighbors to create recommendations. This module is useful for anyone with users, a store of products/movies/items, and the desire to give their users the ability to like/dislike and receive recommendations based on similar users. Raccoon takes care of all the recommendation and rating logic. It can be paired with any database as it does not keep track of any user/item information besides a unique ID.

## Requirements

* Node.js 6.x
* Redis
* Async
* Underscore
* Bluebird

## Installation

``` bash
npm install procyon
```

## Quickstart

Procyon keeps track of the ratings and recommendations from your users. It does not need to store any meta data of the user or product aside from an id. To get started:

#### Install Procyon:
``` bash
npm install procyon
```

#### Setup Redis:
If local:
``` bash
npm install redis
redis-server
```

If remote or you need to customize the connection settings use the process.env variables:
- PROCYON_REDIS_URL
- PROCYON_REDIS_PORT
- PROCYON_REDIS_AUTH

#### Require procyon:
``` js
const procyon = require('procyon')
```

#### Instantiate 
Unlike Raccoon, you need to instantiate it. Optionally you can override options. 

``` js 
// with default options 
var procyon = new Procyon()
```

These are the default options:
``` js 
{
    nearestNeighbors: 5,
    className: 'movie',
    numOfRecsStore: 30,
    factorLeastSimilarLeastLiked: false,
    redisUrl: process.env.PROCYON_REDIS_URL || '127.0.0.1',
    redisPort: process.env.PROCYON_REDIS_PORT || 6379,
    redisAuth: process.env.PROCYON_REDIS_AUTH || ''
}
```

#### Multiple

``` js 
var procyon1 = new Procyon({
    className: 'actor'
})

var procyon2 = new Procyon({
    className: 'movie'
})

/* Actors */
await procyon1.liked('gary', 'Jim Carrey')
await procyon1.liked('gary', 'Keanu Reeves')
await procyon1.liked('gary', 'Asa Akira')
await procyon1.liked('gary', 'Riley Reid')
await procyon1.disliked('gary', 'Sylvester Stallone')
    
await procyon1.liked('pete', 'Asa Akira')
await procyon1.liked('pete', 'Jim Carrey')
await procyon1.disliked('pete', 'Hillary Duff')

/* Movies */
await procyon2.liked('gary', 'The Matrix')
await procyon2.liked('gary', 'John Wick')
await procyon2.disliked('gary', 'Titanic')
    
await procyon2.liked('pete', 'John Wick')
await procyon2.liked('pete', 'Hunger Games')
await procyon2.liked('pete', 'Wall-E')
await procyon2.disliked('pete', 'Titanic')

let recommendations_actors = await procyon1.recommendFor('pete', 10)
let recommendations_movies = await procyon2.recommendFor('gary', 10)

console.log({ recommendations_actors, recommendations_movies })
```

##### Result 
``` js
{
  recommendations_actors: [ 'Riley Reid', 'Keanu Reeves', 'Sylvester Stallone' ],
  recommendations_movies: [ 'Wall-E', 'Hunger Games' ]
}
```

``` js
procyon.liked('userId', 'itemId', options).then(() => {
    //
})
// available options are:
{
  updateRecs: false
    // this will stop the update sequence for this rating
    // and greatly speed up the time to input all the data
    // however, there will not be any recommendations at the end.
    // if you fire a like/dislike with updateRecs on it will only update
    // recommendations for that user.
    // default === true
}

// options are available to liked, disliked, unliked, and undisliked.

```

``` js
procyon.unliked('userId', 'itemId').then(() => {
    // removes the liked rating from all sets and updates. not the same as disliked.
})
```

#### Dislikes:
``` js
procyon.disliked('userId', 'itemId').then(() => {
    // negative rating of the item. if user1 liked movie1 and user2 disliked it, their
    // jaccard would be -1 meaning the have opposite preferences.
})
```

``` js
procyon.undisliked('userId', 'itemId').then(() => {
    // similar to unliked. removes the negative disliked rating as if it was never rated.
})

```

### Recommendations

``` js
procyon.recommendFor('userId', 'numberOfRecs').then((results) => {
  // returns an ranked sorted array of itemIds which represent the top recommendations
  // for that individual user based on knn.
  // numberOfRecs is the number of recommendations you want to receive.
  // asking for recommendations queries the 'recommendedZSet' sorted set for the user.
  // the movies in this set were calculated in advance when the user last rated
  // something.
  // ex. results = ['batmanId', 'supermanId', 'chipmunksId']
});

procyon.mostSimilarUsers('userId').then((results) => {
  // returns an array of the 'similarityZSet' ranked sorted set for the user which
  // represents their ranked similarity to all other users given the
  // Jaccard Coefficient. the value is between -1 and 1. -1 means that the
  // user is the exact opposite, 1 means they're exactly the same.
  // ex. results = ['garyId', 'andrewId', 'jakeId']
});

procyon.leastSimilarUsers('userId').then((results) => {
  // same as mostSimilarUsers but the opposite.
  // ex. results = ['timId', 'haoId', 'phillipId']
});
```

### User Statistics

#### Ratings:
``` js
procyon.bestRated().then((results) => {
  // returns an array of the 'scoreboard' sorted set which represents the global
  // ranking of items based on the Wilson Score Interval. in short it represents the
  // 'best rated' items based on the ratio of likes/dislikes and cuts out outliers.
  // ex. results = ['iceageId', 'sleeplessInSeattleId', 'theDarkKnightId']
});

procyon.worstRated().then((results) => {
  // same as bestRated but in reverse.
});
```

#### Liked/Disliked lists and counts:
``` js
procyon.mostLiked().then((results) => {
  // returns an array of the 'mostLiked' sorted set which represents the global
  // number of likes for all the items. does not factor in dislikes.
});

procyon.mostDisliked().then((results) => {
  // same as mostLiked but the opposite.
});

procyon.likedBy('itemId').then((results) => {
  // returns an array which lists all the users who liked that item.
});

procyon.likedCount('itemId').then((results) => {
  // returns the number of users who have liked that item.
});

procyon.dislikedBy('itemId').then((results) => {
  // same as likedBy but for disliked.
});

procyon.dislikedCount('itemId').then((results) => {
  // same as likedCount but for disliked.
});

procyon.allLikedFor('userId').then((results) => {
  // returns an array of all the items that user has liked.
});

procyon.allDislikedFor('userId').then((results) => {
  // returns an array of all the items that user has disliked.
});

procyon.allWatchedFor('userId').then((results) => {
  // returns an array of all the items that user has liked or disliked.
});
```


## Recommendation Engine Components

### Jaccard Coefficient for Similarity

There are many ways to gauge the likeness of two users. The original implementation of recommendation Procyon used the Pearson Coefficient which was good for measuring discrete values in a small range (i.e. 1-5 stars). However, to optimize for quicker calcuations and a simplier interface, recommendation Procyon instead uses the Jaccard Coefficient which is useful for measuring binary rating data (i.e. like/dislike). Many top companies have gone this route such as Youtube because users were primarily rating things 4-5 or 1. The choice to use the Jaccard's instead of Pearson's was largely inspired by David Celis who designed Recommendable, the top recommendation engine on Rails. The Jaccard Coefficient also pairs very well with Redis which is able to union/diff sets of like/dislikes at O(N).

### K-Nearest Neighbors Algorithm for Recommendations

To deal with large user bases, it's essential to make optimizations that don't involve comparing every user against every other user. One way to deal with this is using the K-Nearest Neighbors algorithm which allows you to only compare a user against their 'nearest' neighbors. After a user's similarity is calculated with the Jaccard Coefficient, a sorted set is created which represents how similar that user is to every other. The top users from that list are considered their nearest neighbors. recommendation Procyon uses a default value of 5, but this can easily be changed based on your needs.

### Wilson Score Confidence Interval for a Bernoulli Parameter

If you've ever been to Amazon or another site with tons of reviews, you've probably ran into a sorted page of top ratings only to find some of the top items have only one review. The Wilson Score Interval at 95% calculates the chance that the 'real' fraction of positive ratings is at least x. This allows for you to leave off the items/products that have not been rated enough or have an abnormally high ratio. It's a great proxy for a 'best rated' list.

### Redis

When combined with hiredis, redis can get/set at ~40,000 operations/second using 50 concurrent connections without pipelining. In short, Redis is extremely fast at set math and is a natural fit for a recommendation engine of this scale. Redis is integral to many top companies such as Twitter which uses it for their Timeline (substituted Memcached).