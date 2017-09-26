Utf8Json - Fast JSON Serializer for C#
===
// TODO: badge

Definitely Fastest and Zero Allocation JSON Serializer for C#(.NET, .NET Core, Unity and Xamarin), this serializer only focus on performance of serialize/deserialize to UTF8 binary. And I adopt the same architecture as the fastest binary serializer, [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp) that I've developed.

// TODO:graph

Utf8Json does not beat MessagePack for C#(binary), but shows a similar memory consumption(there is no additional memory allocation) and higher performance than other JSON serializers.

The crucial difference is that read and write directly to UTF8 binaries means that there is no overhead. Normaly serialization requires serialize to `Stream` or `byte[]`, it requires additional UTF8.GetBytes cost or StreamReader/Writer overhead(it is very slow!).

```csharp
TargetClass obj1;

// Object to UTF8 byte[]
[Benchmark]
public byte[] Utf8JsonSerializer()
{
    return Utf8Json.JsonSerializer.Serialize(obj1, jsonresolver);
}

// Object to String to UTF8 byte[]
[Benchmark]
public byte[] Jil()
{
    return utf8.GetBytes(global::Jil.JSON.Serialize(obj1));
}

// Object to Stream with StreamWriter
[Benchmark]
public void JilTextWriter()
{
    using (var ms = new MemoryStream())
    using (var sw = new StreamWriter(ms, utf8))
    {
        global::Jil.JSON.Serialize(obj1, sw);
    }
}
```

