# Incremental Generators

## Summary


증분 생성기는 다음과 함께 존재하는 새로운 API입니다.
[소스 생성기](source-generators.md)와 함께 존재하는 새로운 API로, 사용자가 생성을 지정할 수 있습니다.
전략을 호스팅 계층에서 고성능으로 적용할 수 있습니다.

### High Level Design Goals

- 제너레이터를 보다 세밀하게 정의할 수 있는 접근 방식 허용
- Visual Studio에서 'Roslyn/CoreCLR' 스케일 프로젝트를 지원하도록 소스 제너레이터 확장
- 세분화된 단계 간에 캐싱을 활용하여 중복 작업 감소
- 소스 텍스트뿐만 아니라 더 많은 항목 생성 지원
- ISourceGenerator` 기반 구현과 함께 존재
  
## Simple Example

먼저 추가 텍스트 파일의 내용을 추출하고 그 내용을 컴파일 시간 `const`로 사용할 수 있도록 하는 간단한 증분 생성기를 정의합니다. 다음 섹션에서는 표시된 개념에 대해 더 자세히 살펴보겠습니다.

```csharp
[Generator]
public class Generator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext initContext)
    {
        // 여기에서 파이프라인을 정의합니다:
        // .txt로 끝나는 모든 추가 파일 (addtional files) 찾기
        IncrementalValuesProvider<AdditionalText> textFiles = initContext.AdditionalTextsProvider.Where(static file => file.Path.EndsWith(".txt"));

        // 내용을 읽고 이름을 저장합니다.
        IncrementalValuesProvider<(string name, string content)> namesAndContents = textFiles.Select((text, cancellationToken) => (name: Path.GetFileNameWithoutExtension(text.Path), content: text.GetText(cancellationToken)!.ToString()));

         //해당 값을 const 문자열로 포함하는 클래스를 생성합니다.
        initContext.RegisterSourceOutput(namesAndContents, (spc, nameAndContent) =>
        {
            spc.AddSource($"ConstStrings.{nameAndContent.name}", $@"
    public static partial class ConstStrings
    {{
        public const string {nameAndContent.name} = ""{nameAndContent.content}"";
    }}");
        });
    }
}
```

## Implementation

증분 제너레이터는 `Microsoft.CodeAnalysis.IIncrementalGenerator`의 구현입니다.


```csharp
namespace Microsoft.CodeAnalysis
{
    public interface IIncrementalGenerator
    {
        void Initialize(IncrementalGeneratorInitializationContext initContext);
    }
}
```

소스 제너레이터와 마찬가지로 증분 제너레이터는 외부 어셈블리에서 정의되며 `-analyzer:` 옵션을 통해 컴파일러에 전달됩니다.

구현에는 제너레이터가 지원하는 언어를 나타내는 선택적 매개변수와 함께 `Microsoft.CodeAnalysis.GeneratorAttribute`로 주석을 달아야 합니다:


```csharp
[Generator(LanguageNames.CSharp)]
public class MyGenerator : IIncrementalGenerator { ... }
```

어셈블리에는 진단 분석기, 소스 생성기, 증분 생성기가 혼합되어 있을 수 있습니다.


### Pipeline based execution

IIncrementalGenerator`에는 추가 컴파일 횟수에 관계없이 호스트[^1]가 정확히 한 번 호출하는 `Initialize` 메서드가 있습니다. 예를 들어 로드된 프로젝트가 여러 개 있는 호스트는 여러 프로젝트에서 동일한
생성기 인스턴스를 여러 프로젝트에서 공유할 수 있으며, 호스트의 수명 동안 `Initialize`를 한 번만 호출합니다.


[^1]: IDE 또는 명령줄 컴파일러 등을 이야기합니다.

- 증분 생성기는 전용 '실행' 메서드 대신 초기화의 일부로 변경 불가능한 실행 파이프라인을 정의합니다.
- 초기화` 메서드는 제너레이터가 변환 집합을 정의하는 데 사용하는 `IncrementalGeneratorInitializationContext`의 인스턴스를 받습니다.

```csharp
public void Initialize(IncrementalGeneratorInitializationContext initContext)
{
    // 일련의 변환을 통해 여기에서 실행 파이프라인을 정의합니다
}
```

정의된 변환은 초기화 시 바로 실행되지 않고 사용 중인 데이터가 변경될 때까지 지연됩니다. 
개념적으로 이것은 열거형이 실제로 반복될 때까지 람다 표현식이 실행되지 않을 수 있는 LINQ와 유사합니다:

**IEnumerable**:

```csharp
    var squares = Enumerable.Range(1, 10).Select(i => i * 2); 

    // the code inside select is not executed until we iterate the collection
    foreach (var square in squares) { ... }
```

이러한 변환을 사용하여 나중에 입력 데이터가 변경될 경우 나중에 입력 데이터가 변경되면 필요에 따라 실행할 수 있습니다.

**Incremental Generators**:

```csharp
    IncrementalValuesProvider<AdditionalText> textFiles = context.AdditionalTextsProvider.Where(static file => file.Path.EndsWith(".txt"));
    // 위의 Where(...)의 코드는 추가 텍스트의 값이 실제로 변경될 때까지 실행되지 않습니다.
```
 
각 변환 사이에 생성된 데이터는 캐싱되어 이전에 계산된 값을 해당되는 경우 재사용할 수 있습니다. 

이 캐싱은 후속 컴파일에 필요한 계산을 줄여줍니다. 자세한 내용은 [캐싱](#캐싱)을 참조하세요.









### IncrementalValue\[s\]Provider&lt;T&gt;

입력 데이터는 파이프라인에서 불투명 데이터 소스인 `IncrementalValueProvider<T>` 또는 `IncrementalValuesProvider<T>` 형태로 사용할 수 있습니다. 

(복수형 _값_에 유의하세요. 여기서 _T_는 제공되는 입력 데이터의 유형입니다.

초기 공급자 세트는 호스트에 의해 생성되며, 호스트의 초기화 중에 제공된 
`IncrementalGeneratorInitializationContext`에서 액세스할 수 있습니다.


The currently available providers are:

- CompilationProvider // 컴파일 공급자
- AdditionalTextsProvider // 추가 텍스트 제공자
- AnalyzerConfigOptionsProvider // 분석기구성옵션제공자
- MetadataReferencesProvider // 메타데이터 참조 공급자
- ParseOptionsProvider // 파싱옵션 공급자

*Note*: there is no provider for accessing syntax nodes. This is handled
in a slightly different way. See [SyntaxValueProvider](#syntaxvalueprovider) for details.



값 제공자는 가치 자체를 담는 '상자'로 생각할 수 있습니다. 실행 파이프라인은 값 프로바이더의 값에 직접 액세스하지 않습니다.

```ascii
IValueProvider<TSource>
   ┌─────────────┐
   |             |
   │   TSource   │
   |             |
   └─────────────┘
