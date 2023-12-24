---
title: "Partially-applied function. An `practical` example"
date: 2023-12-23T14:41:00+02:00
draft: true
---
## Partial application
>In computer science, partial application (or partial function application) refers to the process of fixing a number of arguments of a function, producing another function of smaller arity. [Source](https://en.wikipedia.org/wiki/Partial_application)

The way I see it, this is a function in which the final result is computed at different time:

```kotlin
  // Example from the Functional Programming with Kotlin book
  fun <A, B, C> partial1(a: A, f: (A, B) -> C): (B) -> C = { b -> f(a, b) }
```
In this case, we call a function passing in as arguments a `a:A` value and a function from (A, B) to C, producing another function from B to C.
In practice, and since there is only one way to implement it, when you call the returning function to produce C by passing the B argument, the `f` function gets called.

Example
```kotlin
    val partialTenMultipliedBy = partial1<Int, Int, Int>(a = 10, f = { a, b -> a * b } )
    val double = partialTenMultipliedBy(2) //yields 20
```

When you call `partialTenMultipliedBy` passing the number 2, this is the `B` argument of the _partial1_ returning function. This results in the block `{ b -> }` being called, which also calls the `f(A, B)` function, finally returning `C`.

Having partially applied functions is possible because Kotlin allows functions to be passed in as arguments or returned by other functions. Such a way of doing it is known as [Higher-order functions](https://kotlinlang.org/docs/lambdas.html#higher-order-functions).
Below, we will see a practical usage of this technique.

**Note:** As we will see below, this might incur a run-time overhead due to the creation of additional closures and anonymous functions.

## Use case
I was working on one of my never-finished side projects to try out new technologies and one of the tasks I had to implement was to display images.
The Android App consumes [TMDB](https://developer.themoviedb.org/) API. According to the documentation, you'll need three pieces of data to generate a fully working image URL: `base_url`, a `file_size` and a `file_path` (backdrop, logo, or poster).

We can retrieve the first two from the `/configuration` endpoint. However, we need to decide which _file_size_ is more suitable for our component based on its size.
At the end, we need to have a URL that looks like this one: _image.tmdb.org/t/p/original/wwemzKWzjKYJFfCeiB57q3r4Bcm.svg_. [Source](https://developer.themoviedb.org/docs/image-basics)

Currently, I have yet to implement any persistent solution (SQL, Room, or SharedPreferences), and left the URL construction to be made on the UI layer is error-prone, and we would be leaking information about what is needed to build a working image URL.

The network DTO would look like this:

```kotlin
data class MovieDTO(
    val id: Int,
    val backdropPath: String?, // this nullable type comes from the API
    val posterPath: String?, // this nullable type comes from the API
)
```

Since the information about `backdropPath` and `posterPath` alone is not that useful in the UI layer, they would be left out, and the UI model would look like this: 

```kotlin
data class Movie(
    val id: Int,
    override inline val buildImgModel: (type: ImageType) -> MovieImageModel?,
) : ImageModelBuilder<MovieImageModel> 
```

`ImageType` is just an enumeration class for  `Backdrop|Poster`, `ImageModelBuilder` is an interface with a function of an image type to a `T`. `MovieImageModel` is just a data class containing path and image type information. 

When we do the mapper between MovieDTO and Movie, we have a couple of options to fulfill the signature of the `buildImgModel`.
* pass a function with the same signature `(ImageType) -> MovieImageModel`
* construct the `MovieImageModel` directly
* create a function to hold the paths' information and return a function with the same signature needed

Let's see what the third option looks like in the map function:

```kotlin
suspend fun getMovies() : List<Movie> {
    val moviesDTO = someDependency.getMovies()
    return moviesDTO.map { dto ->
        Movie(
            id = dto.id,
            buildImgModel = buildPartialMovieImageModel(dto.backdropPath, dto.posterPath),
        )
    }
}

//Note: This is a closure, both backdropPath and posterPath are available within the body of the inner function.
private fun buildPartialMovieImageModel(
    backdropPath: String?,
    posterPath: String?,
) : (ImageType) -> MovieImageModel? = { type ->
    val path = when (type) {
        ImageType.Backdrop -> backdropPath
        ImageType.Poster -> posterPath
    }
    
    if (path.isNullOrEmpty()) null // for the brevity we're returning null, you could return a fallback ImageModel that displays local asset instead
    else MovieImageModel(path, type)
}
```

We partially apply the function, holding the paths' information _in memory_  waiting for the `type` information to be provided to create a fully formed `MovieImageModel` instance.
This is what we need; since the UI layer decides the type, this is the only information this layer is responsible for providing.

With the help of [coil](https://coil-kt.github.io/coil/getting_started/), we can pass `Any` model to be loaded, so depending on the component, we can create a _backdrop_ or a _poster_ model:
```
AsyncImage(model = state.movie.buildImgModel(ImageType.Backdrop))
```

We also need to [intercept](https://coil-kt.github.io/coil/image_pipeline/#interceptors) the request to use the information about the size of the component displaying the image and decide which size we will pass in the URL, and then proceed with the request with a full built URL.

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

    private fun buildFinalUrl(model: MovieImageModel, width: Int) : String { ... }
```

That's it :) 

With this, we create an `ImageModel` lazily by composing functions that partially apply their arguments at different levels of abstraction.

**Note:** If you already have a local persistent logic implemented, you won't need this; at the intercept level, you could, for example, query your local store to retrieve a path related to a movie ID. 

Resources
* [Tivi](https://github.com/chrisbanes/tivi) project consumed the same API and had to solve a similar issue to download images.
* [Functional Programming in Kotlin](https://www.goodreads.com/book/show/49199400-functional-programming-in-kotlin) p. 27

Thanks to [Marcello Galhardo](https://twitter.com/marcellogalhard) for reviewing the first draft. 
