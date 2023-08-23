---
title: Food Discovery Demo
short_description: Feeling hungry? Multimodal semantic search makes it easy to find the perfect meal.
description: Feeling hungry? Multimodal semantic search makes it easy to find the perfect meal.
preview_dir: /articles_data/food-discovery-demo/preview
social_preview_image: /articles_data/food-discovery-demo/preview/social_preview.jpg
small_preview_image: /articles_data/food-discovery-demo/icon.svg
weight: -3
author: Kacper Łukawski
author_link: https://medium.com/@lukawskikacper
date: 2023-08-24T11:00:00.000Z
---

Not every search journey begins with a specific destination in mind. Sometimes, you just want to explore and see what’s out there and what you might like.
This is especially true when it comes to food. You might be craving something sweet, but you don’t know what. Or you might be looking for a new dish to try,
and you just want to see the options available. In these cases, you cannot express your needs in a textual query, as the thing you are looking for is not 
yet defined. This is where semantic search comes in. And it shines even brighter when combined with multiple modalities, such as text and images. It’s 
natural that we often choose our meals based on how they look, not how they are named, so exposing that way of search improves the UX significantly.

We are happy to announce a refreshed version of our Food Discovery Demo. This time available as an open source project, so you can easily deploy it on your own
and play with it. If you prefer to dive into the source code directly, then feel free to check out the [GitHub repository](https://github.com/qdrant/demo-food-discovery/). 
Otherwise, read on to learn more about the demo and how it works!

## General architecture

TODO: add graph like in project README

## Why did we use a CLIP model?

CLIP is a neural network that can be used to encode both images and texts into vectors. And more importantly, both images and texts are vectorized into the same
latent space, so we can compare them directly. This allows us to perform semantic search on images using text queries and the other way around. For example, if 
you search for “flat bread with toppings”, you will get images of pizza. Or if you search for “pizza”, you will get images of some flat bread with toppings, even
if they were not labeled as “pizza”. This is because CLIP embeddings capture the semantics of the images and texts and can find the similarities between them
no matter the wording.

TODO: add image describing how CLIP works or link a different article

CLIP is available in many different ways. We used the pretrained `clip-ViT-B-32` model available in the [Sentence-Transformers](https://www.sbert.net/examples/applications/image-search/README.html) 
library, as this is the easiest way to get started. 

## The dataset

The demo is based on the [Wolt](https://wolt.com/) dataset. It contains over 2M images of dishes from different restaurants along with some additional metadata. 
This is how a payload for a single dish looks like:

```json
{
    "cafe": {
        "address": "VGX7+6R2 Vecchia Napoli, Valletta",
        "categories": ["italian", "pasta", "pizza", "burgers", "mediterranean"],
        "location": {"lat": 35.8980154, "lon": 14.5145106},
        "menu_id": "610936a4ee8ea7a56f4a372a",
        "name": "Vecchia Napoli Is-Suq Tal-Belt",
        "rating": 9,
        "slug": "vecchia-napoli-skyparks-suq-tal-belt"
    },
    "description": "Tomato sauce, mozzarella fior di latte, crispy guanciale, Pecorino Romano cheese and a hint of chilli",
    "image": "https://wolt-menu-images-cdn.wolt.com/menu-images/610936a4ee8ea7a56f4a372a/005dfeb2-e734-11ec-b667-ced7a78a5abd_l_amatriciana_pizza_joel_gueller1.jpeg",
    "name": "L'Amatriciana"
}
```

Processing this amount of records takes some time, so we precomputed the CLIP embeddings, stored them in Qdrant collection and exported the collection as 
a snapshot. It's available for download from the [GCP bucket](https://storage.googleapis.com/common-datasets-snapshots/wolt-clip-ViT-B-32.snapshot).

## Different search modes

The FastAPI backend [exposes just a single endpoint](https://github.com/qdrant/demo-food-discovery/blob/6b49e11cfbd6412637d527cdd62fe9b9f74ac699/backend/main.py#L37), 
however it handles multiple scenarios. Let's dive into them one by one and understand why they are needed.

### Cold start

Recommendation systems struggle with a cold start problem. When a new user joins the system, there is no data about their preferences, so it’s hard to recommend
anything. The same applies to our demo. When you open it, you will see a random selection of dishes, and it changes every time you refresh the page. Internally, 
the demo [chooses some random points](https://github.com/qdrant/demo-food-discovery/blob/6b49e11cfbd6412637d527cdd62fe9b9f74ac699/backend/discovery.py#L70) in the 
vector space.

TODO: add a gif that shows how it works in UI

### Textual search

Since the demo suffers from the cold start problem, we implemented a textual search mode that is useful to start exploring the data. You can type in any text query
by clicking a search icon in the top right corner. The demo will use the CLIP model to encode the query into a vector and then search for the nearest neighbors
in the vector space. 

TODO: add gif showing how it works in UI

Under the hood this is [a group search query to Qdrant](https://github.com/qdrant/demo-food-discovery/blob/6b49e11cfbd6412637d527cdd62fe9b9f74ac699/backend/discovery.py#L44). 
We didn't use a simple search, but performed grouping by the restaurant to get more diverse results. [Search groups](https://qdrant.tech/documentation/concepts/search/#search-groups) 
is a mechanism similar to `GROUP BY` clause in SQL, and it's useful when you want to get a specific number of result per group (in our case just one). 

```python
import settings

# Encode query into a vector, model is an instance of
# sentence_transformers.SentenceTransformer that loaded CLIP model
query_vector = model.encode(query).tolist()

# Search for nearest neighbors, client is an instance of 
# qdrant_client.QdrantClient that has to be initialized before
response = client.search_groups(
    settings.QDRANT_COLLECTION,
    query_vector=query_vector,
    group_by=settings.GROUP_BY_FIELD,
    limit=search_query.limit,
)
```

### Exploring the results

The main feature of the demo is the ability to explore the space of the dishes. You can click on any of them to see more details, but first of all you can like or dislike it,
and the demo will update the search results accordingly. 

TODO: show how it works on a gif

#### Positive and negative feedback

#### Negative feedback only

The [Recommendation API](https://qdrant.tech/documentation/concepts/search/#recommendation-api) needs at least one positive example to work. However, in our demo
we want to be able to provide only negative examples. This is because we want to be able to say “I don’t like this dish” without having to like anything first.
To achieve this, we use a trick. We negate the vectors of the disliked dishes and use their mean as a query. This way, the disliked dishes will be pushed away 
from the search results.

TODO: a graphic showing how it works with cosine on 2d vectors

Food Discovery Demo [implements that trick](https://github.com/qdrant/demo-food-discovery/blob/6b49e11cfbd6412637d527cdd62fe9b9f74ac699/backend/discovery.py#L122)
by calling Qdrant twice. First of all we use the [Scroll API](https://qdrant.tech/documentation/concepts/points/#scroll-points) to find the disliked items, 
and then calculate a negated mean of all their vectors. That allows using the [Search Groups API](https://qdrant.tech/documentation/concepts/search/#search-groups)
to find the nearest neighbors of the negated mean vector and we are there!

```python
import numpy as np

# Retrieve the disliked points based on their ids
disliked_points, _ = client.scroll(
    settings.QDRANT_COLLECTION,
    scroll_filter=models.Filter(
        must=[
            models.HasIdCondition(has_id=search_query.negative),
        ]
    ),
    with_vectors=True,
)

# Calculate a mean vector of disliked points
disliked_vectors = np.array([point.vector for point in disliked_points])
mean_vector = np.mean(disliked_vectors, axis=0)
negated_vector = -mean_vector

# Search for nearest neighbors of the negated mean vector
response = client.search_groups(
    settings.QDRANT_COLLECTION,
    query_vector=negated_vector.tolist(),
    group_by=settings.GROUP_BY_FIELD,
    limit=search_query.limit,
)
```

### Location-based search

Last but not least. Location plays an important role in the food discovery process. You are definitely looking for something you can find nearby, not on the other
side of the globe. Therefore, all the search modes accepts your current location as a filtering criteria. This way you can find the best pizza in your neighborhood, 
not in the whole world. Qdrant [geo radius filter](https://qdrant.tech/documentation/concepts/filtering/#geo-radius) is a perfect choice for this. It allows you to
filter the results by distance from a given point. 

```python
from qdrant_client import models

# Create a geo radius filter
query_filter = models.Filter(
    must=[
        models.FieldCondition(
            key="cafe.location",
            geo_radius=models.GeoRadius(
                center=models.GeoPoint(
                    lon=location.longitude,
                    lat=location.latitude,
                ),
                radius=location.radius_km * 1000,
            ),
        )
    ]
)
```

Such a filter needs [a payload index](https://qdrant.tech/documentation/concepts/indexing/#payload-index) to work efficiently, and it was created on a collection
we used to create the snapshot. So if you import it, the index will be already there.

## Using the demo

The Food Discovery Demo is available online, but if you prefer to run it locally, you can do it with Docker. The [README](https://github.com/qdrant/demo-food-discovery/blob/main/README.md)
describes all the steps more in detail, but here is a quick start:

```bash
git clone git@github.com:qdrant/demo-food-discovery.git
cd demo-food-discovery
docker-compose up -d
```

The demo will be available at http://localhost:8000.

TODO: add a screenshot of the UI

## Fork and reuse

The full source code of this demonstration is open-sourced and available for you to fork, adapt, and build upon. Whether you’re looking to understand the mechanics 
of semantic search or to have a foundation to build a larger project, this demo can serve as a starting point.