```

대신, 제너레이터는 공급자 내에 포함된 데이터에 적용될 일련의 변환을 제공하여 새로운 값 공급자를 생성합니다.

### Select

가장 간단한 변환은 `Select`입니다. 이것은 한 공급자에 있는 값을 변환을 적용하여 새 공급자로 매핑합니다. (Linq와 비슷함)

```ascii
 IValueProvider<TSource>                   IValueProvider<TResult>
    ┌─────────────┐                           ┌─────────────┐
    │             │  Select<TSource,TResult>  │             │
    │   TSource   ├──────────────────────────►│   TResult   │
    │             │                           │             │
    └─────────────┘                           └─────────────┘
```

제너레이터 변환은 개념적으로 '값 공급자'가 'Inumerable<T>'를 대신한다는 점에서 LINQ와 어느 정도 유사하다고 생각할 수 있습니다.
트랜스폼은 일련의 확장 메서드를 통해 생성됩니다:

```csharp
public static partial class IncrementalValueSourceExtensions
{
    // 1 => 1 transform 
    public static IncrementalValueProvider<TResult> Select<TSource, TResult>(this IncrementalValueProvider<TSource> source, Func<TSource, CancellationToken, TResult> selector);
    public static IncrementalValuesProvider<TResult> Select<TSource, TResult>(this IncrementalValuesProvider<TSource> source, Func<TSource, CancellationToken, TResult> selector);
}
```
이 메서드의 반환 유형이 `IncrementalValue[s]Provider`의 인스턴스이기도 하다는 점에 유의하세요. 

이를 통해 제너레이터는 여러 변환을 함께 연결할 수 있습니다:

```ascii
 IValueProvider<TSource>                     IValueProvider<TResult1>                 IValueProvider<TResult2>
    ┌─────────────┐                            ┌─────────────┐                           ┌─────────────┐
    │             │  Select<TSource,TResult1>  │             │ Select<TResult1,TResult2> │             │
    │   TSource   ├───────────────────────────►│   TResult1  │──────────────────────────►│   TResult2  │
    │             │                            │             │                           │             │
    └─────────────┘                            └─────────────┘                           └─────────────┘
```

예제 : 

```csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// apply a 1-to-1 transform on each text, which represents extracting the path
IncrementalValuesProvider<string> transformed = additionalTexts.Select(static (text, _) => text.Path);

// transform each extracted path into something else
IncrementalValuesProvider<string> prefixTransform = transformed.Select(static (path, _) => "prefix_" + path);
```

Note how `transformed` and `prefixTransform` are themselves an
`IncrementalValuesProvider`. They represent the outcome of the transformation
that will be applied, rather than the resulting data.

### Multi Valued providers

An `IncrementalValueProvider<T>` will always provide a single value, whereas an
`IncrementalValuesProvider<T>` may provide zero or more values. For example the
`CompilationProvider` will always produce a single compilation instance, whereas
the `AdditionalTextsProvider` will produce a variable number of values,
depending on how many additional texts where passed to the compiler.

Conceptually it is simple to think about the transformation of a single item
from an `IncrementalValueProvider<T>`: the single item has the selector function
applied to it which produces a single value of `TResult`.

For an `IncrementalValuesProvider<T>` however, this transformation is more
subtle. The selector function is applied multiple times, one to each item in the
values provider. The results of each transformation are then used to create the
values for the resulting values provider:

```ascii
                                          Select<TSource, TResult>
                                   .......................................
                                   .                   ┌───────────┐     .
                                   .   selector(Item1) │           │     .
                                   . ┌────────────────►│  Result1  ├───┐ .
                                   . │                 │           │   │ .
IncrementalValuesProvider<TSource> . │                 └───────────┘   │ . IncrementalValuesProvider<TResult>
          ┌───────────┐            . │                 ┌───────────┐   │ .        ┌────────────┐
          │           │            . │ selector(Item2) │           │   │ .        │            │
          │  TSource  ├──────────────┼────────────────►│  Result2  ├───┼─────────►│   TResult  │
          │           │            . │                 │           │   │ .        │            │
          └───────────┘            . │                 └───────────┘   │ .        └────────────┘
            3 items                . │                 ┌───────────┐   │ .            3 items
     [Item1, Item2, Item3]         . │ selector(Item3) │           │   │ .  [Result1, Result2, Result3]
                                   . └────────────────►│  Result3  ├───┘ .
                                   .                   │           │     .
                                   .                   └───────────┘     .
                                   .......................................
