# PureConfig
A boilerplate-free Scala library for loading configuration files

[![Build Status](https://travis-ci.org/melrief/pureconfig.svg?branch=master)](https://travis-ci.org/melrief/pureconfig)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.melrief/pureconfig_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.github.melrief/pureconfig_2.11)
[![Join the chat at https://gitter.im/melrief/pureconfig](https://badges.gitter.im/melrief/pureconfig.svg)](https://gitter.im/melrief/pureconfig?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

![](http://i.imgur.com/S5QUS8c.gif)

## Table of Contents
- [Why](#why)
- [Not Yet Another Configuration Library](#not-yet-another-configuration-library)
- [Add PureConfig to your project](#add-pureconfig-to-your-project)
- [Use PureConfig](#use-pureconfig)
- [Supported types](#supported-types)
- [Customizing naming conventions](#customizing-naming-conventions)
- [Extend the library to support new types](#extend-the-library-to-support-new-types)
- [Override behaviour for types](#override-behaviour-for-types)
- [Override behaviour for sealed families](#override-behaviour-for-sealed-families)
- [Handling missing keys](#handling-missing-keys)
- [Example](#example)
- [Whence the config files](#whence-the-config-files)
- [Contribute](#contribute)
- [License](#license)
- [Special thanks](#special-thanks)


## Why
Loading configurations has always been a tedious and error-prone procedure. A common way to do it
consists in writing code to deserialize each fields of the configuration. The more fields there are,
the more code must be written (and tested and maintained...) and this must be replicated for each project.

This kind of code is boilerplate because most of the times the code can be automatically generated by
the compiler based on what must be loaded. For instance, if you are going to load an `Int` for a field
named `foo`, then probably you want some code that gets the values associated with the key `foo` in
the configuration and assigns it to the proper field after converting it to `Int`.

The goal of this library is to create at compile-time the boilerplate necessary to load a configuration of a
certain type. In other words, you define **what** to load and PureConfig provides **how** to load it.


## Not yet another configuration library
PureConfig is not a configuration library in the sense that it doesn't search for files or parse them.
It can be seen as a better front-end for the existing libraries.
It uses [typesafe config][typesafe-config] library for loading raw configurations and then
uses the raw configurations to do its magic.


## Add PureConfig to your project

In the sbt configuration file:

use scala `2.10`, `2.11` or `2.12`:

```scala
scalaVersion := "2.12.1" // or "2.11.8", "2.10.5"
```

Add the library. For scala `2.11` and `2.12`

```scala
libraryDependencies ++= Seq(
  "com.github.melrief" %% "pureconfig" % "0.5.0"
)
```

For scala `2.10` you need also the scala macro paradise plugin:

```scala
libraryDependencies ++= Seq(
  "com.github.melrief" %% "pureconfig" % "0.5.0",
compilerPlugin("org.scalamacros" % "paradise" % "2.0.1" cross CrossVersion.full)
)
```



For a full example of `build.sbt` you can have a look at this [build.sbt](https://github.com/melrief/pureconfig/blob/master/example/build.sbt)
used for the example.


## Use PureConfig

Import the library and use one of the `loadConfig` methods:


```scala
import pureconfig._

val config: Try[YourConfClass] = loadConfig[YourConfClass]
```


## Supported Types

Currently supported types for fields are:
- `String`, `Boolean`, `Double` (standard
and percentage format ending with `%`), `Float` (also supporting percentage),
`Int`, `Long`, `Short`, `URL`, `Duration`, `FiniteDuration`
- all collections implementing the `TraversableOnce` trait where the elements type
is in this list
- `Option` for optional values, i.e. value that can or cannot be in the configuration
- `Map` with `String` keys and any value type that is in this list
- typesafe `ConfigValue`, `ConfigObject` and `ConfigList`
- case classes
- sealed families of case classes (ADTs)

An almost comprehensive example is:

```scala
> import pureconfig.loadConfig
> import com.typesafe.config.ConfigFactory.parseString
> sealed trait MyAdt
> case class AdtA(a: String) extends MyAdt
> case class AdtB(b: Int) extends MyAdt
> case class MyClass(int: Int, adt: MyAdt, list: List[Double], map: Map[String, String], option: Option[String])
> val conf = parseString("""{ "int": 1, "adt": { "type": "adtb", "b": 1 }, "list":["1", "20%"], "map": { "key": "value" } }""")
> loadConfig[MyClass](conf)
res0: util.Try[MyClass] = Success(MyClass(1,AdtB(1),List(1.0, 0.2),Map(key -> value),None))
```

## Customizing naming conventions

In case the naming convention you use in your configuration files differs from
the one used in the objects you're loading the configs into, PureConfig allows
you to define proper mappings. That configuration should be done by an implicit
`ConfigFieldMapping` that should be in scope when loading or writing
configurations. The `ConfigFieldMapping` trait has a single `apply` method that
maps field names in Scala objects to field names in the source configuration
file. For instance, here's a contrived example where the configuration file has
all keys in upper case and we're loading it into a type whose fields are all in
lower case:

```scala
case class SampleConf(foo: Int, bar: String)

implicit val mapping = new ConfigFieldMapping[SampleConf] {
  def apply(fieldName: String) = fieldName.toUpperCase
}

val conf = ConfigFactory.parseString("""{
  FOO = 2
  BAR = "two"
}""")

loadConfig[SampleConf](conf)
// returns Success(SampleConf(2, "two"))
```

PureConfig provides a way to create a `ConfigFieldMapping` by defining the
naming conventions of the fields in the Scala object and in the configuration
file. Some of the most used naming conventions are supported directly in the
library:

* [`CamelCase`](https://en.wikipedia.org/wiki/Camel_case): `camelCase`, `useMorePureconfig`
* [`SnakeCase`](https://en.wikipedia.org/wiki/Snake_case): `snake_case`, `use_more_pureconfig`
* [`KebabCase`](http://wiki.c2.com/?KebabCase): `kebab-case`, `use-more-pureconfig`
* [`PascalCase`](https://en.wikipedia.org/wiki/PascalCase): `PascalCase`, `UseMorePureconfig`

You can use the `apply` method of `ConfigFieldMapping` that accepts the two
naming conventions (for the fields in the Scala object and for the fields in the
configuration file, respectively). A common use case is to have your field names
in `camelCase` and your configuration files in `kebab-case`. In order to support
it, you can make sure the following implicit is in scope before loading or
writing configuration files:

```scala
implicit def conv[T] = ConfigFieldMapping.apply[T](CamelCase, KebabCase)
```

## Extend the library to support new types

Not all types are supported automatically by `pureconfig`. For instance, classes
that are not case classes are not supported out-of-the-box:

```scala
> class Foo(var bar: Int) { override def toString: String = s"Foo(${bar.toString})" }
> loadConfig[Foo]
could not find implicit value for parameter conv: pureconfig.ConfigConvert[Foo]
```

`pureconfig` can be extended to support those types. To do so, an instance for the
`ConfigConvert` typeclass must be provided implicitly, like

```scala
> implicit val foocc = ConfigConvert.stringConvert[Foo](s => Try(new Foo(s.toInt)), foo => foo.bar.toString)
> loadConfig[Foo]
Foo(1)
```


## Override behaviour for types

It is possible to override the behaviour of `pureconfig` for a certain type by
implementing another instance of the `ConfigConvert` typeclass. For instance,
the default behaviour of `pureconfig` for `String` is to return the string itself
in the configuration:

```scala
> import import com.typesafe.config.ConfigValueFactory
> ConfigConvert[String].from(ConfigValueFactory.fromAnyRef("FooBar"))
util.Try[String] = Success(FooBar)
```

Now let's say that we want to override this behaviour such that `String`s are
always read lower case. We can do:

```scala
> implicit val overridestrcc = ConfigConvert.fromString(s => Try(s.toLowerCase))
> ConfigConvert[String].from(ConfigValueFactory.fromAnyRef("FooBar"))
util.Try[String] = Success(foobar)
```

## Override behaviour for sealed families

In order for `pureconfig` to disambiguate between different options of a sealed
family of case classes, it must read and write additional information in
configurations. By default it uses the additional field `type`, encoding the
concrete class represented in the configuration:

```scala
sealed trait AnimalConf
case class DogConf(age: Int) extends AnimalConf
case class BirdConf(canFly: Boolean) extends AnimalConf

loadConfig[AnimalConf](parseString("""{ type: "dogconf", age: 4 }"""))
// returns Success(DogConf(4))
```

For sealed families, `pureconfig` provides a way to customize the conversion
without replacing the default `ConfigConvert`. By putting in scope an instance
of `CoproductHint` for that sealed family, we can customize how the
disambiguation is made. For example, if `type` clashes with one of the fields
of a case class option, we can use another field:

```scala
implicit val animalConfHint = new FieldCoproductHint[AnimalConf]("kind")
loadConfig[AnimalConf](parseString("""{ kind: "dogconf", age: 4 }"""))
// returns Success(DogConf(4))
```

`FieldCoproductHint` can also be adapted to write class names in a different
way:

```scala
implicit val animalConfHint = new FieldCoproductHint[AnimalConf]("type") {
  override def fieldValue(name: String) = name.dropRight("Conf".length)
}
loadConfig[AnimalConf](parseString("""{ type: "Bird", canFly: true }"""))
// returns Success(BirdConf(true))
```

With a `CoproductHint` you can even opt not to use any extra field at all. For
example, if you encode enumerations using sealed traits, you can just write the
name of the class:

```scala
import com.typesafe.config.{ConfigFactory,ConfigValue}
import pureconfig.syntax._
import scala.util.Success

sealed trait Season
case object Spring extends Season
case object Summer extends Season
case object Autumn extends Season
case object Winter extends Season

implicit val seasonHint = new CoproductHint[Season] {

  // Reads a config for Season (`cv`).
  // - If `name` is the name of the concrete season `cv` refers to, returns
  //   `Success(Some(conf))`, where `conf` is the config for the concrete class
  //   (in this case, an empty object).
  // - If `name` is not the name of the class for `cv`, returns
  //   `Success(None)`.
  // - If `cv` is an invalid config for Season (in this case, if it isn't a
  //   string), returns a `Failure`.
  def from(cv: ConfigValue, name: String) = cv.to[String].map { strConf =>
    if(strConf == name) Some(ConfigFactory.empty.root) else None
  }
  
  // Writes a config for a Season. `cv` is a config for the concrete season
  // `name` (in this case, `cv` is always an empty object).
  def to(cv: ConfigValue, name: String) = Success(name.toConfig)
  
  // If `from` returns a `Failure` for a concrete class, should we try other
  // concrete classes?
  def tryNextOnFail(name: String) = false
}

case class MyConf(list: List[Season])
loadConfig[MyConf](parseString("""list = [Spring, Summer, Autumn, Winter]"""))
// returns Success(MyConf(List(Spring, Summer, Autumn, Winter)))
```


## Handling missing keys

The default behavior of `ConfigConvert`s that are derived in PureConfig is to
raise a `KeyNotFoundException` when a required key is missing. The only
exception is the `Option[_]` type, which is read as `None` when a key is
missing:

```scala
> import pureconfig.syntax._
> import com.typesafe.config._
> ConfigFactory.empty.to[Foo]
scala.util.Try[Foo] = Failure(pureconfig.error.KeyNotFoundException: Could not find the key a)
> ConfigFactory.empty.to[FooOpt]
scala.util.Try[FooOpt] = Success(FooOpt(None))
```

However, if you want to allow your custom `ConfigConvert`s to handle missing
keys, you can extend the `AllowMissingKey` trait. For `ConfigConvert`s extending
`AllowMissingKey`, a missing key will issue a call to the `from` method of the
available `ConfigConvert` for that type with a `null` value:

```scala
> implicit val cc = new ConfigConvert[Int] with AllowMissingKey {
|   override def from(config: ConfigValue): Try[Int] =
|     if (config == null) Success(42) else Try(config.render(ConfigRenderOptions.concise).toInt)
|   override def to(t: Int): ConfigValue = ConfigValueFactory.fromAnyRef(t)
| }
> ConfigFactory.empty.to[Foo]
scala.util.Try[Foo] = Success(Foo(42))
```

## Example

In the [example directory](https://github.com/melrief/pureconfig/tree/master/example/src/main/scala/pureconfig/example)
there is an example of usage of pureconfig. In the example, the idea is to load a configuration for a directory
watcher service. The configuration file
(a real one is available [here](https://github.com/melrief/pureconfig/blob/master/example/src/main/resources/application.conf))
for this program will look like

```
dirwatch.path="/path/to/observe"
dirwatch.filter="*"
dirwatch.email.host=host_of_email_service
dirwatch.email.port=port_of_email_service
dirwatch.email.message="Dirwatch new path found report"
dirwatch.email.recipients=["recipient1,recipient2"]
dirwatch.email.sender="sender"
```

To load it, we define some classes that have proper fields and names

```scala
import java.nio.file.Path

case class Config(dirwatch: DirWatchConfig)
case class DirWatchConfig(path: Path, filter: String, email: EmailConfig)
case class EmailConfig(host: String, port: Int, message: String, recipients: Set[String], sender: String)
```

The use of `Path` gives us a chance to use a custom converter

```scala
import pureconfig._

import java.nio.file.Paths
import scala.util.Try

implicit val deriveStringConvertForPath = fromString[Path](s => Try(Paths.get(s)))
```

And then we load the configuration

```scala
val config = loadConfig[Config].get // loadConfig returns a Try
```

And that's it.

You can then use the configuration as you want:

```scala
println("dirwatch.path: " + config.dirwatch.path)
println("dirwatch.filter: " + config.dirwatch.filter)
println("dirwatch.email.host: " + config.dirwatch.email.host)
println("dirwatch.email.port: " + config.dirwatch.email.port)
println("dirwatch.email.message: " + config.dirwatch.email.message)
println("dirwatch.email.recipients: " + config.dirwatch.email.recipients)
println("dirwatch.email.sender: " + config.dirwatch.email.sender)
```

It's also possible to operate directly on `Config` and `ConfigValue` types
of [typesafe config][typesafe-config] with the implicit helpers provided in the
`pureconfig.syntax` package:

```scala
import com.typesafe.config.ConfigFactory
import pureconfig.syntax._

val config = ConfigFactory.load().to[Config].get
println("The loaded configuration is: " + config.toString)
```

## Whence the config files?

By default, PureConfig's `loadConfig()` methods load all resources in the classpath named:

- `application.conf`,
- `application.json`,
- `application.properties`, and
- `reference.conf`.

The various `pureconfig.loadConfig()` methods defer to [typesafe config][typesafe-config]'s
[`ConfigFactory`](https://typesafehub.github.io/config/latest/api/com/typesafe/config/ConfigFactory.html) to
select where to load the config files from. Typesafe Config has [well-documented rules for configuration
loading](https://github.com/typesafehub/config#standard-behavior) which we'll not repeat. Please see Typesafe
Config's documentation for a full telling of the subtleties.  If you need greater control over how config
files are loaded, refer to `ConfigFactory`'s options.

Alternatively, PureConfig also provides `pureconfig.loadConfigFromFiles()` which builds a configuration from
an explicit list of files. Files earlier in the list have greater precedence than later ones. Each file can
include a partial configuration as long as the whole list produces a complete configuration. For an example,
see the test of `loadConfigFromFiles()` in
[`PureconfSuite.scala`](https://github.com/melrief/pureconfig/blob/master/core/src/test/scala/pureconfig/PureconfSuite.scala).

Because PureConfig uses Typesafe Config to load configuration, it supports reading files in [HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md#hocon-human-optimized-config-object-notation), JSON, and Java `.properties` formats. HOCON is a delightful superset of both JSON and `.properties` that is highly recommended. As an added bonus it supports [advanced features](https://github.com/typesafehub/config/blob/master/README.md#features-of-hocon) like variable substitution and file sourcing.


## Contribute

`pureconfig` is a free library developed by several people around the world.
Contributions are welcomed and encouraged. If you want to contribute, we suggest to have a look at the
[available issues](https://github.com/melrief/pureconfig/issues) and to talk with
us on the [pureconfig gitter channel](https://gitter.im/melrief/pureconfig?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge).


## License

[Mozilla Public License, version 2.0](https://github.com/melrief/pureconfig/blob/master/LICENSE)


## Special Thanks

To the [Shapeless](https://github.com/milessabin/shapeless) and to the [Typesafe config](https://github.com/typesafehub/config)
developers.

[typesafe-config]: https://github.com/typesafehub/config
