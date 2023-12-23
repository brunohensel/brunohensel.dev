---
title: "Partially-applied function an `practical` example"
date: 2023-12-23T14:41:00+02:00
draft: true
---
## Partial application
>In computer science, partial application (or partial function application) refers to the process of fixing a number of arguments of a function, producing another function of smaller arity. [Source](https://en.wikipedia.org/wiki/Partial_application)

The way I see it, this is a function, which the final result is computed in different times:

```kotlin
  // example from the Functional Programming with Kotlin
  fun <A, B, C> partial1(a: A, f: (A, B) -> C): (B) -> C = { b -> f(a, b) }
```
In this case, we call a function passing in as arguments a `a:A` value and a function from (A, B) to C, this produces another function from B to C.
In practice, and since there is only one way to implement it, when you call the returning function to produce C, by passing the B argument, the `f` function gets called.

Example
```kotlin
    val partialTenMultipliedBy = partial1<Int, Int, Int>(a = 10, f = { a, b -> a * b } )
    val double = partialTenMultipliedBy(2) //yields 20
```

When you call `partialTenMultipliedBy` passing the number 2, this is the `B` argument of the _partial1_ returning function. This results the block `{ b -> }` being called, which also calls the `f(A, B)` function, finally returning `C`.

Above we will see a practical usage of this technic.

**Note:** As we will see above, this might incur a run-time overhead due to the creation of additional closures and anonymous functions.

## Use case
I was working on one of my never-finished side project to try out new technologies and one of the tasks I had to implement was to display images.
The Android App consumes [TMDB](https://developer.themoviedb.org/) API. According to the documentation, to generate a fully working image URL, you'll need 3 pieces of data: `base_url`, a `file_size` and a `file_path` (backdrop, logo, or poster).

The first two we can retrieve from the `/configuration` endpoint. However, we need to decide which size is more suitable for our component based on its size.
At the end, we need to have a URl that looks like this one: _image.tmdb.org/t/p/original/wwemzKWzjKYJFfCeiB57q3r4Bcm.svg_. [source](https://developer.themoviedb.org/docs/image-basics)

At this point, I don't have any persistent solution implemented (SQL, Room, or SharedPreferences). And left the construction of the URL to be made on the UI layer is error-prone, and we would be leaking an information about how to build an image URL.

The network DTO would look like this:

```kotlin
data class MovieDTO(
    val id: Int,
    val backdropPath: String?,
    val posterPath: String?
)
```

Since the information about `backdropPath` and `posterPath` alone is not the useful in the UI layer, they would be left out and the UI model would look like this: 

```kotlin
data class Movie(
    val id: Int,
    override inline val buildImgModel: (type: ImageType) -> MovieImageModel,
) : ImageModelBuilder<MovieImageModel> 
```

`ImageType` is just an enumeration class for  `Backdrop|Poster`, `ImageModelBuilder` is an interface with a function of an image type to a `T`. `MovieImageModel` is a data class, which holds information about backdrop and poster paths and image type. 

When we do the mapper between the MovieDTO and Movie we have a couple of options to fulfill the signature of the `buildImgModel`.
* pass a function with the same signature `(ImageType) -> MovieImageModel`
* construct the `MovieImageModel` directly
* create a function to hold the paths' information, and return a function with the same signature needed

Let's see how the third option looks like in the map function:

```kotlin
suspend fun getMovies() : List<Movie> {
    val moviesDTO = someDependency.getMovies()
    return moviesDTO.map { dto ->
        Movie(
            id = dto.id,
            buildImgModel = partiallyCompute(dto.backdropPath, dto.posterPath)
        )
    }
}
//Note: This is a closure, both backdropPath and posterPath are available within the body of the inner function.
private fun partiallyCompute(
    backdropPath: String?,
    posterPath: String?
) : (ImageType) -> MovieImageModel = { type ->
    MovieImageModel(backdropPath,posterPath, type)
}
```

We partially compute the function, hold the paths' information _in memory_ and wait for the `type` information, when it's available, we finish the computation and return the fully constructed `MovieImageModel`.
This is what we need, since the type is decided by the UI layer, this is the only information this layer is responsible to provide.

With the help of [coil](https://coil-kt.github.io/coil/getting_started/), we can pass `Any` model to be loaded, so depending on the component, we can create a _backdrop_ or a _poster_ model:
```
AsyncImage(model = state.movie.buildImgModel(ImageType.Backdrop))
```

We also need to [intercept](https://coil-kt.github.io/coil/image_pipeline/#interceptors) the request in order to use the information about the size of the component requesting the image and decide which size we will pass in the URL, and then proceed with the request with a full built URL.

On a high level, this could be similar to this one:

```kotlin
    override suspend fun intercept(chain: Interceptor.Chain): ImageResult {
        val rq = when (val model = chain.request.data) {
            is MovieImageModel -> {
                chain.request.newBuilder()
                    .data(buildFinalUrl(model, chain.size.width))
                    .build()
            }

            else -> chain.request
        }

        return chain.proceed(rq)
    }
```

That's it :)