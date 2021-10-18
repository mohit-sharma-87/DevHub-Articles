# Migrating Realm SDK database - Local Database

> This is followup article in **Getting Started Series**. In this article, we learn how to
> modify/migrate realm **local only** database schema.

As you add and change application features, you need to modify database schema and the need for
migrations arises, which is very important for a seamless user experience. By the end of this
article, you will learn

1. How to update database schema post-production release on play store.
2. How to migrate user data from one schema to another.

Before we jump down to business, let's quickly recap how we set `Realm` in our application.

```kotlin
const val REALM_SCHEMA_VERSION: Long = 1
const val REALM_DB_NAME = "rMigrationSample.db"

fun setupRealm(context: Context) {
    Realm.init(context)

    val config = RealmConfiguration.Builder()
        .name(REALM_DB_NAME)
        .schemaVersion(REALM_SCHEMA_VERSION)
        .build()

    Realm.setDefaultConfiguration(config)
}
```

Doing migration in Realm is very straightforward and simple. For any database migration high level
steps for successful migration would be

1. Update the database version.
2. Make changes to the database schema.
3. Migrate user data from old schema to new.

### Update the database version

This is the simplest step, which can be done by incrementing the version of
`REALM_SCHEMA_VERSION` which notify `Relam` about database changes which in turn run triggers
migration if provided.

To add migration we use `migration` function available in `RealmConfiguration.Builder`, which take
an argument of `RealmMigration`, which we would review in the next step.

```kotlin
val config = RealmConfiguration.Builder()
    .name(REALM_DB_NAME)
    .schemaVersion(REALM_SCHEMA_VERSION)
    .migration(DBMigrationScript())
    .build()
```

### Make changes to the database schema

In `Realm` all the migration-related operation has to perform within the scope of `RealmMigration`.

```kotlin
class DBMigrationScript : RealmMigration {

    override fun migrate(realm: DynamicRealm, oldVersion: Long, newVersion: Long) {
        migration1to2(realm.schema)
        migration2to3(realm.schema)
        migration3to4(realm.schema)
    }

    private fun migration3to4(schema: RealmSchema?) {
        TODO("Not yet implemented")
    }

    private fun migration2to3(schema: RealmSchema?) {
        TODO("Not yet implemented")
    }

    private fun migration1to2(schema: RealmSchema) {
        TODO("Not yet implemented")
    }
}
```

To Add / Update / Rename any field

```kotlin

private fun migration1to2(schema: RealmSchema) {
    val userSchema = schema.get(UserInfo::class.java.simpleName)
    userSchema?.run {
        addField("phoneNumber", String::class.java, FieldAttribute.REQUIRED)
        renameField("phoneNumber", "phoneNo")
        removeField("phoneNo")
    }
}
```

### Migrate user data from old schema to new

All the data transformation during migration can be done with `transform` function

```kotlin

private fun migration2to3(schema: RealmSchema) {
    val userSchema = schema.get(UserInfo::class.java.simpleName)
    userSchema?.run {
        addField("fullName", String::class.java, FieldAttribute.REQUIRED)
        transform {
            it.set("fullName", it.get<String>("firstName") + it.get<String>("lastName"))
        }
    }
}
```

We can also use `transform` to update the data type as well

```kotlin

val personSchema = schema!!["Person"]

// Change type from String to int
schema["Pet"]?.run {
    addField("type_tmp", Int::class.java)
    transform { obj ->
        val oldType = obj.getString("type")
        if (oldType == "dog") {
            obj.setLong("type_tmp", 1)
        } else if (oldType == "cat") {
            obj.setInt("type_tmp", 2)
        } else if (oldType == "hamster") {
            obj.setInt("type_tmp", 3)
        }
    }
    removeField("type")
    renameField("type_tmp", "type")
}
```

In case, you want to delete the complete `Realm` you can use `deleteRealmIfMigrationNeeded` with
`RealmConfiguration.Builder`, but that should be considered as the last resort.

Thank you for reading, in the next article we would discuss how to migrate `Realm` database with
sync. 

You can also check out the iOS tutorial
for [Realm iOS Migration](https://www.mongodb.com/developer/how-to/realm-schema-migration/). 




