#### StorIO — modern API for SQLiteDatabase and ContentResolver

#####Overview:
* Powerful & Simple set of Operations: `Put`, `Get`, `Delete`
* API for Humans: Type Safety, Immutability & Thread-Safety
* Convenient builders with compile-time guarantees for required params. Forget about 6-7 `null` in queries
* Optional Type-Safe Object Mapping, if you don't want to work with `Cursor` and `ContentValues` you don't have to
* No reflection in Operations and no annotations in the core, also `StorIO` is not ORM
* **Full control** over queries, transaction and object mapping
* Every Operation over `StorIO` can be executed as blocking call or as `rx.Observable`/`rx.Single`/`rx.Completable`
* `RxJava` as first class citizen, but it's not required dependency!
* **Reactive**: `rx.Observable` from `Get` Operation **will observe changes** in `StorIO` (`SQLite` or `ContentProvider`) and receive updates automatically
* `StorIO` is replacements for `SQLiteDatabase` and `ContentResolver` APIs
* `StorIO` + `RxJava` is replacement for `Loaders` API
* We are working on `MockStorIO` (similar to [MockWebServer](https://github.com/square/okhttp/tree/master/mockwebserver)) for easy unit testing

----

#####Why StorIO?
* Simple concept of just three main Operations: `Put`, `Get`, `Delete` -> less bugs
* Almost everything is immutable and thread-safe -> less bugs
* Builders for everything make code much, much more readable and obvious -> less bugs
* Our builders give compile time guarantees for required parameters -> less bugs
* `StorIO` annotated with `@NonNull` and `@Nullable` annotations -> less bugs
* Open Source -> less bugs
* Documentation, Sample app and Design tests -> less bugs
* `StorIO` has unit and integration tests [![codecov.io](https://codecov.io/github/pushtorefresh/storio/coverage.svg?branch=master)](https://codecov.io/github/pushtorefresh/storio?branch=master) -> less bugs
* Less bugs -> less bugs

####Documentation:

* [`Why we made StorIO`](https://engineering.pushtorefresh.com/2015/07/02/storio-modern-replacement-for-sqlitedatabase-and-contentresolver-apis/)
* [`StorIO SQLite`](docs/StorIOSQLite.md)
* [`StorIO ContentResolver`](docs/StorIOContentResolver.md)

Easy ways to learn how to use `StorIO` -> check out `Documentation`, `Design Tests` and `Sample App`:

* [Design tests for StorIO SQLite](storio-sqlite/src/test/java/com/pushtorefresh/storio/sqlite/design)
* [Design tests for StorIO ContentResolver](storio-content-resolver/src/test/java/com/pushtorefresh/storio/contentresolver/design)
* [Sample App](storio-sample-app)

####Download:
```groovy
// If you need StorIO for SQLite
compile 'com.pushtorefresh.storio:sqlite:1.9.1'

// If you need StorIO for ContentResolver
compile 'com.pushtorefresh.storio:content-resolver:1.9.1'

// IN StorIO 2.0 we will remove default Scheduling from Rx Operations!
// You'll have to put subscribeOn() manually!

// Notice that RxJava is optional dependency for StorIO,
// So if you need it -> please add it manually.
```

You can find all releases on [Maven Central](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.pushtorefresh.storio%22).

####Some examples

#####Get list of objects from SQLiteDatabase
```java
List<Tweet> tweets = storIOSQLite
  .get()
  .listOfObjects(Tweet.class) // Type safety
  .withQuery(Query.builder() // Query builder
    .table("tweets")
    .where("author = ?")
    .whereArgs("artem_zin") // Varargs Object..., no more new String[] {"I", "am", "tired", "of", "this", "shit"}
    .build()) // Query is immutable — you can save it and share without worries
  .prepare() // Operation builder
  .executeAsBlocking(); // Control flow is readable from top to bottom, just like with RxJava

```

#####Put something to SQLiteDatabase
```java
storIOSQLite
  .put() // Insert or Update
  .objects(someTweets) // Type mapping!
  .prepare()
  .executeAsBlocking();
```

#####Delete something from SQLiteDatabase
```java
storIOSQLite
  .delete()
  .byQuery(DeleteQuery.builder()
    .table("tweets")
    .where("timestamp <= ?")
    .whereArgs(System.currentTimeMillis() - 86400) // No need to write String.valueOf()
    .build())
  .prepare()
  .executeAsBlocking();
```

####Reactive? Observable.just(true)!

#####Get something as rx.Observable and receive updates!
```java
storIOSQLite
  .get()
  .listOfObjects(Tweet.class)
  .withQuery(Query.builder()
    .table("tweets")
    .build())
  .prepare()
  .asRxObservable() // Get Result as rx.Observable and subscribe to further updates of tables from Query!
  .observeOn(mainThread()) // All Rx operations work on Schedulers.io()
  .subscribe(tweets -> { // Please don't forget to unsubscribe
  	  // Will be called with first result and then after each change of tables from Query
  	  // Several changes in transaction -> one notification
  	  adapter.setData(tweets);
  	}
  );
```

#####Want to work with plain Cursor, no problems
```java
Cursor cursor = storIOSQLite
  .get()
  .cursor()
  .withQuery(Query.builder() // Or RawQuery
    .table("tweets")
    .where("who_cares = ?")
    .whereArgs("nobody")
    .build())
  .prepare()
  .executeAsBlocking();
```

####How object mapping works?
#####You can set default type mappings when you build instance of `StorIOSQLite` or `StorIOContentResolver`

```java
StorIOSQLite storIOSQLite = DefaultStorIOSQLite.builder()
  .sqliteOpenHelper(someSQLiteOpenHelper)
  .addTypeMapping(Tweet.class, SQLiteTypeMapping.<Tweet>builder()
    .putResolver(new TweetPutResolver()) // object that knows how to perform Put Operation (insert or update)
    .getResolver(new TweetGetResolver()) // object that knows how to perform Get Operation
    .deleteResolver(new TweetDeleteResolver())  // object that knows how to perform Delete Operation
    .build())
  .addTypeMapping(...)
  // other options
  .build(); // This instance of StorIOSQLite will know how to work with Tweet objects
```

You can override Operation Resolver per each individual Operation, it can be useful for working with `SQL JOIN`.

---

To **save you from coding boilerplate classes** we created **Annotation Processor** which will generate `PutResolver`, `GetResolver` and `DeleteResolver` at compile time, you just need to use generated classes

*Notice that annotation processors are not part of the library core, you can work with StorIO without them, we just made them to save you from boilerplate*.

```groovy
dependencies {
	// At the moment there is annotation processor only for StorIOSQLite 
	compile 'com.pushtorefresh.storio:sqlite-annotations:insert-latest-version-here'

	// We recommend to use Android Gradle Apt plugin: https://bitbucket.org/hvisser/android-apt
	apt 'com.pushtorefresh.storio:sqlite-annotations-processor:insert-latest-version-here'
}
```

```java
@StorIOSQLiteType(table = "tweets")
public class Tweet {
	
	// annotated fields should have package-level visibility
	@StorIOSQLiteColumn(name = "author")
	String author;

	@StorIOSQLiteColumn(name = "content")
	String content;

    // please leave default constructor with package-level visibility
	Tweet() {}
}
```

Annotation Processor will generate three classes in same package as annotated class during compilation:

* `TweetStorIOSQLitePutResolver`
* `TweetStorIOSQLiteGetResolver`
* `TweetStorIOSQLiteDeleteResolver`

You just need to apply them:

```java
StorIOSQLite storIOSQLite = DefaultStorIOSQLite.builder()
  .sqliteOpenHelper(someSQLiteOpenHelper)
  .addTypeMapping(Tweet.class, SQLiteTypeMapping.<Tweet>builder()
    .putResolver(new TweetStorIOSQLitePutResolver()) // object that knows how to perform Put Operation (insert or update)
    .getResolver(new TweetStorIOSQLiteGetResolver()) // object that knows how to perform Get Operation
    .deleteResolver(new TweetStorIOSQLiteDeleteResolver())  // object that knows how to perform Delete Operation
    .build())
  .addTypeMapping(...)
  // other options
  .build(); // This instance of StorIOSQLite will know how to work with Tweet objects
```

BTW: [Here is a class](storio-sample-app/src/main/java/com/pushtorefresh/storio/sample/db/entities/AllSupportedTypes.java) with all types of fields, supported by StorIO SQLite Annotation Processor.

Few tips about Operation Resolvers:

* If your entities are immutable or they have builders or they use AutoValue/AutoParcel -> write your own Operation Resolvers
* If you want to write your own Operation Resolver -> take a look at Default Operation resolvers, they can fit your needs
* Via custom Operation Resolvers you can implement any Operation as you want -> store one object in multiple tables, use custom sql things and so on

API of `StorIOContentResolver` is same.

----

####Documentation:

* [`Why we made StorIO`](https://engineering.pushtorefresh.com/2015/07/02/storio-modern-replacement-for-sqlitedatabase-and-contentresolver-apis/)
* [`StorIO SQLite`](docs/StorIOSQLite.md)
* [`StorIO ContentResolver`](docs/StorIOContentResolver.md)

Easy ways to learn how to use `StorIO` -> check out `Design Tests` and `Sample App`:

* [Design tests for StorIO SQLite](storio-sqlite/src/test/java/com/pushtorefresh/storio/sqlite/design)
* [Design tests for StorIO ContentResolver](storio-content-resolver/src/test/java/com/pushtorefresh/storio/contentresolver/design)
* [Sample App](storio-sample-app)

----

####Versioning:
Because StorIO works with important things like User data and so on, we use Semantic Versioning 2.0.0 scheme for releases (http://semver.org).

Short example:
`1.2.3` -> `MAJOR.MINOR.PATCH`

* `MAJOR` version changes when we make incompatible API changes.
* `MINOR` version changes when we add functionality in a backwards-compatible manner.
* `PATCH` version changes when we make backwards-compatible bug fixes.

Please read [`CHANGELOG`](CHANGELOG.md) and check what part of the version has changed, before switching to new version.

####Architecture:
`StorIOSQLite` and `StorIOContentResolver` — are abstractions with default implementations: `DefaultStorIOSQLite` and `DefaultStorIOContentResolver`.

It means, that you can have your own implementation of `StorIOSQLite` and `StorIOContentResolver` with custom behavior, such as memory caching, verbose logging and so on or mock implementation for unit testing (we are working on `MockStorIO`).

One of the main goals of `StorIO` — clean API for Humans which will be easy to use and understand, that's why `StorIOSQLite` and `StorIOContentResolver` have just several methods, but we understand that sometimes you need to go under the hood and `StorIO` allows you to do it: `StorIOSQLite.Internal` and `StorIOContentResolver.Internal` encapsulates low-level methods, you can use them if you need, but please try to avoid it.

####Queries

All `Query` objects are immutable, you can share them safely.

####Concept of Prepared Operations
You may notice that each Operation (Get, Put, Delete) should be prepared with `prepare()`. `StorIO` has an entity called `PreparedOperation<T>`, and you can use them to perform group execution of several Prepared Operations or provide `PreparedOperation<T>` as a return type of your API (for example in Model layer) and client will decide how to execute it: `executeAsBlocking()` or `asRxObservable()`. Also, Prepared Operations might be useful for ORMs based on `StorIO`.

You can customize behavior of every Operation via `Resolvers`: `GetResolver`, `PutResolver`, `DeleteResolver`.

####Rx Support Design
Every Operation can be executed as `rx.Observable`, `rx.Single` or `rx.Completable`. Get Operations will be automatically subscribed to the updates of the data.
Every Observable runs on `Schedulers.io()`, in v2.0 we will remove default scheduling!

####3rd party additions/integrations for StorIO

* [CodeGenUnderStorIO](https://github.com/shivan42/CodeGenUnderStorIO) allows you generate Java classes for db entities from the db schema built in some visual editor.

----
Master branch build status: [![Master branch build status](https://travis-ci.org/pushtorefresh/storio.svg?branch=master)](https://travis-ci.org/pushtorefresh/storio)


**Made with love** in [Pushtorefresh.com](https://pushtorefresh.com) by [@artem_zin](https://twitter.com/artem_zin) and [@nikitin-da](https://github.com/nikitin-da)