```

It is this item-wise transformation that allows the caching to be particularly
powerful in this model. Consider when the values inside
`IncrementalValueProvider<TSource>` change. Its likely that any given change
will only change one item at a time rather than the whole collection (for example
a user typing in an additional text only changes the given text, leaving the
other additional texts unmodified).

When this occurs the generator driver can compare the input items with the ones
that were used previously. If they are considered to be equal then the
transformations for those items can be skipped and the previously computed
versions used instead. See [Comparing Items](#comparing-items) for more details.

In the above diagram if `Item2` were to change we would execute the selector on
the modified value producing a new value for `Result2`. As `Item1`and `Item3`
are unchanged the driver is free to skip executing the selector and just use the
cached values of `Result1` and `Result3` from the previous execution.

### Select Many

In addition to the 1-to-1 transform shown above, there are also transformations
that produce batches of data. For instance a given transformation may want to
produce multiple values for each input. There are a set of `SelectMany` methods that allow a transformation of 1 to
many, or many to many items:

**1 to many:**

``` csharp
public static partial class IncrementalValueSourceExtensions
{
    public static IncrementalValuesProvider<TResult> SelectMany<TSource, TResult>(this IncrementalValueProvider<TSource> source, Func<TSource, CancellationToken, IEnumerable<TResult>> selector);
}
```

```ascii
                                         SelectMany<TSource, TResult>
                                   .......................................
                                   .                   ┌───────────┐     .
                                   .                   │           │     .
                                   .               ┌──►│  Result1  ├───┐ .
                                   .               │   │           │   │ .
 IncrementalValueProvider<TSource> .               │   └───────────┘   │ . IncrementalValuesProvider<TResult>
          ┌───────────┐            .               │   ┌───────────┐   │ .        ┌────────────┐
          │           │            . selector(Item)│   │           │   │ .        │            │
          │  TSource  ├────────────────────────────┼──►│  Result2  ├───┼─────────►│   TResult  │
          │           │            .               │   │           │   │ .        │            │
          └───────────┘            .               │   └───────────┘   │ .        └────────────┘
              Item                 .               │   ┌───────────┐   │ .            3 items
                                   .               │   │           │   │ .  [Result1, Result2, Result3]
                                   .               └──►│  Result3  ├───┘ .
                                   .                   │           │     .
                                   .                   └───────────┘     .
                                   .......................................
```

**Many to many:**

``` csharp
public static partial class IncrementalValueSourceExtensions
{
    public static IncrementalValuesProvider<TResult> SelectMany<TSource, TResult>(this IncrementalValuesProvider<TSource> source, Func<TSource, CancellationToken, IEnumerable<TResult>> selector);
}
```

```ascii
                                             SelectMany<TSource, TResult>
                                   ...............................................
                                   .                        ┌─────────┐          .
                                   .                        │         │          .
                                   .                  ┌────►│ Result1 ├───────┐  .
                                   .                  │     │         │       │  .
                                   .                  │     └─────────┘       │  .
                                   .  selector(Item1) │                       │  .
                                   .┌─────────────────┘     ┌─────────┐       │  .
                                   .│                       │         │       │  .
 IncrementalValuesProvider<TSource>.│                 ┌────►│ Result2 ├───────┤  .    IncrementalValuesProvider<TResult>
          ┌───────────┐            .│                 │     │         │       │  .            ┌────────────┐
          │           │            .│ selector(Item2) │     └─────────┘       │  .            │            │
          │  TSource  ├─────────────┼─────────────────┤     ┌─────────┐       ├──────────────►│  TResult   │
          │           │            .│                 │     │         │       │  .            │            │
          └───────────┘            .│                 └────►│ Result3 ├───────┤  .            └────────────┘
             3 items               .│                       │         │       │  .               7 items
       [Item1, Item2, Item3]       .│ selector(Item3)       └─────────┘       │  .  [Result1, Result2, Result3, Result4, 
                                   .└─────────────────┐                       │  .      Result5, Result6, Result7 ]
                                   .                  │     ┌─────────┐       │  .
                                   .                  │     │         │       │  .
                                   .                  ├────►│ Result4 ├───────┤  .
                                   .                  │     │         │       │  .
                                   .                  │     └─────────┘       │  .
                                   .                  │     ┌─────────┐       │  .
                                   .                  │     │         │       │  .
                                   .                  ├────►│ Result5 ├───────┤  .
                                   .                  │     │         │       │  .
                                   .                  │     └─────────┘       │  .
                                   .                  │     ┌─────────┐       │  .
                                   .                  │     │         │       │  .
                                   .                  └────►│ Result6 ├───────┘  .
                                   .                        │         │          .
                                   .                        └─────────┘          .
                                   ...............................................
```

For example, consider a set of additional XML files that contain multiple
elements of the same type. The generator may want to treat each element as a
distinct item for generation, effectively splitting a single additional file
into multiple sub-items.

``` csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// extract each element from each additional file
IncrementalValuesProvider<MyElementType> elements = additionalTexts.SelectMany(static (text, _) => /*transform text into an array of MyElementType*/);

// now the generator can consider the union of elements in all additional texts, without needing to consider multiple files
IncrementalValuesProvider<string> transformed = elements.Select(static (element, _) => /*transform the individual element*/);
```

### Where

Where allows the author to filter the values in a value provider by a given
predicate. Where is actually a specific form of select many, where each input
transforms to exactly 1 or 0 outputs. However, as it is such a common operation
it is provided as a primitive transformation directly.

``` csharp
public static partial class IncrementalValueSourceExtensions
{
    public static IncrementalValuesProvider<TSource> Where<TSource>(this IncrementalValuesProvider<TSource> source, Func<TSource, bool> predicate);
}
```

```ascii
                                               Where<TSource>
                                   .......................................
                                   .                   ┌───────────┐     .
                                   .   predicate(Item1)│           │     .
                                   . ┌────────────────►│   Item1   ├───┐ .
                                   . │                 │           │   │ .
IncrementalValuesProvider<TSource> . │                 └───────────┘   │ . IncrementalValuesProvider<TSource>
          ┌───────────┐            . │                                 │ .        ┌───────────┐
          │           │            . │ predicate(Item2)                │ .        │           │
          │  TSource  ├──────────────┼─────────────────X               ├─────────►│  TSource  │
          │           │            . │                                 │ .        │           │
          └───────────┘            . │                                 │ .        └───────────┘
             3 Items               . │                 ┌───────────┐   │ .           2 Items
                                   . │ predicate(Item3)│           │   │ .
                                   . └────────────────►│   Item3   ├───┘ .
                                   .                   │           │     .
                                   .                   └───────────┘     .
                                   .......................................
