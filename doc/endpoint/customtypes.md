# Custom types

To support a custom type, you'll need to provide an implicit `Codec` for that type.

This can be done by writing a codec from scratch, mapping over an existing codec, or automatically deriving one.
Which of these approaches can be taken, depends on the context in which the codec will be used.

## Providing an implicit codec

To create a custom codec, you can either directly implement the `Codec` trait, which requires to provide the following
information:

* `encode` and `rawDecode` methods
* codec meta-data (`CodecMeta`) consisting of:
  * schema of the type (for documentation)
  * validator for the type
  * codec format (`text/plain`, `application/json` etc.)
  * type of the raw value, to which data is serialised (`String`, `Int` etc.)

This might be quite a lot of work, that's why it's usually easier to map over an existing codec. To do that, you'll 
need to provide two mappings: 

* an `encode` method which encodes the custom type into the base type
* a `decode` method which decodes the base type into the custom type, optionally reporting decode errors (the return
type is a `DecodeResult`)

For example, to support a custom id type:

```scala
def decode(s: String): DecodeResult[MyId] = MyId.parse(s) match {
  case Success(v) => DecodeResult.Value(v)
  case Failure(f) => DecodeResult.Error(s, f)
}
def encode(id: MyId): String = id.toString

implicit val myIdCodec: Codec[MyId, TextPlain, String] = Codec.stringPlainCodecUtf8
  .mapDecode(decode)(encode)

// or, using the type alias for codecs in the TextPlain format and String as the raw value:
implicit val myIdCodec: PlainCodec[MyId] = Codec.stringPlainCodecUtf8
  .mapDecode(decode)(encode)
```

> Note that inputs/outputs can also be mapped over. However, this kind of mapping is always an isomorphism, doesn't
> allow any validation or reporting decode errors. Hence, it should be used only for grouping inputs or outputs
> from a tuple into a custom type.

## Automatically deriving codecs

In some cases, codecs can be automatically derived:

* for supported [json](json.html) libraries
* for urlencoded and multipart [forms](forms.html)

Automatic codec derivation usually requires other implicits, such as:

* json encoders/decoders from the json library
* codecs for individual form fields
* schema of the custom type, through the `Schema[T]` implicit

### Schema derivation

For case classes types, `Schema[_]` values are derived automatically using [Magnolia](https://propensive.com/opensource/magnolia/), given
that schemas are defined for all of the case class's fields. It is possible to configure the automatic derivation to use
snake-case, kebab-case or a custom field naming policy, by providing an implicit `tapir.generic.Configuration` value:

```scala
implicit val customConfiguration: Configuration =
  Configuration.default.withSnakeCaseMemberNames
```

Alternatively, `Schema[_]` values can be defined by hand, either for whole case classes, or only for some of its fields.
For example, here we state that the schema for `MyCustomType` is a `String`:

```scala
implicit val schemaForMyCustomType: SchemaFor[MyCustomType] = Schema(Schema.SString)
```

If you have a case class which contains some non-standard types (other than strings, number, other case classes, 
collections), you only need to provide the schema for the non-standard types. Using these schemas, the rest will
be derived automatically.

#### Sealed traits / coproducts

Tapir supports schema generation for coproduct types (sealed trait hierarchies) of the box, but they need to be defined
by hand (as implicit values). To properly reflect the schema in [OpenAPI](../openapi.html) documentation, a 
discriminator object can be specified. 

For example, given following coproduct:

```scala
sealed trait Entity{
  def kind: String
} 
case class Person(firstName:String, lastName:String) extends Entity {
  def kind: String = "person"
}
case class Organization(name: String) extends Entity {
  def kind: String = "org"  
}
```

The schema may look like this:

```scala
val sPerson = implicitly[SchemaFor[Person]]
val sOrganization = implicitly[SchemaFor[Organization]]
implicit val sEntity: SchemaFor[Entity] = 
    SchemaFor.oneOf[Entity, String](_.kind, _.toString)("person" -> sPerson, "org" -> sOrganization)
```

### Customising derived schemas

In some cases, it might be desirable to customise the derived schemas, e.g. to add a description to a particular
field of a case class. This can be done by looking up an implicit instance of the `Derived[Schema[T]]` type, 
and assigning it to an implicit schema. When such an implicit `Schmea[T]` is in scope will have higher priority 
than the built-in low-priority conversion from `Derived[Schema[T]]` to `Schema[T]`.

Schemas for products/coproducts (case classes and case class families) can be traversed and modified using
`.modify` method. To traverse collections, use `.each`.

For example:

```scala
case class Basket(fruits: List[FruitAmount])
case class FruitAmount(fruit: String, amount: Int)
implicit val customBasketSchema: Schema[Basket] = implicitly[Derived[Schema[Basket]]].value
      .modify(_.fruits.each.amount)(_.description("How many fruits?"))
```

There is also an unsafe variant of this method, but it should be avoided in most cases. 
The "unsafe" prefix comes from the fact that the method takes a list of strings, 
which represent fields, and the correctness of this specification is not checked.

Non-standard collections can be unwrapped in the modification path by providing an implicit value of `ModifyFunctor`.

### Schema for cats datatypes

The `tapir-cats` module contains `Schema[_]` instances for some cats datatypes. See the `tapir.codec.cats.TapirCodecCats` 
trait or `import sttp.tapir.codec.cats._` to bring the implicit values into scope.

## Next

Read on about [validation](validation.html).
