---
id: configdescriptor_index
title:  "Manual creation of ConfigDescriptor"
---

Config Descriptor is the core of your configuration management. You can write a description by hand,
or rely on zio-config-magnolia of zio-config-shapeless that can automatically generate the description for you, based on the case classes (or sealed traits)
that represents your config.

```scala mdoc:silent
import zio.{ ZIO, IO, Layer }
import zio.config._, ConfigDescriptor._, ConfigSource._
```

## A Simple example

We must fetch the configuration from the environment to a case class (product) in scala. Let it be `MyConfig`

```scala mdoc:silent

case class MyConfig(ldap: String, port: Int, dburl: String)

```
To perform any action using zio-config, we need a configuration description.
Let's define a simple one.


```scala mdoc:silent
val myConfig: ConfigDescriptor[MyConfig] =
  (string("LDAP") zip int("PORT") zip string("DB_URL")).to[MyConfig]

```


Case classes with a single field are simple too.

```scala mdoc:silent
case class MySingleConfig(ldap: String)

val mySingleConfig: ConfigDescriptor[MySingleConfig] =
  string("LDAP").to[MySingleConfig]
```

If the config is not a case class, but a tuple, then all you need to do is `zip`

```scala mdoc:silent

val mySingleConfigTupled: ConfigDescriptor[(String, Int)] =
  (string("LDAP") zip int("PORT"))

```

Think of this as removing fields one-by-one, along with the `zip` combinator syntax, ending up with a single field being applied.

## Fully Automated Config Description: zio-config-magnolia

If you don't like describing your configuration manually, and rely on the names of the parameter in the case class (or sealed trait),
there is a separate module called `zio-config-magnolia` which uses `Magnolia` library to automatically derive the descriptions for you.

```scala mdoc:silent

import zio.config.magnolia._

val myConfigAutomatic = descriptor[MyConfig]

```

`myConfig` and `myConfigAutomatic` are same description, and is of the same type.