```

An obvious use case is to filter out inputs the generator knows it isn't
interested in. For example, the generator will likely want to filter additional
texts on file extensions:

```csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// filter additional texts by extension
IncrementalValuesProvider<string> xmlFiles = additionalTexts.Where(static (text, _) => text.Path.EndsWith(".xml", StringComparison.OrdinalIgnoreCase));
```

### Collect

When performing transformations on a value provider with multiple items, it can
often be useful to view the items as a single collection rather than one item at
a time. For this there is the `Collect` transformation.

`Collect` transforms an `IncrementalValuesProvider<T>` to an
`IncrementalValueProvider<ImmutableArray<T>>`. Essentially it transforms a multi-valued source
into a single value source with an array of all the items.

```csharp
public static partial class IncrementalValueSourceExtensions
{
    IncrementalValueProvider<ImmutableArray<TSource>> Collect<TSource>(this IncrementalValuesProvider<TSource> source);
}
```

```ascii
IncrementalValuesProvider<TSource>                IncrementalValueProvider<ImmutableArray<TSource>>
          ┌───────────┐                                  ┌─────────────────────────┐
          │           │          Collect<TSource>        │                         │
          │  TSource  ├─────────────────────────────────►│ ImmutableArray<TSource> │
          │           │                                  │                         │
          └───────────┘                                  └─────────────────────────┘
             3 Items                                             Single Item

              Item1                                         [Item1, Item2, Item3]
              Item2
              Item3
```

```csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// collect the additional texts into a single item
IncrementalValueProvider<AdditionalText[]> collected = additionalTexts.Collect();

// perform a transformation where you can access all texts at once
var transform = collected.Select(static (texts, _) => /* ... */);
```

### Multi-path pipelines

The transformations described so far are all effectively single-path operations:
while there may be multiple items in a given provider, each transformation
operates on a single input value provider and produce a single derived output
provider.

While sufficient for simple operations, it is often necessary to combine the
values from multiple input providers or use the results of a transformation
multiple times. For this there are a set of transformations that split and
combine a single path of transformations into a multi-path pipeline.

### Split

It is possible to split the output of a transformations into multiple
parallel inputs. Rather than having a dedicated transformation this can be
achieved by simply using the same value provider as the input to multiple
transforms.

```ascii

                                                     IncrementalValueProvider<TResult>
                                                              ┌───────────┐
                                      Select<TSource,TResult> │           │
 IncrementalValueProvider<TSource>   ┌───────────────────────►│  TResult  │
           ┌───────────┐             │                        │           │
           │           │             │                        └───────────┘
           │  TSource  ├─────────────┤
           │           │             │
           └───────────┘             │                    IncrementalValuesProvider<TResult2>
                                     │                              ┌───────────┐
                                     │ SelectMany<TSource,TResult2> │           │
                                     └─────────────────────────────►│  TResult2 │
                                                                    │           │
                                                                    └───────────┘
```

Those transforms can then be used as the inputs to new single path transforms, independent of one another.

For example:

```csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = context.AdditionalTextsProvider;

// apply a 1-to-1 transform on each text, extracting the path
IncrementalValuesProvider<string> transformed = additionalTexts.Select(static (text, _) => text.Path);

// split the processing into two paths of derived data
IncrementalValuesProvider<string> nameTransform = transformed.Select(static (path, _) => "prefix_" + path);
IncrementalValuesProvider<string> extensionTransform = transformed.Select(static (path, _) => Path.ChangeExtension(path, ".new"));
```

`nameTransform` and `extensionTransform` produce different values for the same
set of additional text inputs. For example if there was an additional file
called `file.txt` then `nameTransform` would produce the string
`prefix_file.txt` where `extensionTransform` would produce the string
`file.new`.

When the value of the additional file changes, the subsequent values produced
may or may not differ. For example if the name of the additional file was
changed to `file.xml` then `nameTransform` would now produce `prefix_file.xml`
whereas `extensionTransform` would still produce `file.new`. Any child transform
with input from `nameTransform` would be re-run with the new value, but any
child of `extensionTransform` would use the previously cached version as it's
input hasn't changed.

### Combine

결합은 가장 강력하지만 가장 복잡한 변환이기도 합니다. 이를 통해 제너레이터는 두 개의 입력 공급자를 가져와 하나의 통합된 출력 공급자를 만들 수 있습니다.

**단일 값에서 단일 값으로**:

```csharp
public static partial class IncrementalValueSourceExtensions
{
    IncrementalValueProvider<(TLeft Left, TRight Right)> Combine<TLeft, TRight>(this IncrementalValueProvider<TLeft> provider1, IncrementalValueProvider<TRight> provider2);
}
```

두 개의 단일 값 공급자를 결합할 때 결과 노드는 두 입력 항목의 '튜플'을 포함하는 새로운 값 공급자로 개념적으로 쉽게 이해할 수 있습니다.

```ascii

IncrementalValueProvider<TSource1>
         ┌───────────┐
         │           │
         │  TSource1 ├────────────────┐
         │           │                │                                 IncrementalValueProvider<(TSource1, TSource2)>
         └───────────┘                │
          Single Item                 │                                          ┌────────────────────────┐
                                      │       Combine<TSource1, TSource2>        │                        │
            Item1                     ├─────────────────────────────────────────►│  (TSource1, TSource2)  │
                                      │                                          │                        │
IncrementalValueProvider<TSource2>    │                                          └────────────────────────┘
         ┌───────────┐                │                                                   Single Item
         │           │                │
         │  TSource2 ├────────────────┘                                                  (Item1, Item2)
         │           │
         └───────────┘
          Single Item

            Item2

```

**Multi-value to single-value:**

```csharp
public static partial class IncrementalValueSourceExtensions
{
    IncrementalValuesProvider<(TLeft Left, TRight Right)> Combine<TLeft, TRight>(this IncrementalValuesProvider<TLeft> provider1, IncrementalValueProvider<TRight> provider2);
}
```

그러나 다중 값 공급자를 단일 값 공급자로 결합하는 경우 의미는 조금 더 복잡해집니다. 

결과 다중값 공급자는 일련의 튜플을 생성합니다.

각 튜플의 왼쪽은 다중값 입력에서 생성된 값이고, 

오른쪽은 항상 단일값 공급자 입력에서 생성된 동일한 단일값입니다.


```ascii
 IncrementalValuesProvider<TSource1>
          ┌───────────┐
          │           │
          │  TSource1 ├────────────────┐
          │           │                │
          └───────────┘                │
             3 Items                   │                                IncrementalValuesProvider<(TSource1, TSource2)>
                                       │
            LeftItem1                  │                                          ┌────────────────────────┐
            LeftItem2                  │       Combine<TSource1, TSource2>        │                        │
            LeftItem3                  ├─────────────────────────────────────────►│  (TSource1, TSource2)  │
                                       │                                          │                        │
                                       │                                          └────────────────────────┘
 IncrementalValueProvider<TSource2>    │                                                  3 Items
          ┌───────────┐                │
          │           │                │                                            (LeftItem1, RightItem)
          │  TSource2 ├────────────────┘                                            (LeftItem2, RightItem)
          │           │                                                             (LeftItem3, RightItem)
          └───────────┘
           Single Item

            RightItem
