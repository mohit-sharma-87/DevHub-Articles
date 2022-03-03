# How to migrate from Realm Java SDK to Realm Kotlin SDK

> This article is targeted to existing Realm developers who want to understand how to migrate to
> Realm Kotlin SDK.

## Introduction

Android has changed a lot in recent years notably after the Kotlin language became a first-class
citizen, so does the Realm SDK. Realm has recently moved its much-awaited Kotlin SDK to beta
enabling developers to use Realm more fluently with Kotlin and opening doors to the world of Kotlin
Multiplatform.

Let's understand the changes required when you migrate from Java to Kotlin SDK starting from setup
till its usage.

## Changes in setup

The Realm Kotlin SDK supports the Kotlin Multiplatform architecture which enables you to have one
common module for all your data needs for all platforms. But this doesn't mean you have to convert
your app to KMM to use it, you can also use it directly in your Android app, and then later move to
KMM.

Let's understand the changes needed in the Gradle file to use Realm Kotlin SDK, by comparing the
previous implementation with the new one.

In the project level `build.gradle`

Earlier with the Java SDK

```kotlin
    buildscript {

    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.3"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.4.31"

        // Realm Plugin 
        classpath "io.realm:realm-gradle-plugin:10.10.1"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

With the Kotlin SDK, we can **delete the Realm plugin** from `dependencies`

```kotlin
    buildscript {

    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.3"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.4.31"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

In the module-level `build.gradle`

With Java SDK, we

1. Enabled Realm Plugin
2. Enabled Sync, if applicable

```groovy

plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
    id 'realm-android'
}
```

```groovy
    android {

    ... ....

    realm {
        syncEnabled = true
    }
}
```

With Kotlin SDK,

1. Replace ``id 'realm-android'`` with  ``id("io.realm.kotlin") version "0.10.0"``

```groovy
    plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
    id("io.realm.kotlin") version "0.10.0"
}
```

2. Remove the Realm block under android tag

```groovy
    android {
    ... ....

}
```

3. Add Realm dependency under `dependencies` tag

```groovy
    dependencies {

    implementation("io.realm.kotlin:library-sync:0.10.0")

}
```

> If you are using only Realm local SDK, then you can add
> ```groovy
> dependencies {
>    implementation("io.realm.kotlin:library-base:0.10.0")
> }
>```

With these changes, our Android app is ready to use Kotlin SDK.

## Changes in implementation

### Realm Initialization

Traditionally before using Realm for querying information in our project, we had to initialize and
set up few basic properties like name, version with sync config for database, let's update them as
well.

Steps with JAVA SDK :

1. Call `Realm.init()`
2. Setup Realm DB properties like name, version, migration rules etc using `RealmConfiguration`.
3. Setup logging
4. Configure Realm Sync

With Kotlin SDK :

1. <s>Call `Realm.init()`</s> _Is not needed anymore as this is done internally now_
2. Setup Realm DB properties like db name, version, migration rules etc. using `RealmConfiguration`-
   _This remains the same apart from a few minor changes_.
3. Setup logging - _This is moved to `RealmConfiguration`_
4. Configure Realm Sync - _No changes_

### Changes to Models

No major changes are required in model classes. Models classes are now expected to implement 
`RealmObject` interface which was a base class earlier.

Apart from that you might have to remove some currently unsupported type like `@RealmClass`
,`ByteArray`, `RealmDictionary`, `RealmSet`, `ObjectId`, `Decimal128`,`UUID`, `@LinkingObjects`, 
which would be available in future releases soon.

> Note: Due to the above changes `Open` keyword against `class`, no longer required. 

### Changes to querying

The most exciting part starts from here 😎(IMO).

Traditionally Realm SDK has been on the top of the latest programming trends like Reactive
programming (Rx), LiveData and many more but with the technological shift in Android programming
language from Java to Kotlin, developers were not able to fully utilize the power of the language
with Realm as underlying SDK was still in Java, few of the notable were support for the Coroutines,
Kotlin Flow, etc.

But with the Kotlin SDK that all has changed and further led to the reduction of boiler code. Let's
understand these by examples.

Example 1: As a user, I would like to register my visit as soon as I open the app or screen.

Steps to complete this operation would be

1. Authenticate with Realm SDK.
2. Based on the user information, create a sync config with the partition key.
3. Open Realm instance.
4. Start a Realm Transaction.
5. Query for current user visit count and based on that add/update count.

With JAVA SDK:

```kotlin
private fun updateData() {
    _isLoading.postValue(true)

    fun onUserSuccess(user: User) {
        val config = SyncConfiguration.Builder(user, user.id).build()

        Realm.getInstanceAsync(config, object : Realm.Callback() {
            override fun onSuccess(realm: Realm) {
                realm.executeTransactionAsync {
                    var visitInfo = it.where(VisitInfo::class.java).findFirst()
                    visitInfo = visitInfo?.updateCount() ?: VisitInfo().apply {
                        partition = user.id
                    }.updateCount()

                    it.copyToRealmOrUpdate(visitInfo).apply {
                        _visitInfo.postValue(it.copyFromRealm(this))
                    }
                    _isLoading.postValue(false)
                }
            }

            override fun onError(exception: Throwable) {
                super.onError(exception)
                // some error handling 
                _isLoading.postValue(false)
            }
        })
    }

    realmApp.loginAsync(Credentials.anonymous()) {
        if (it.isSuccess) {
            onUserSuccess(it.get())
        } else {
            _isLoading.postValue(false)
        }
    }
}
```

With Kotlin SDK:

```kotlin
private fun updateData() {
    viewModelScope.launch(Dispatchers.IO) {
        _isLoading.postValue(true)
        val user = realmApp.login(Credentials.anonymous())
        val config = SyncConfiguration.Builder(
            user = user,
            partitionValue = user.identity,
            schema = setOf(VisitInfo::class)
        ).build()

        val realm = Realm.open(configuration = config)
        realm.write {
            val visitInfo = this.query<VisitInfo>().first().find()
            copyToRealm(visitInfo?.updateCount()
                ?: VisitInfo().apply {
                    partition = user.identity
                    visitCount = 1
                })
        }
        _isLoading.postValue(false)
    }
}
```

Upon quick comparing, you would notice that lines of code have decreased by 30%, and we are using
coroutines for doing the async call, which is the natural way of doing asynchronous programming in
Kotlin. Let's check this with one more example.

Example 2: As user, I should be notified immediately about any change in user visit info. This is
more like observing the change to visit count.

With Java SDK:

```kotlin
 fun onRefreshCount() {
    _isLoading.postValue(true)

    fun getUpdatedCount(realm: Realm) {
        val visitInfo = realm.where(VisitInfo::class.java).findFirst()
        visitInfo?.let {
            _visitInfo.value = it
            _isLoading.postValue(false)
        }
    }

    fun onUserSuccess(user: User) {
        val config = SyncConfiguration.Builder(user, user.id).build()

        Realm.getInstanceAsync(config, object : Realm.Callback() {
            override fun onSuccess(realm: Realm) {
                getUpdatedCount(realm)
            }

            override fun onError(exception: Throwable) {
                super.onError(exception)
                //TODO: Implementation pending
                _isLoading.postValue(false)
            }
        })
    }

    realmApp.loginAsync(Credentials.anonymous()) {
        if (it.isSuccess) {
            onUserSuccess(it.get())
        } else {
            _isLoading.postValue(false)
        }
    }
}
```

With Kotlin SDK :

```kotlin

fun onRefreshCount(): Flow<VisitInfo?> {

    val user = runBlocking { realmApp.login(Credentials.anonymous()) }
    val config = SyncConfiguration.Builder(
        user = user,
        partitionValue = user.identity,
        schema = setOf(VisitInfo::class)
    ).build()

    val realm = Realm.open(config)
    return realm.query<VisitInfo>().first().asFlow()
}

```

Again upon quick comparing you would notice that lines of code have decreased drastically, by more
than **60%**, and apart coroutines for doing async call, we are using _Kotlin Flow_ to observe the
value changes.

With this, as mentioned earlier, we are further able to reduce our boilerplate code,
no [callback hell](http://callbackhell.com) and writing code is more natural now.

## Other major changes

Apart from Realm Kotlin SDK being written in Kotlin language, it is fundamentally little different
from the JAVA SDK in a few ways:

- **Frozen by default**: All objects are now frozen. Unlike live objects, frozen objects do not
  automatically update after the database writes. You can still access live objects within a write
  transaction, but passing a live object out of a write transaction freezes the object.
- **Thread-safety**: All realm instances, objects, query results, and collections can now be
  transferred across threads.
- **Singleton**: You now only need one instance of each realm.

## Should you migrate now?

There is no straight answer to question, it really depends on usage, complexity of the app and time.
But I think so this the perfect time to evaluate the efforts and changes required to migrate as
Realm Kotlin SDK would be the future.