For example, the `OutputFormatter` of [ASP.NET Core](https://github.com/aspnet/Home) needs to write to Body(`Stream`), but using [Jil](https://github.com/kevin-montrose/Jil)'s TextWriter overload is slow.

```csharp
// ASP.NET Core, OutputFormatter
public class JsonOutputFormatter : IOutputFormatter //, IApiResponseTypeMetadataProvider
{
    const string ContentType = "application/json";
    static readonly string[] SupportedContentTypes = new[] { ContentType };

    public Task WriteAsync(OutputFormatterWriteContext context)
    {
        context.HttpContext.Response.ContentType = ContentType;

        // Jil, normaly JSON Serializer requires serialize to Stream or byte[].
        using (var writer = new StreamWriter(context.HttpContext.Response.Body))
        {
            Jil.JSON.Serialize(context.Object, writer, _options);
            writer.Flush();
            return Task.CompletedTask;
        }

        // Utf8Json
        // Utf8Json.JsonSerializer.NonGeneric.Serialize(context.ObjectType, context.HttpContext.Response.Body, context.Object, resolver);
    }
}
```

The approach of directly write/read from JSON binary is similar to [corefxlab/System.Text.Json](https://github.com/dotnet/corefxlab/tree/master/src/System.Text.Json/System/Text/Json). But it is not yet finished and not be general serializer.

Install and QuickStart
---
The library provides in NuGet except for Unity. Standard library availables for .NET Standard 2.0.

```
Install-Package Utf8Json
```

And official Extension Packages for support other library(ImmutableCollection) or binding for framework(ASP.NET Core MVC).

```
Install-Package MessagePack.ImmutableCollection
Install-Package MessagePack.AspNetCoreMvcFormatter
```

QuickStart, you can call `Utf8Json.JsonSerializer`.`Serialize/Deserialize`.

```csharp
// 
byte[] result = JsonSerializer.Serialize();



JsonSerializer.Deserialize();


// 
JsonSerializer.ToJsonString();
```

In default, you can serialize all public members. You can customize serialize to private, exclude null, change DateTime format(default is ISO8601), enum handling, etc. see the [TODO] section.

Performance of Serialize
---

// image

* Cached


* High-level API uses internal memory pool, don't allocate working memory under 64K
* Struct JsonWriter does not allocate any more and wrie underlying byte[] directly, don't use TextWriter
* Avoid boxing all codes, all platforms(include Unity/IL2CPP)
* Heavyly tuned dynamic il code generation, it generates per option so reduce option check: see:[DynamicObjectResolver.cs](https://github.com/neuecc/Utf8Json/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/Utf8Json/Resolvers/DynamicObjectResolver.cs#L729-L963)
* Call Primitive API directly when il code generation knows target is primitive
* Getting cached generated formatter on static generic field(don't use dictinary-cache because dictionary lookup is overhead)
* Cache property name with delimiter("{", ",") and optimized fixed sized binary copy in IL, see: [UnsafeMemory.cs](https://github.com/neuecc/MessagePack-CSharp/blob/f724c83986d7c919a336c63e55f5a5886cca3575/src/MessagePack/Internal/UnsafeMemory.cs)







Reduce branch of variable length format when il code generation knows target(integer/string) range

Don't use IEnumerable<T> abstraction on iterate collection, see:CollectionFormatterBase and inherited collection formatters




Uses optimized type key dictionary for non-generic methods, see: ThreadsafeTypeKeyHashTable











releated to [dotnet/coreclr - issue #9786 Optimize Buffer.MemoryCopy](https://github.com/dotnet/coreclr/pull/9786)



```csharp
public sealed class PersonFormatter : IJsonFormatter<Person>
{
    private readonly byte[][] stringByteKeys;
    
    public PersonFormatter()
    {
        // pre-encoded escaped string byte with "{", ":" and ",".
        this.stringByteKeys = new byte[][]
        {
            JsonWriter.GetEncodedPropertyNameWithBeginObject("Age"), // {\"Age\":
            JsonWriter.GetEncodedPropertyNameWithPrefixValueSeparator("Name") // ",\"Name\":
        };
    }

    public sealed Serialize(ref JsonWriter writer, Person person, IJsonFormatterResolver jsonFormatterResolver)
    {
        if (person == null) { writer.WriteNull(); return; }

        // WriteRawX is optimize byte->byte copy when we know src size.
        UnsafeMemory64.WriteRaw7(ref writer, this.stringByteKeys[0]);
        writer.WriteInt32(person.Age); // itoa write directly to avoid ToString + UTF8 encode
        UnsafeMemory64.WriteRaw8(ref writer, this.stringByteKeys[1]);
        writer.WriteString(person.Name);

        writer.WriteEndObject();
    }

    // public unsafe Person Deserialize(ref JsonReader reader, IJsonFormatterResolver jsonFormatterResolver)
}
```

WriteRaw7(7 byte memory copy, sizeof(long) copy * 2) + WriteInt32(itoa algorithm requires some branch and copy memory by digits) + WriteRaw8(8 byte memory copy, sizeof(long) copy * 1) + WriteString(two-path encoding, search escape char + Encoding.UTF8.GetBytes) + WriteEndObject(write `}` directly.

This is the everything of serialization cost, probably it will be the smallest.

Performance of Deserialize
---

// image

* Avoid string key decode for lookup map(string key) key and uses automata based name lookup with il inlining code generation, see: AutomataDictionary


```csharp
public sealed class PersonFormatter : IJsonFormatter<Person>
{
    // public sealed Serialize(ref JsonWriter writer, Person person, IJsonFormatterResolver jsonFormatterResolver)

    public unsafe Person Deserialize(ref JsonReader reader, IJsonFormatterResolver jsonFormatterResolver)
    {
        if (reader.ReadIsNull()) return null;
        
        reader.ReadIsBeginObjectWithVerify(); // "{"
        
        byte[] bufferUnsafe = reader.GetBufferUnsafe();
        int age;
        string name;
        fixed (byte* ptr2 = &bufferUnsafe[0])
        {
            int num;
            while (!reader.ReadIsEndObjectWithSkipValueSeparator(ref num)) // "}" or ","
            {
                // get unescaped string-span for property name match
                ArraySegment<byte> arraySegment = reader.ReadPropertyNameSegmentUnescaped();
                byte* ptr3 = ptr2 + arraySegment.Offset;
                int count = arraySegment.Count;
                if (count != 0)
                {
                    // match automata per long
                    ulong key = AutomataKeyGen.GetKey(ref ptr3, ref count);
                    if (count == 0)
                    {
                        if (key == 6645569uL)
                        {
                            age = reader.ReadInt32(); // atoi read directly to avoid GetString + int.Parse
                            continue;
                        }
                        if (key == 1701667150uL)
                        {
                            name = reader.ReadString();
                            continue;
                        }
                    }
                }
                reader.ReadNextBlock();
            }
        }

        return new PersonSample
        {
            Age = age,
            Name = name
        };
    }
}
```


Built-in support types
---
These types can serialize by default.

Primitives(`int`, `string`, etc...), `Enum`, `Nullable<>`,  `TimeSpan`,  `DateTime`, `DateTimeOffset`, `Guid`, `Uri`, `Version`, `StringBuilder`, `BitArray`, `ArraySegment<>`, `BigInteger`, `Complext`, `Task`, `Array[]`, `Array[,]`, `Array[,,]`, `Array[,,,]`, `KeyValuePair<,>`, `Tuple<,...>`, `ValueTuple<,...>`, `List<>`, `LinkedList<>`, `Queue<>`, `Stack<>`, `HashSet<>`, `ReadOnlyCollection<>`, `IList<>`, `ICollection<>`, `IEnumerable<>`, `Dictionary<,>`, `IDictionary<,>`, `SortedDictionary<,>`, `SortedList<,>`, `ILookup<,>`, `IGrouping<,>`, `ObservableCollection<>`, `ReadOnlyOnservableCollection<>`, `IReadOnlyList<>`, `IReadOnlyCollection<>`, `ISet<>`, `ConcurrentBag<>`, `ConcurrentQueue<>`, `ConcurrentStack<>`, `ReadOnlyDictionary<,>`, `IReadOnlyDictionary<,>`, `ConcurrentDictionary<,>`, `Lazy<>`, `Task<>`, custom inherited `ICollection<>` or `IDictionary<,>` with paramterless constructor, `IList`, `IDictionary` and custom inherited `ICollection` or `IDictionary` with paramterless constructor(includes `ArrayList` and `Hashtable`) and your own class or struct(includes anonymous type).

Utf8Json has sufficient extensiblity. You can add custom type support and has some official/third-party extension package. for ImmutableCollections(`ImmutableList<>`, etc). Please see [extensions section](https://github.com/neuecc/Utf8Json#extensions).

Object Serialization
---
MessagePack for C# can serialze your own public `Class` or `Struct`. Serialization target must marks `[MessagePackObject]` and `[Key]`. Key type can choose int or string. If key type is int, serialized format is used array. If key type is string, serialized format is used map. If you define `[MessagePackObject(keyAsPropertyName: true)]`, does not require `KeyAttribute`.

All patterns serialization target are public instance member(field or property). If you want to avoid serialization target, you can add `[IgnoreMember]` to target member.

> target class must be public, does not allows private, internal class.

Which should uses int key or string key? I recommend use int key because faster and compact than string key. But string key has key name information, it is useful for debugging.

MessagePackSerializer requests target must put attribute is for robustness. If class is grown, you need to be conscious of versioning. MessagePackSerializer uses default value if key does not exists. If uses int key, should be start from 0 and should be sequential. If unnecessary properties come out, please make a missing number. Reuse is bad. Also, if Int Key's jump number is too large, it affects binary size.




I don't need type, I want to use like BinaryFormatter! You can use as typeless resolver and helpers. Please see [Typeless section](https://github.com/neuecc/MessagePack-CSharp#typeless).

Resolver is key customize point of MessagePack for C#. Details, please see [extension point](https://github.com/neuecc/MessagePack-CSharp#extension-pointiformatterresolver).


You can use `[DataContract]` instead of `[MessagePackObject]`. If type is marked DataContract, you can use `[DataMember]` instead of `[Key]` and `[IgnoreDataMember]` instead of `[IgnoreMember]`.

`[DataMember(Order = int)]` is same as `[Key(int)]`, `[DataMember(Name = string)]` is same as `[Key(string)]`. If use `[DataMember]`, same as `[Key(nameof(propertyname)]`.

Using DataContract makes it a shared class library and you do not have to refer to MessagePack for C#. However, it is not included in analysis by Analyzer or code generation by `mpc.exe`. Also, functions like `UnionAttribute`, `MessagePackFormatterAttribute`, `SerializationConstructorAttribute` etc can not be used. For this reason, I recommend that you use the MessagePack for C# attribute basically.



Serialize ImmutableObject(SerializationConstructor)

Object Type Serialization




Dynamic Deserialization
---

MessagePackSerializer.Deserialize<dynamic>



Resolvers and Configuration(DateTime format, SnakeCase, etc...)
---


Extensions
----

Which serializer should be used
---
The performance of binary(protobuf, msgpack, avro, etc...) vs text(json, xml, yaml, etc...) depends on the implementation. However, binary has advantage basically. Utf8Json write directly to `byte[]` it is close to the binary serializer. But especialy `double` is still slower than binary write(Utf8Json uses [google/double-conversion](https://github.com/google/double-conversion/) algorithm, it is good but there are many processes, it can not be the fastest), write `string` requires escape and large payload must pay copy cost.

I recommend use [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp/) for general use serializer, C# to C#, C# to NoSQL, Save to File, communicate internal cross platform(multi-language), etc. MessagePack for C# has many options(Union ,Typeless, Compression) and definitely fastest.

But JSON is still better on web, for public Web API, send for JavaScript and  easy to integrate between multi-language communication. For example, use Utf8Json for Web API formatter and use MessagePack for C# for Redis. It is perfect.

For that reason Utf8Json is focusing performance and cross-platform compatibility. I don't implement original format(like Union, Typeless, Cyclic-Reference) if you want to use it should be use binary serializer. But customizability for serialize/deserialize JSON is important for cross-platform communication. IJsonFormatterResolver can serialize/deserialize all patterns and you can create own pattern.

High-Level API(JsonSerializer)
---
`JsonSerializer` is the entry point of Utf8Json. Its static methods are main API of Utf8Json.

| API | Description |
| --- | --- |
| `DefaultResolver` | FormatterResolver that used resolver less overloads. If does not set it, used StandardResolver.Default. |
| `SetDefaultResolver` | Set default resolver of JsonSerializer APIs. |
| `Serialize<T>` | Convert object to byte[] or write to stream. There has IJsonFormatterResolver overload, used specified resolver. |
| `SerializeUnsafe<T>` | Same as `Serialize<T>` but return `ArraySegement<byte>`. The result of ArraySegment is contains internal buffer pool, it can not share across thread and can not hold, so use quickly. |
| `ToJsonString<T>` | Convert object to string. |
| `Deserialize<T>` | Convert byte[] or `ArraySegment<byte>` or stream to object. There has IFormatterResolver overload, used specified resolver. |
| `NonGeneric.*` | NonGeneric APIs of Serialize/Deserialize. There accept type parameter at first argument. This API is bit slower than generic API but useful for framework integration such as ASP.NET formatter. |

Utf8Json operates at the byte[] level, so `Deserialize<T>(Stream)` read to end at first, it is not truly streaming deserialize.

High-Level API uses memory pool internaly to avoid unnecessary memory allocation. If result size is under 64K, allocates GC memory only for the return bytes.

Low-Level API(IJsonFormatter)
---


Primitive API(JsonReader/JsonWriter)
---
`JsonReader` and `JsonWriter` is most low-level API. It is mutable struct so it must pass by ref. C# 7.2 supports ref-like types(see: [csharp-7.2/span-safety.md](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md) but not yet implements in C#, be careful to use.

`JsonReader` and `JsonWriter` is too primitive(performance reason), slightly odd. Internal state manages only int offset. All state(Is in array, object...) manage manualy.

* JsonReader

| Method | Description |
| --- | --- |
| ReadNext | Skip JSON token. |
| ReadNextBlock | Skip JSON token with sub structures(array/object). This is useful for create deserializer. |
| ReadIsNull | If is null return true. |
| ReadIsBeginArray | If is '[' return true. |
| ReadIsBeginArrayWithVerify | If is '[' return true. |
| ReadIsEndArray | If is ']' return true. |
| ReadIsEndArrayWithVerify | If is not ']' throws exception. |
| ReadIsEndArrayWithSkipValueSeparator | check reached ']' or advance ',' when (ref int count) is not zero. |
| ReadIsBeginObject | If is '{' return true. |
| ReadIsBeginObjectWithVerify | If is not '{' throws exception. |
| ReadIsEndObject | If is '}' return true. |
| ReadIsEndObjectWithVerify | If is not '}' throws exception. |
| ReadIsEndObjectWithSkipValueSeparator | check reached '}' or advance ',' when (ref int count) is not zero. |
| ReadIsValueSeparator |  If is ',' return true. |
| ReadIsValueSeparatorWithVerify | If is not ',' throws exception. |
| ReadIsNameSeparator |  If is ':' return true. |
| ReadIsNameSeparatorWithVerify | If is not ':' throws exception. |
| ReadString | ReadString, unescaped. |
| ReadStringSegmentUnsafe | ReadString block but does not decode string. Return buffer is in internal buffer pool, be careful to use.  |
| ReadPropertyName | ReadString + ReadIsNameSeparatorWithVerify. |
| ReadPropertyNameSegmentRaw | Get raw string-span(do not unescape) + ReadIsNameSeparatorWithVerify. |
| ReadBoolean | ReadBoolean. |
| ReadSByte | atoi.  |
| ReadInt16 | atoi. |
| ReadInt32 | atoi.  |
| ReadInt64 | atoi.  |
| ReadByte | atoi. |
| ReadUInt16 | atoi. |
| ReadUInt32 | atoi. |
| ReadUInt64 | atoi. |
| ReadUInt16 | atoi. |
| ReadSingle | atod. |
| ReadDouble | atod. |
| GetBufferUnsafe | return underlying buffer. |
| GetCurrentOffsetUnsafe | return underlying offset. |
| GetCurrentJsonToken | Get current token(skip whitespace), do not advance. |

`Read***` methods advance next token when called. JsonReader reads utf8 byte[] to primitive directly.

How to use, see the List formatter.

```csharp
// JsonReader is struct, always pass ref and do not set local variable.
public List<T> Deserialize(ref JsonReader reader, IJsonFormatterResolver formatterResolver)
{
    // check current state is null. when null, advanced offset. if not, do not.
    if (reader.ReadIsNull()) return null;

    var formatter = formatterResolver.GetFormatterWithVerify<T>();
    var list = new List<T>();

    var count = 0; // managing array-count state in outer.
    reader.ReadIsBeginArrayWithVerify(); // read '['
    while (!reader.ReadIsEndArrayWithSkipValueSeparator(ref count)) // loop when reached ']' or advance when ','
    {
        list.Add(formatter.Deserialize(ref reader, formatterResolver));
    }

    return list;
}
```

* JsonWriter

| Method | Description |
| --- | --- |
| static GetEncodedPropertyName | Get JSON Encoded byte[]. |
| static GetEncodedPropertyNameWithPrefixValueSeparator | Get JSON Encoded byte[] with ',' on prefix. |
| static GetEncodedPropertyNameWithBeginObject | Get JSON Encoded byte[] with '{' on prefix. |
| static GetEncodedPropertyNameWithoutQuotation | Get JSON Encoded byte[] without pre/post '"'. |
| CurrentOffset | Get current offset. |
| AdvanceOffset | Advance offset manually. |
| GetBuffer | Get current buffer. |
| ToUtf8ByteArray | Finish current buffer to byte[]. |
| ToString | Finish current buffer to json stirng. |
| EnsureCapacity | Ensure inner buffer capacity. |
| WriteRaw | Write byte/byte[] directly. |
| WriteRawUnsafe | WriteRaw but don't check and ensure capacity. |
| WriteBeginArray | Write '['. |
| WriteEndArray | Write ']'. |
| WriteBeginObject | Write '{'. |
| WriteEndObject | Write '}'. |
| WriteValueSeparator | Write ','. |
| WriteNameSeparator | Write ':'. |
| WritePropertyName | WriteString + WriteNameSeparator. |
| WriteQuotation | Write '"'. |
| WriteNull | Write 'null'. |
| WriteBoolean | Write 'true' or 'false'. |
| WriteTrue | Write 'true'. |
| WriteFalse | Write 'false'. |
| WriteSByte | itoa.  |
| WriteInt16 | itoa. |
| WriteInt32 | itoa.  |
| WriteInt64 | itoa.  |
| WriteByte | itoa. |
| WriteUInt16 | itoa. |
| WriteUInt32 | itoa. |
| WriteUInt64 | itoa. |
| WriteUInt16 | itoa. |
| WriteSingle | dtoa. |
| WriteDouble | dtoa. |

`GetBuffer`, `ToUtf8ByteArray`, `ToString` get the wrote result. JsonWriter writes primitive to utf8 bytes directly.

How to use, see the List formatter.

```csharp
// JsonWriter is struct, always pass ref and do not set local variable.
public void Serialize(ref JsonWriter writer, List<T> value, IJsonFormatterResolver formatterResolver)
{
    if (value == null) { writer.WriteNull(); return; }

    var formatter = formatterResolver.GetFormatterWithVerify<T>();

    writer.WriteBeginArray(); // "["
    if (value.Count != 0)
    {
        formatter.Serialize(ref writer, value[0], formatterResolver);
    }
    for (int i = 1; i < value.Count; i++)
    {
        writer.WriteValueSeparator(); // write "," manually
        formatter.Serialize(ref writer, value[i], formatterResolver);
    }
    writer.WriteEndArray(); // "]"
}
```

Extension Point(IJsonFormatterResolver)
---
`IJsonFormatterResolver` is storage of typed serializers. Serializer api accepts resolver and can customize serialization.

| Resovler Name | Description |
| --- | --- |
| BuiltinResolver | Builtin primitive and standard classes resolver. It includes primitive(int, bool, string...) and there nullable, array and list. and some extra builtin types(Guid, Uri, BigInteger, etc...). |
| DynamicGenericResolver | Resolver of generic type(`Tuple<>`, `List<>`, `Dictionary<,>`, `Array`, etc). It uses reflection call for resolve generic argument at first time. |
| AttributeFormatterResolver | Get formatter from `[MessagePackFormatter]` attribute. |
| EnumResolver | `EnumResolver.Default` serialize as name, `EnumResolver.UnderlyingValue` serialize as underlying value. Deserialize, can be both. |
| StandardResolver | Composited resolver. It resolves in the following order `object fallback -> (builtin -> enum -> dynamic generic -> attribute ->  dynamic object)`. `StandardResolver.Default` is default resolver of JsonSerialzier and has many option resolvers, see below. |
| CompositeResolver | Singleton custom composite resolver.  |

StandardResolver has 12 option resolvers it combinate

* AllowPrivate = true/false
* ExcludeNull = true/false
* NameMutate = Original/CamelCase/SnakeCase.

for example `StandardResolver.SnakeCase`, `StandardResolver.ExcludeNullCamelCase`, `StandardResolver.AllowPrivateExcludeNullSnakeCase`. `StandardResolver.Default` is AllowPrivate:False, ExcludeNull:False, NameMutate:Original.




It is the only configuration point to assemble the resolver's priority. In most cases, it is sufficient to have one custom resolver globally. CompositeResolver will be its helper.









IJsonFormatterAttribute
---


Text Protocol Foundation
---
Utf8Json implements fast itoa/atoi, dtoa/atod. It can be useful for text protocol serialization. For example I'm implementing [MySqlSharp](https://github.com/neuecc/MySqlSharp/) that aims fastest MySQL Driver on C#(work in progress yet), MySQL protocol is noramlly text so requires fast parser for text protocol.

`Utf8Json.Internal.NumberConverter` is Read/Write primitive to bytes. It is `public` API so you can use if requires itoa/atoi, dtoa/atod algorithm.

```csharp
byte[] buffer = null; // buffer is automatically ensure.
var offset = 0;
var writeSize = NumberConverter.WriteInt64(ref buffer, offset, 99999);

int readCount;
var value = NumberConverter.ReadInt64(buffer, 0, out readCount);
```

for Unity
---
Unity has the [JsonUtility](https://docs.unity3d.com/2017.2/Documentation/Manual/JSONSerialization.html). It is well fast but has many limitations, can not serialize/deserialize dictionary or other collections and nullable, can not root array, can not handle null correctly, etc... Utf8Json has no limitation and performance is same or better especialy convert to/from byte[], Utf8Json achieves true no zero-allocation.

In Unity version, added `UnityResolver` to StandardResolver in default. It enables serialize `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Color`, `Bounds`, `Rect`.

Pre Code Generation(Unity/Xamarin Supports)
---
Utf8Json generates object formatter dynamically by [ILGenerator](https://msdn.microsoft.com/en-us/library/system.reflection.emit.ilgenerator.aspx). It is fast and transparently generated at run time. But it does not work on AOT environment(Xamarin, Unity IL2CPP, etc.).

If you want to run on IL2CPP(or other AOT env), you need pre-code generation. `Utf8Json.UniversalCodeGenerator.exe` is code generator of Utf8Json. It is located in `packages\Utf8Json.*.*.*\tools\Utf8Json.UniversalCodeGenerator.exe` or includes for unity's package. It is using [Roslyn](https://github.com/dotnet/roslyn) so analyze source code.





How to Build
---
Open `Utf8Json.sln` on Visual Studio 2017(latest) and install .NET Core 2.0 SDK.

Unity Project is using symbolic link. At first, run `make_unity_symlink.bat` so linked under Unity project. You can open `src\Utf8Json.UnityClient` on Unity Editor.

Author Info
---
Yoshifumi Kawai(a.k.a. neuecc) is a software developer in Japan.  
He is the Director/CTO at Grani, Inc.  
Grani is a mobile game developer company in Japan and well known for using C#.  
He is awarding Microsoft MVP for Visual C# since 2011.  
He is known as the creator of [UniRx](http://github.com/neuecc/UniRx/)(Reactive Extensions for Unity)  

Blog: [https://medium.com/@neuecc](https://medium.com/@neuecc) (English)  
Blog: [http://neue.cc/](http://neue.cc/) (Japanese)  
Twitter: [https://twitter.com/neuecc](https://twitter.com/neuecc) (Japanese)  

License
---
This library is under the MIT License.