```

**Multi-value to multi-value:**

As shown by the definitions above it is not possible to combine a multi-value
source to another multi-value source. The resulting cross join would potentially
contain a large number of values, so the operation is not provided by default.

Instead, an author can call `Collect()` on one of the input multi-value providers
to produce a single-value provider that can be combined as above.

```ascii
                                           IncrementalValuesProvider<TSource1>
                                                  ┌───────────┐
                                                  │           │
                                                  │ TSource1  ├──────────────┐
                                                  │           │              │
                                                  └───────────┘              │
                                                     3 Items                 │                                IncrementalValuesProvider<(TSource1, TSource2[])>
                                                                             │
                                                    LeftItem1                │                                          ┌────────────────────────┐
                                                    LeftItem2                │       Combine<TSource1, TSource2[]>      │                        │
                                                    LeftItem3                ├─────────────────────────────────────────►│  (TSource1, TSource2)  │
                                                                             │                                          │                        │
                                                                             │                                          └────────────────────────┘
IncrementalValuesProvider<TSource2>     IncrementalValueProvider<TSource2[]> │                                                  3 Items
         ┌───────────┐                           ┌────────────┐              │
         │           │      Collect<TSource2>    │            │              │                               (LeftItem1, [RightItem1, RightItem2, RightItem3])
         │  TSource2 ├───────────────────────────┤ TSource2[] ├──────────────┘                               (LeftItem2, [RightItem1, RightItem2, RightItem3])
         │           │                           │            │                                              (LeftItem3, [RightItem1, RightItem2, RightItem3])
         └───────────┘                           └────────────┘
            3 Items                                Single Item

          RightItem1                 [RightItem1, RightItem2, RightItem3]
          RightItem2
          RightItem3
```

With the above transformations the generator author can now take one or more
inputs and combine them into a single source of data. For example:

```csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// combine each additional text with the parse options
IncrementalValuesProvider<(AdditionalText, ParseOptions)> combined = additionalTexts.Combine(initContext.ParseOptionsProvider);

// perform a transform on each text, with access to the options
var transformed = combined.Select(static (pair, _) => 
{
    AdditionalText text = pair.Left;
    ParseOptions parseOptions = pair.Right;
    // do the actual transform ...
});
```

If either of the inputs to a combine change, then subsequent transformation will
re-run. However, the caching is considered on a pairwise basis for each output
tuple. For instance, in the above example, if only additional text changes the
subsequent transform will only be run for the text that changed. The other text
and parse options pairs are skipped and their previously computed value are
used. If the single value changes, such as the parse options in the example,
then the transformation is executed for every tuple.

### SyntaxValueProvider

구문 노드는 값 공급자를 통해 직접 사용할 수 없습니다.

 대신 제너레이터 작성자는 특수한 `SyntaxValueProvider`(`IncrementalGeneratorInitializationContext.SyntaxProvider`를 통해 제공)를 사용하여 관심 있는 구문의 하위 집합을 대신 노출하는 전용 입력 노드를 생성합니다. 

구문 제공자는 원하는 수준의 성능을 달성하기 위해 이러한 방식으로 특화되어 있습니다.
#### CreateSyntaxProvider

Currently the provider exposes a single method `CreateSyntaxProvider` that
allows the author to construct an input node.

```csharp
    public readonly struct SyntaxValueProvider
    {
        public IncrementalValuesProvider<T> CreateSyntaxProvider<T>(Func<SyntaxNode, CancellationToken, bool> predicate, Func<GeneratorSyntaxContext, CancellationToken, T> transform);
    }
```

Note how this takes _two_ lambda parameters: one that examines a `SyntaxNode` in
isolation, and a second one that can then use the `GeneratorSyntaxContext` to
access a semantic model and transform the node for downstream usage.

It is because of this split that performance can be achieved: as the driver is
aware of which nodes are chosen for examination, it can safely skip the first
`predicate` lambda when a syntax tree remains unchanged. The driver will still
re-run the second `transform` lambda even for nodes in unchanged files, as a
change in one file can impact the semantic meaning of a node in another file.

Consider the following syntax trees:

```csharp
// file1.cs
public class Class1
{
    public int Method1() => 0;
}

// file2.cs
public class Class2
{
    public Class1 Method2() => null;
}

// file3.cs
public class Class3 {}
```

As an author I can make an input node that extracts the return type information 

```csharp
// create a syntax provider that extracts the return type kind of method symbols
var returnKinds = initContext.SyntaxProvider.CreateSyntaxProvider(static (n, _) => n is MethodDeclarationSyntax,
                                                                  static (n, _) => ((IMethodSymbol)n.SemanticModel.GetDeclaredSymbol(n.Node)).ReturnType.Kind);
