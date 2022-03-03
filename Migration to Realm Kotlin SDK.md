# How to migrate from Realm Java SDK to Realm Kotlin SDK

> This article is targeted to existing Realm developers who want to understand how to migrate to
> Realm Kotlin SDK.

## Introduction

Android has changed a lot in recent years, notably after Kotlin language as first class citizen, so
does the Realm SDK. Realm has recently moved it's much awaited Kotlin SDK to beta enabling to use
Realm more fluently with Kotlin and opening doors of Kotlin Multiplatform to them.

Let's understand the changes required when you migrate from Java SDK to Kotlin starting from setup
till usage.

## Changes in setup

The new Realm Kotlin SDK is based on Kotlin Multiplatform architecture which enables you to have one
common module for all your data needs on all platforms. But this doesn't mean that to use the new
SDK you would have to convert your Android app to KMM app right away, you can do that later.

Let's understand the changes needed in the gradle file to use Realm Kotlin SDK, by comparing
previous implementation and new one.

In project level `build.gradle`

Earlier with Java SDK

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

With Kotlin SDK, we can **delete the plugin** configuration

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

With Java SDK, we added two properties

1. Apply Realm Plugin
2. Enable Sync

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

1. Replace ``id 'realm-android'`` with  ``id("io.realm.kotlin") version "0.9.0"``

```groovy
    plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
    id("io.realm.kotlin") version "0.9.0"
}
```

2. Remove the Realm block under android

```groovy
    android {

    ... ....

}
```

3. Add Realm dependency under dependency tag

```groovy
dependencies {

    implementation("io.realm.kotlin:library-sync:0.9.0")

}
```

> If you are using only Realm mobile SDK, then you can add
> ```groovy
> dependencies {
>    implementation("io.realm.kotlin:library-base:0.9.0")
> }
>```

With this, our Android app is ready to use the new Realm Kotlin SDK.

## Changes in implementation

### Realm Initialization

Traditionally before using Realm for querying in our project, we have to set up or initialize a few
basic properties of our database like name, version, sync config etc. Let's understand the impact on
these.

With JAVA SDK :

1. Call `Realm.init()`
2. Setup Realm DB properties like db name, version, migration rules etc using `RealmConfiguration`.
3. Setup logging
4. Configure Realm Sync

With Kotlin SDK :

1. <s>Call `Realm.init()`</s> _Is not needed anymore_.
2. Setup Realm DB properties like db name, version, migration rules etc using `RealmConfiguration` -
   _Remain the same apart from a few minor changes_.
3. Setup logging - _This is moved to `RealmConfiguration`_
4. Configure Realm Sync - _No changes_

### Changes to Models

No changes are needed to model class for using Kotlin SDK, except you might have to remove few
currently unsupported annotation like `@RealmClass` for the embedded object.

> Note: You can remove `Open` keyword against `class` which was mandatory for using Java SDK in
> Kotlin.

### Changes on how to Query

The most exciting part starts from here 😎(IMO).

Traditionally Realm SDK has been on the top of the latest programming treads like Reactive
programming (Rx), LiveData and many more but with the technological shift in Android programming
language from Java to Kotlin, developers were not able to fully utilize the power the language and
Realm SDK as underlying SDK was still in Java. A few of such were Coroutines, Kotlin Flow, etc.

But with the new Kotlin SDK that is all changed and further reduced the boiler code. Let's see
understand by example.

Use-case: As a user, I would like to register my visit as soon as I open the app or screen.

Steps to complete the operation would be

1. Login into the system.
2. Based on user info create a sync config with the corresponding partition key.
3. Open Realm instance
4. Get Realm Transaction instance
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

Upon quick comparing, you would notice that lines of code have decreased by 30% and we are using
coroutines for doing the async call, which is the natural way of doing asynchronous programming in
Kotlin. Let's check this with one more example.

Use-case: As user, I should be notified immediately about any change in user visit info. This is
more like observing the change in the value of visit count.

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
than
**50%**, and again using coroutines for doing async call and _Kotlin Flow_ to observe the value
changes.

With this, as mentioned earlier, we are further able to reduce our boilerplate code,
no [callback hell](http://callbackhell.com) and writing code is more natural now.

## Other major changes

Realm Kotlin SDK is fundamentally little different from JAVA SDK in a few ways:

- **Frozen by default**: All objects are now frozen. Unlike live objects, frozen objects do not
  automatically update after the database writes. You can still access live objects within a write
  transaction, but passing a live object out of a write transaction freezes the object.
- **Thread-safety**: All realm instances, objects, query results, and collections can now be
  transferred across threads.
- **Singleton**: You now only need one instance of each realm.

## Should you migrate now?

## Non supported features