If you need more control over the description,
probably you may choose to write it manually (such as, for adding extra documentations).
More examples on automatic derivation is in examples module of [zio-config](https://github.com/zio/zio-config)

## Running the description to ZIO

To read a config, means it has to perform some effects, and for that reason, it returns with a ZIO.
To be specific it returns an `IO` where `type IO[E, A] = ZIO[Any, E, A]`

```scala mdoc:silent
import zio.System

// That's system environment
val result: Layer[ReadError[String], MyConfig] = ZConfig.fromSystemEnv(myConfig)
```

Another way of doing this is:

```scala mdoc:silent
val systemSource = ConfigSource.fromSystemEnv()

read(myConfig from systemSource)
```

You can run this to [completion](https://zio.dev/docs/getting_started.html#main) as in any zio application.

We will not be discussing about running with ZIO again, as it is just the same regardless of what the description is.
We will discuss only about how to describe your configuration for the rest of this page.

## Built-in types

We have already seen `string("TOKEN")` and `int("PORT")` to fetch string and int types respectively.
We support the following:

```scala
string
boolean
byte
short
int
long
bigInt
float
double
bigDecimal
uri
uuid
localDate
localTime
localDateTime
instant
file
url
etc

```

Complex types include `list`, `map` etc. More details to follow

## Optional Types

Say, dburl is an optional type, then it is as simple as

```scala mdoc:silent
string("DB_URL").optional
```

That is,

```scala mdoc:silent
case class MyConfigWithOptionalUrl(ldap: String, port: Port, dburl: Option[String])

val myConfigOptional =
  (string("LDAP") zip int("PORT").to[Port] zip
    string("DB_URL").optional).to[MyConfigWithOptionalUrl]

```

## Default
Sometimes, we don't need an optional value and instead happy providing a default value.

```scala mdoc:silent

 val defaultConfig =
  string("USERNAME").default("ROOT")

```

 That is,

```scala mdoc:silent

case class MyConfigWithDefaultUserName(username: String, port: Int)

val myConfigDefault =
  (string("USERNAME").default("root-oh") zip int("PORT")).to[MyConfigWithDefaultUserName]

```

We can also do things like fully overriding the entire configuration; might be helpful for tests.

```scala mdoc:silent
myConfigDefault.default(MyConfigWithDefaultUserName("test", 80))
```

## New types
We love `Port` instead of `Int` that represents a db port.

In this scenario, you could do

```scala mdoc:silent

int("PORT").to[Port]

```

where port is;

```scala mdoc:silent
 case class Port(value: Int)

```

That is,

```scala mdoc:silent

 case class MyCustomConfig(ldap: String, port: Port, dburl: String)

 // Before
  val myConfigWithCustomType =
    (string("LDAP") zip int("PORT").to[Port] zip
      string("DB_URL")).to[MyCustomConfig]

```

## Multiple sources

While it may not be always a good idea having to rely on multiple sources to form the application config,
zio-config do support this scenario. This can happen in complex applications.


```scala mdoc:silent


val source1 = ConfigSource.fromSystemProps()
val source2 = ConfigSource.fromSystemEnv()
 
val configDesc =
  (string("LDAP").from(source1.orElse(source2)) zip int("PORT").to[Port].from(source1) zip
    string("DB_URL").optional.from(source2)).to[MyConfigWithOptionalUrl]


read(configDesc)

// we can also separately add new config
read(configDesc from ConfigSource.fromMap(Map.empty))

// In this case, `ConfigSource.fromMap` will also be tried along with the sources that are already given.

```

We can reset the sources for the config using

```scala mdoc:silent

configDesc.unsourced

```

By that way, in tests we could remove the sources from each parameter and ask it to get it
from a constant map for all of it.

```scala mdoc:silent

val testConfig =
  configDesc.unsourced from ConfigSource.fromMap(Map("LDAP" -> "x", "DB_URL" -> "y",  "PORT" -> "1235"))

```

## Combining various effectful sources

There are various ways you can interact with zio-config when dealing with multiple sources.
Below given is a naive example, that will help you get a gist of how to get config values from various
sources, especially when some of the sources returns ZIO.

```scala mdoc:silent
import java.io.File

import zio._
import zio.config._
import zio.config.typesafe._

/**
 * One of the ways you can summon various sources especially
 * when some of the `fromSource` functions return ZIO.
 */
object CombineSourcesExample extends ZIOAppDefault {
  override def run =
    application.either.flatMap(r => Console.printLine(s"Result: ${r}"))

  case class Config(username: String , password: String)

  val desc: ConfigDescriptor[Config] = {
    val hoconFile = TypesafeConfigSource.fromHoconFile(new File("/invalid/path"))
    val constant  = TypesafeConfigSource.fromHoconString(s"")
    val env       = ConfigSource.fromSystemEnv()
    val sysProp   = ConfigSource.fromSystemProps()
    val source    = hoconFile <> constant <> env <> sysProp
    (descriptor[Config] from source)
  }

  val application =
    for {
      configValue <- read(desc).mapError(_.prettyPrint())
      string      <- ZIO.fromEither(configValue.toJson(desc))
      _           <- Console.printLine(string)
    } yield ()
}

```

## Either Types (orElseEither)

For instance, if we are ok accepting either a token or username, then our target type should be `Either[String, String]`.
In this case, We can use `orElseEither` or `<+>`.

```scala mdoc:silent
string("USERNAME").orElseEither(string("TOKEN"))
```

That is,

```scala mdoc:silent
case class MyConfigWithEither(usernameOrToken: Either[String, String], port: Int)

val myConfigEither =
  (string("USERNAME").orElseEither(string("TOKEN")) zip int("PORT")).to[MyConfigWithEither]

```

We can also use `<+>` combinator.

```scala mdoc:silent
 string("USERNAME") <+> (string("TOKEN"))
```

We can apply the `Either` logic at a much more global level, as in, give me either a `Prod` or `Test` config.

```scala mdoc:silent
case class Dev(userName: String, password: String)
case class Prod(token: String, code: Int)

type ZConfig = Either[Prod, Dev]

val dev = (string("USERNAME") zip string("PASSWORD")).to[Dev]
val prod = (string("TOKEN") zip int("CODE")).to[Prod]

prod <+> dev // that represents a description returning Config
// ConfigDescriptor[ Config]

```

`orElseEither` works with complex hocon sources, which is an additional benefit compared to existing configuration libraries.
You can also choose to avoid a `sealed trait` encoding if all we need is `Either[Int, String]`, for instance.

## OrElse

Sometimes, we can try two different values and pick one. That means, the target is "NOT" `Either` but any raw type.
In this scenario, We can use `orElse` or `<>`

```scala mdoc:silent

string("TOKEN") orElse string("token")

```

Example:

```scala mdoc:silent
val configOrElse =
  (string("TOKEN").orElse(string("token_x")) zip int("CODE")).to[Prod]

```

It tries to fetch the value corresponding to "TOKEN", and if it fails, it tries "token_x" and returns the corresponding value.

We can also use `<>` combinator.

```scala mdoc:silent
string("TOKEN") <> string("token") <> string("TOKEN_INFO")
```

## Composing multiple configurations

This is more of a real life scenario, where you can different micro configurations for readability and maintainability.

```scala mdoc:silent
  case class Database(url: String, port: Int)
  case class AwsConfig(c1: Database, c3: String)


  val databaseConfig =
    (string("connection") zip int("port")).to[Database]

  (databaseConfig zip string("c3")).to[AwsConfig]

```

## Nesting

In addition to the primitive types, zio-config provides a combinator for nesting a configuration within another.

This might not feel intuitive at first, however, zio-config is designed to easily adapt to
any other configuration parsing libraries that deal with file formats such as HOCON that natively support nested configurations.

```scala mdoc:silent
  case class AwsConfigExtended(c1: Database, c2: Database, c3: String)

  val appConfig =
    (nested("south") { databaseConfig } zip
      nested("east") { databaseConfig } zip
      string("appName")).to[AwsConfigExtended]

  // Let's say, a nested configuration as a flattened map is just "." delimited keys in a flat map.
  val constantMap =
    Map(
      "south.connection" -> "abc.com",
      "east.connection"  -> "xyz.com",
      "east.port"        -> "8888",
      "south.port"       -> "8111",
      "appName"          -> "myApp"
    )

  ZConfig.fromMap(constantMap, appConfig)
```

Note that, you can write this back as well. This is discussed in write section

## CollectAll (Sequence)


```scala mdoc:silent
 def database(i: Int) =
   (string(s"${i}_URL") zip int(s"${i}_PORT")).to[Database]

 val list: ConfigDescriptor[List[Database]] =
   collectAll(database(0), (1 to 10).map(database): _*)

```
Running this to ZIO will result in non empty list of database

NOTE: `collectAll` is a synonym for `sequence`.

## Handling list is just easy!

```scala

  final case class PgmConfig(a: String, b: List[String])

  val configWithList =
    (string("xyz") zip list("regions")(string)).to[PgmConfig]


  Config.fromEnv(configWithList, valueDelimiter = Some(","))
  // or read(configWithList from ConfigSource.fromEnv(valueDelimiter = Some(",")))

```

List is probably better represented in HOCON files.
zio-config-typesafe enables you to depend on HOCON files to manage your configuration.

Given;

```scala

val listHocon = """
    accounts = [
      {
         region : us-east
         accountId: jon
      }
      {
         region : us-west
         accountId: chris
      }
    ]
    database {
        port : 100
        url  : postgres
    }
  """

```

```scala

import zio.config.typesafe.TypesafeConfigSource._
import zio.config.magnolia._

  // A nested example with type safe config, and usage of magnolia
final case class Accnt(region: String, accountId: String)
final case class Db(port: Int, url: String)
final case class AwsDetails(accounts: List[Accnt], database: Db)

val autoListConfig = descriptor[AwsDetails]

read(autoListConfig from hocon(listHocon))

  // yields
  //  AwsDetails(
  //    List(Accnt("us-east", "jon"), Accnt("us-west", "chris")),
  //    Db(100, "postgres")
  //  )

```

Note that `autoListConfig` (automatically generated) config, is exactly similar to:

```scala

  val accnt =
    (string("region") zip string("accountId")).to[Accnt]

  val db = (int("port") zip string("url")).to[Db]

  val nonAutomatic =
    (nested("accounts")(list(accnt)) zip nested("database")(db)).to[AwsDetails]

```

Please find more details on the behaviour of `List` for various sources in `Sources` section of the documentation.