```

Initially the `predicate` will run for all syntax nodes, and select the two
`MethodDeclarationSyntax` nodes `Method1()` and `Method2()`. These are then
passed to the `transform` where the semantic model is used to obtain the method
symbol and extract the kind of the return type for the method. `returnKinds`
will contain two values, both `NamedType`.

Now imagine that `file3.cs` is edited:

```csharp
// file3.cs
public class Class3 {
    public int field;
}
```

The `predicate` will only be run for syntax nodes inside `file3.cs`, and will
not return any as it still doesn't contain any method symbols. The `transform`
however will still be run again for the two methods from `Class1` and `Class2`.

To see why it was necessary to re-run the `transform` consider the following
edit to `file1.cs` where we change the classes name:

```csharp
// file1.cs
public class Class4
{
    public int Method1() => 0;
}
```

The `predicate` will be re-run for `file1.cs` as it has changed, and will pick
out the method symbol `Method1()` again.  Next, because the `transform` is
re-run for _all_ the methods, the return type kind for `Method2()` is correctly
changed to `ErrorType` as `Class1` no longer exists.

Note that we didn't need to run the `predicate` over for nodes in `file2.cs`
even though they referenced something in `file1.cs`. Because the first check is
purely _syntactic_ we can be sure the results for `file2.cs` would be the same.

While it may seem unfortunate that the driver must run the `transform` for all
selected syntax nodes, if it did not it could end up producing incorrect data
due to cross file dependencies. Because the initial syntactic check
allows the driver to substantially filter the number of nodes on which the
semantic checks have to be re-run, significantly improved performance
characteristics are still observed when editing a syntax tree.

#### ForAttributeWithMetadataName (FAWMN)

제너레이터가 작성되는 매우 일반적인 작업 중 하나는 특정 구문 구조에 적용된 속성에 기반한 작업을 수행하는 것입니다.

```csharp
public readonly struct SyntaxValueProvider
{
    public IncrementalValuesProvider<T> ForAttributeWithMetadataName<T>(
        string fullyQualifiedMetadataName,
        Func<SyntaxNode, CancellationToken, bool> predicate,
        Func<GeneratorAttributeSyntaxContext, CancellationToken, T> transform);
}
```

이 영역은 사용자가 제공한 `술어`를 호출하기도 전에 상당한 수의 구문 노드와 편집을 효율적으로 제거하여 상당한 수의 `SyntaxNode` 인스턴스를 구현하지 않아도 되므로 최적화에 특히 유용합니다. 


Roslyn은 초기 휴리스틱으로 작은 인덱스를 유지하고 유형 이름을 비교하여 주어진 속성이 생성기가 신경 쓰는 속성일 가능성이 있는지 여부를 추적함으로써 이를 더욱 최적화할 수 있습니다.


이 인덱스는 유지 관리 비용이 저렴하며, 중요한 것은 오탐이 아닌 오탐만 있을 수 있다는 점입니다. 

이를 통해 컴파일에서 99%의 구문을 의미론적 정보(휴리스틱 캐시에서 오탐을 제거하기 위해) 또는 사용자 `predicate` 함수에 의해 검사할 필요가 없도록 할 수 있습니다(`SyntaxNode` 인스턴스 할당을 상당히 많이 절약할 수 있습니다).




이러한 점을 고려할 때 가능하면 다른 구문 구조보다는 속성을 사용하여 소스 생성기를 구동하는 것이 좋습니다. 

실제 테스트 결과 이 접근 방식은 일반적으로 제너레이터가 제대로 작동하지 않는 경우에도 `CreateSyntaxProvider`보다 99배 더 효율적이며, 일부 병리적 시나리오에서는 그보다 훨씬 더 효율적이라는 것이 밝혀졌습니다.

어트리뷰트는 어셈블리 이름 부분 없이 사용자가 정규화된 메타데이터 이름으로 제공합니다. 

예를 들어, C# typeMy.Namespace.MyAttribute<T>`,

이 정규화된 메타데이터 이름은 ``My.Namespace.MyAttribute`1``이 됩니다. 
 
속성은 일반적으로 `AttributeUsage` 속성에 의해 특정 구조로 제한되므로, 사용자가 제공하는 `술어`는 단순히 `true`를 반환하는 것이 일반적입니다.

변환 단계의 경우, [이전 섹션](#createsyntaxprovider)에 명시된 모든 내용이 여전히 관련되며, 변경된 의미가 준수되도록 변경할 때마다 해당 단계가 다시 실행됩니다.


## Outputting values

At some point in the pipeline the author will want to actually use the
transformed data to produce an output, such as a `SourceText`. There are a set
of `Register...Output` methods on the
`IncrementalGeneratorInitializationContext` that allow the generator author to
construct an output from a series of transformations.

These output registrations are terminal, in that the they do not return a value
provider and can have no further transformations applied to them. However an
author is free to register multiple outputs of the same type with different
input transformations.

The set of output methods are

- RegisterSourceOutput
- RegisterImplementationSourceOutput
- RegisterPostInitializationOutput

**RegisterSourceOutput**:

등록 소스 출력`은 생성기 작성자가 사용자 컴파일에 포함될 소스 파일과 진단을 생성할 수 있도록 합니다. 

입력으로 `Value[s]Provider`와 값 제공자의 모든 값에 대해 호출될 `Action<SourceProductionContext, TSource>`를 받습니다.

``` csharp
public static partial class IncrementalValueSourceExtensions
{
    public void RegisterSourceOutput<TSource>(IncrementalValueProvider<TSource> source, Action<SourceProductionContext, TSource> action);
    public void RegisterSourceOutput<TSource>(IncrementalValuesProvider<TSource> source, Action<SourceProductionContext, TSource> action);
}
```

제공된 '소스 프로덕션 컨텍스트'는 소스 파일을 추가하고 진단을 보고하는 데 사용할 수 있습니다:

```csharp
public readonly struct SourceProductionContext
{
    public CancellationToken CancellationToken { get; }

    public void AddSource(string hintName, string source);
   
    public void ReportDiagnostic(Diagnostic diagnostic);
}
```

For example, a generator can extract out the set of paths for the additional
files and create a method that prints them out:

``` csharp
// get the additional text provider
IncrementalValuesProvider<AdditionalText> additionalTexts = initContext.AdditionalTextsProvider;

// apply a 1-to-1 transform on each text, extracting the path
IncrementalValuesProvider<string> transformed = additionalTexts.Select(static (text, _) => text.Path);

// collect the paths into a batch
IncrementalValueProvider<ImmutableArray<string>> collected = transformed.Collect();

// take the file paths from the above batch and make some user visible syntax
initContext.RegisterSourceOutput(collected, static (sourceProductionContext, filePaths) =>
{
    sourceProductionContext.AddSource("additionalFiles.cs", @"
namespace Generated
{
    public class AdditionalTextList
    {
        public static void PrintTexts()
        {
            System.Console.WriteLine(""Additional Texts were: " + string.Join(", ", filePaths) + @" "");
        }
    }
}");
});
```

**RegisterImplementationSourceOutput**:

RegisterImplementationSourceOutput`은 `RegisterSourceOutput`과 같은 방식으로 작동하지만 코드 분석의 관점에서 생성된 소스가 사용자 코드에 의미론적 영향을 미치지 않는다고 선언합니다. 

이를 통해 IDE와 같은 호스트는 성능 최적화를 위해 이러한 출력을 실행하지 않도록 선택할 수 있습니다. 

실행 코드를 생성하는 호스트는 항상 이러한 출력을 실행합니다.


**RegisterPostInitializationOutput**:

'RegisterPostInitializationOutput'을 사용하면 제너레이터 작성자가 초기화가 실행된 직후 소스 코드를 제공할 수 있습니다. 이 함수는 입력을 받지 않으므로 사용자가 작성한 소스 코드나 다른 컴파일러 입력을 참조할 수 없습니다.

초기화 후 소스는 다른 변환이 실행되기 전에 컴파일에 포함되므로 나머지 일반 실행 파이프라인의 일부로 표시되며, 작성자는 이에 대해 의미론적 질문을 할 수 있습니다.
에 대해 시맨틱 질문을 할 수 있습니다.

특히 사용자의 소스 코드에 속성 정의를 추가할 때 유용합니다. 그런 다음 사용자가 코드에 이를 적용할 수 있으며, 생성기는 시맨틱 모델을 통해 어트리뷰션된 코드를 찾을 수 있습니다.

## Handling Cancellation

Incremental generators are designed to be used in interactive hosts such as an
IDE. As such, it is critically important that generators respect and respond to
the passed-in cancellation tokens.

In general, it is likely that the amount of user computation performed per
transformation is low, but often will be calling into Roslyn APIs that may have
a significant performance impact. As such the author should always forward the
provided cancellation token to any Roslyn APIs that accept it.

For example, when retrieving the contents of an additional file, the token
should be passed into `GetText(...)`:

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    var txtFiles = context.AdditionalTextsProvider.Where(static f => f.Path.EndsWith(".txt", StringComparison.OrdinalIgnoreCase));
    
    // ensure we forward the cancellation token to GetText
    var fileContents = txtFiles.Select(static (file, cancellationToken) => file.GetText(cancellationToken));   
}
```

This will ensure that an incremental generator correctly and quickly responds to
cancellation requests and does not cause delays in the host.

If the generator author is doing something expensive, such as looping over
values, they should regularly check for cancellation themselves. It is recommend
that the author use `CancellationToken.ThrowIfCancellationRequested()` at
regular intervals, and allow the host to re-run them, rather than attempting to
save partially generated results which can be extremely difficult to author
correctly.

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    var txtFilesArray = context.AdditionalTextsProvider.Where(static f => f.Path.EndsWith(".txt", StringComparison.OrdinalIgnoreCase)).Collect();
    
    var expensive = txtFilesArray.Select(static (files, cancellationToken) => 
    {
        foreach (var file in files)
        {
            // check for cancellation so we don't hang the host
            cancellationToken.ThrowIfCancellationRequested();

            // perform some expensive operation (ideally passing in the token as well)
            ExpensiveOperation(file, cancellationToken);
        }
    });   
}
```

## Caching

세분화된 단계를 사용하면 제너레이터 호스트를 통해 출력 유형을 다소 거칠게 제어할 수 있지만, 드라이버가 한 파이프라인 단계에서 다음 단계로 출력을 캐시할 수 있을 때만 성능상의 이점을 실제로 볼 수 있습니다. 

일반적으로 `ISourceGenerator`의 실행 메서드는 다음과 같아야 한다고 말씀드렸습니다.
결정론적이어야 한다고 말했지만, 증분 제너레이터는 이 속성이 참일 것을 적극적으로 _요구_합니다.



단계의 일부로 적용할 필수 변환을 계산할 때 제너레이터 드라이버는 이전에 보았던 입력을 자유롭게 살펴보고  
이러한 입력에 대해 이전에 계산되고 캐시된 변환 값을 사용할 수 있습니다.

다음 변환을 생각해 보세요:

```csharp
IValuesProvider<string> transform = context.AdditionalTextsProvider
                                           .Select(static (t, _) => t.Path)
                                           .Select(static (p, _) => "prefix_" + p);
```

파이프라인을 처음 실행하는 동안 두 람다 각각은 다음과 같이 실행됩니다.
각 추가 파일에 대해 실행됩니다:

AdditionalText          | Select1    | Select2
------------------------|------------|-----------------
Text{ Path: "abc.txt" } | "abc.txt"  | "prefix_abc.txt"
Text{ Path: "def.txt" } | "def.txt"  | "prefix_def.txt"
Text{ Path: "ghi.txt" } | "ghi.txt"  | "prefix_ghi.txt"

이제 향후 반복 작업에서 첫 번째 추가 파일이 변경되어 경로가 달라지고 두 번째 파일이 변경되었지만 경로가 동일하게 유지되는 경우를 생각해 보겠습니다.

AdditionalText               | Select1    | Select2
-----------------------------|------------|-----------
**Text{ Path: "diff.txt" }** |            |
**Text{ Path: "def.txt" }**  |            |
Text{ Path: "ghi.txt" }      |            |

생성기는 첫 번째 파일과 두 번째 파일에 대해 select1을 실행하여 각각 "diff.txt"와 "def.txt"를 생성합니다. 그러나 입력이 변경되지 않았으므로 세 번째 파일에 대해 select를 다시 실행할 필요는 없습니다. 이전에 캐시된 값을 사용하면 됩니다.


AdditionalText               | Select1        | Select2
-----------------------------|----------------|-----------
**Text{ Path: "diff.txt" }** | **"diff.txt"** |
**Text{ Path: "def.txt" }**  | **"def.txt"**  |
Text{ Path: "ghi.txt" }      | "ghi.txt"      |

다음으로 드라이버는 Select2를 실행하려고 합니다. 

이 드라이버는 `"diff.txt"`에서 작동하여 `"prefix_diff.txt"`를 생성하지만 `"def.txt"`의 경우 생성된 항목이 마지막 반복과 동일하다는 것을 관찰할 수 있습니다. 

원래 입력(`Text{ Path: "def.txt" }`)이 변경되었음에도 불구하고 Select1의 결과는 동일합니다. 

따라서 `"def.txt"`에서 Select2를 다시 실행할 필요 없이 이전부터 캐시된 값을 사용할 수 있습니다. 

마찬가지로 캐시된 상태의
"ghi.txt"의 캐시된 상태를 사용할 수 있습니다.

AdditionalText               | Select1        | Select2
-----------------------------|----------------|----------------------
**Text{ Path: "diff.txt" }** | **"diff.txt"** | **"prefix_diff.txt"**
**Text{ Path: "def.txt" }**  | **"def.txt"**  | "prefix_def.txt"
Text{ Path: "ghi.txt" }      | "ghi.txt"      | "prefix_ghi.txt"

이렇게 하면 결과적인 변경 사항만 파이프라인을 통해 흐르고 중복 작업을 피할 수 있습니다. 

제너레이터가 `AdditionalTexts`에만 의존하는 경우 드라이버는 `SyntaxTree`가 변경될 때 수행할 작업이 없다는 것을 알고 있습니다.

### Comparing Items

For a user-provided result to be comparable across iterations, there needs to be
some concept of equivalence. By default the host will use `EqualityComparer<T>`
to determine equivalence. There are obviously times where this is insufficient,
and there exists an extension method that allows the author to supply a comparer
that should be used when comparing values for the given transformation:

```csharp
public static partial class IncrementalValueProviderExtensions
{
        public static IncrementalValueProvider<TSource> WithComparer<TSource>(this IncrementalValueProvider<TSource> source, IEqualityComparer<TSource> comparer);

        public static IncrementalValuesProvider<TSource> WithComparer<TSource>(this IncrementalValuesProvider<TSource> source, IEqualityComparer<TSource> comparer);
}
```

Allowing the generator author to specify a given comparer.

```csharp
var withComparer = context.AdditionalTextsProvider
                          .Select(static t => t.Path)
                          .WithComparer(myComparer);
```

Note that the comparer is on a per-transformation basis, meaning an author can
specify different comparers for different parts of the pipeline.

```csharp
var select = context.AdditionalTextsProvider.Select(static t => t.Path);

var noCompareSelect = select.Select(...);
var compareSelect = select.WithComparer(myComparer).Select(...);
```

The same select node can have no comparer when acting as input to one transformation,
and still provide one when acting as input to a different transformation.

The host will only invoke the given comparer when the item it is derived from
has been modified. When the input value is new or being removed, or the input
transformation was determined to be cached (possibly by a provided comparer) the
given comparer is not considered.

### Authoring a cache friendly generator

증분 생성기의 성공 여부는 캐싱이 가능한 최적의 파이프라인을 만드는 데 달려 있습니다. 

이 섹션에는 이를 달성하기 위한 몇 가지 일반적인 팁과 모범 사례가 포함되어 있습니다.

**정보를 빨리 빼내기**: 파이프라인에서 가능한 한 빨리 입력에서 정보를 꺼내는 것이 가장 좋습니다. 이렇게 하면 호스트가 심볼과 같은 크고 비용이 많이 드는 객체를 캐싱하지 않습니다.

**가능하면 값 유형을 사용하세요**: 값 유형은 캐싱에 더 적합하며 일반적으로 비교 의미가 잘 정의되어 있고 이해하기 쉽습니다.

**다중 변환 사용**: 연산을 더 많은 변환으로 나눌수록 캐시할 수 있는 기회가 많아집니다. 변환을 실행 그래프의 '체크 포인트'라고 생각하면 됩니다. 체크 포인트가 많을수록 캐시된 값과 일치하고 나머지 작업을 건너뛸 수 있는 기회가 많아집니다.


**데이터 모델 구축**: 각 입력 항목을 `등록...출력` 메서드에 전달하려고 하는 대신, 출력에 전달되는 최종 항목이 될 데이터 모델을 구축하는 것을 고려하세요. 

변환을 사용하여 데이터 모델을 조작하고, 모델의 수정본 간에 정확하게 비교할 수 있도록 동등성을 잘 정의하세요. 

이렇게 하면 증분 변환을 에뮬레이트하는 대신 더미 데이터 모델로 메서드를 호출하고 생성된 코드를 확인하기만 하면 되므로 최종 `Register...Outputs`를 훨씬 더 간단하게 테스트할 수 있습니다.



**결합 순서를 고려하세요**: 
필요한 최소한의 정보만 결합하고 있는지 확인하세요(이는 '정보 빨리 추출하기'로 다시 돌아옵니다). 

기본 입력이 결합된 다음 소스를 생성하는 데 사용되는 다음과 같은 (잘못된) 결합을 고려해 보세요:



```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    var compilation = context.CompilationProvider;
    var texts = context.AdditionalTextsProvider;

    // Don't do this!
    var combined = texts.Combine(compilation);

    context.RegisterSourceOutput(combined, static (spc, pair) =>
    {
        var assemblyName = pair.Right.AssemblyName;
        // produce source ...
    });
```
컴파일이 변경될 때마다, 즉 사용자가 IDE에 입력하는 동안 자주 변경될 때마다 `RegisterSourceOutput`이 다시 실행됩니다. 

대신 컴파일 종속 정보를 먼저 조회한 다음 _그것_을 추가 파일과 결합하세요.

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    var assemblyName = context.CompilationProvider.Select(static (c, _) => c.AssemblyName);
    var texts = context.AdditionalTextsProvider;

    var combined = texts.Combine(assemblyName);

    context.RegisterSourceOutput(combined, (spc, pair) =>
    {
        var assemblyName = pair.Right;
        // produce source ...
    });
}
```
이제 사용자가 IDE에 입력하면 `assemblyName` 트랜스폼이 다시 실행되지만 매우 저렴하고 매번 동일한 값을 빠르게 반환합니다. 

즉, 추가 텍스트도 변경되지 않는 한 호스트는 결합을 다시 실행하거나 소스를 다시 생성할 필요가 없습니다.
