# Myriad

Myriad is a code generator, it takes an arbitary file and the library provides different mechanisms to allow F# code to be produced in response to the file, whether that file be an F# source file or a simple text file.  

Myriad can be used from either an MSBuild extension or from its CLI tool.

The idea behind Myriad is to un-complicate, as far as possible, the ability to generate and do meta-programming in F#. By meta-programming in F# I mean generating actual, readable F# code like discriminated unions and records, not just IL output.

Myriad is an evolution of the ideas I developed while working with F#'s type providers and other meta-programming functionality like quotations and AST manipulation. Myriad aims to make it easy to extend the compiler via Myriad plugins. Myriad provides an approach to compiler extension that isn't modifying or adjusting Type Providers or waiting a long time for other F# language improvements. You write a Myriad plugin that works on a fragment of AST input, and the plugin supplies AST output with the final form being source code that is built into your project. This lets the compiler optimise generated output in addition to allowing tooling to operate effectively.

![Build](https://github.com/MoiraeSoftware/myriad/workflows/Build/badge.svg)

[![ko-fi](https://www.ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/K3K115UYS)

- - -

## MSBuild usage

To use Myriad via its MSBuild support you add the `Myriad.Core` and `Myriad.Sdk` package references:
```xml
    <ItemGroup>
      <PackageReference Include="Myriad.Core" Version="0.5.0" />
      <PackageReference Include="Myriad.Sdk" Version="0.5.0" />
    </ItemGroup>
```

An input file is specified by using the usual `Compile` element:
```xml
<Compile Include="Generated.fs">
    <MyriadFile>Library.fs</MyriadFile>
</Compile>
```

This is configuring Myriad so that a file called `Generated.fs` will be included in the build using `Library.fs` as input to the Myriad.  

Myriad works by using plugins to generate code.  A plugin called fields is included with Myriad which takes inspiration from OCaml's [ppx_fields_conv](https://github.com/janestreet/ppx_fields_conv) plugin of the same name.

The input file in this example `Library.fs` looks like this:
```fsharp
namespace Example
open Myriad.Plugins

[<Generator.Fields "fields">]
type Test1 = { one: int; two: string; three: float; four: float32 }
type Test2 = { one: Test1; two: string }
```

Attribute's are use so that the code generator knows which parts of the input AST are to be processed by the plugin. If you had several records and you only want the fields plugin to operate on `Test1` then the attribute would be used like in the example to only apply `Generator.Fields` to `Test1`. Note, if you wanted a plugin that just needs the whole input AST then there is no need to provide an input. Myriad aims to be a library rather than a full framework that ties you to the mechanism used to input and generate code.  The parameter passed to the attribute "fields" specifies the configuration section that is used for the plugin in the `myriad.toml` file.   In this instance fields is used and the `myriad.toml` file is as follows:

```toml
[fields]
namespace = "TestFields"
```

This specifies the namespace that is used for the plugin, which in this case is "TestFields".  

The fields plugin in this example will generate the following code at pre-build time and compile the code into your assembly:
```fsharp
//------------------------------------------------------------------------------
//        This code was generated by myriad.
//        Changes to this file will be lost when the code is regenerated.
//------------------------------------------------------------------------------
namespace rec TestFields

module Test1 =
    open Example

    let one (x : Test1) = x.one
    let two (x : Test1) = x.two
    let three (x : Test1) = x.three
    let four (x : Test1) = x.four

    let create (one : int) (two : string) (three : float) (four : float32) : Test1 =
        { one = one
          two = two
          three = three
          four = four }

    let map (mapone : int -> int) (maptwo : string -> string) (mapthree : float -> float) (mapfour : float32 -> float32) (record': Test1) =
      { record' with
          one = mapone record'.one
          two = maptwo record'.two
          three = mapthree record'.three
          four = mapfour record'.four }
```

The fields plugin generates a `map` for each field in the input record, a `create` function taking each field, and a `map` function that takes one function per field in the input record.

The map functions for each field are useful in situations where you just want to use a single field from a record in a lambda like a list of records:
```fsharp
let records = [{one = "a"; two = "aa"; three = 42.0; four = 172.0f}
               {one = "b"; two = "bb"; three = 42.0; four = 172.0f}]
 records |> List.sortBy Test1.one
```

### Lenses

Myriad can also generate [lenses](https://fsprojects.github.io/FSharpPlus/tutorial.html#Lens) for records and single-case discriminated unions.
Lens is a _pair_ of a `getter` and a `setter` for one property of the type. Given the object `Lens` allows you to get the value of the property or to update it, creating a new object. The advantage of _lenses_ is an ability to combine them to read or update nested fields of the object.

To create lenses for your type, first annotate the type for which you want lenses to be generated with `Generator.Lenses` attribute:

```fsharp
[<Generator.Lenses("lens")>]
type Record =
    { one: int
      two: string }
```

Myriad will generate the following code:

```fsharp
module RecordLenses =
    let one = (fun (x: Test1) -> x.one), (fun (x: Test1) (value: int) -> { x with one = value })
    let two = (fun (x: Test1) -> x.two), (fun (x: Test1) (value: string) -> { x with two = value })
```

Often lenses are defined as a single-case union around a pair of getter and setter. Myriad is also capable of adding the invocation of such DU's constructor.

To achieve this, decorate your type with the `Lens` attribute, specifying the name of the DU constructor: `[<Generator.Lenses("Lens")>]`, and Myriad will generate this code:

```fsharp
module RecordLenses =
    let one = Lens((fun (x: Test1) -> x.one), (fun (x: Test1) (value: int) -> { x with one = value }))
    let two = Lens((fun (x: Test1) -> x.two), (fun (x: Test1) (value: string) -> { x with two = value }))
```

You can provide the name of DU constructor in several ways:
- As a string: `[<Generator.Lenses("lens", "Lens")>]`;
- Or as a type: `[<Generator.Lenses("lens", typedefof<Lens<_, _>>)>]` or `[<Generator.Lenses(typeof<Lens<_, _>>)>]`.

If the `Lens` type is in different namespace/module than the type decorated with the attribute, provide the full name of the `Lens` constructor: `[<Generator.Lenses("Namespace.And.Module.Of.Lens")>]`.

---

The full fsproj is detail below:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net5.0</TargetFramework>
    </PropertyGroup>
    <ItemGroup>
        <Compile Include="Library.fs" />
        <Compile Include="Generated.fs">
            <MyriadFile>Library.fs</MyriadFile>
        </Compile>
    </ItemGroup>
    <ItemGroup>
      <PackageReference Include="Myriad.Core" Version="0.5.0" />
      <PackageReference Include="Myriad.Sdk" Version="0.5.0" />
    </ItemGroup>
</Project>
```

## Plugins

Plugins for Myriad are supplied by simply including the nuget package in your project. The nuget infrastructure supplies the necessary MSBuild props and targets so that the plugin is used by Myriad automatically. Following the source for the fields plugin can be used as reference until more details about authoring plugins is created.

### Using external Plugins

To consume external plugins that aren't included in the `Myriad.Plugins` package, you must register them with Myriad. If you are using the CLI tool then the way to do this is by passing in the `--plugin <path to dll>` command-line argument. If you are using MSBuild then this can be done by adding to the `MyriadSdkGenerator` property to your project file:

```xml
<ItemGroup>
    <MyriadSdkGenerator Include="<path to plugin dll>" />
</ItemGroup>
```

For example, if you had a project layout like this:

```
\src
-\GeneratorLib
 - Generator.fs
 - Generator.fsproj
-\GeneratorTests
 - Tests.fs
 - GeneratorTests.fsproj
```

You would add the following to Generator.fsproj:
```xml
  <ItemGroup>
    <Content Include="build\Generator.props">
      <Pack>true</Pack>
      <PackagePath>%(Identity)</PackagePath>
      <Visible>true</Visible>
    </Content>
  </ItemGroup>
```

Then add a new folder `build` with the `Generator.props` file within:
```xml
<Project>
    <ItemGroup>
        <MyriadSdkGenerator Include="$(MSBuildThisFileDirectory)/../lib/netstandard2.1/Generator.dll" />
    </ItemGroup>
</Project>
```

Often an additional props file (In this sample the file would be `Generator.InTest.props`) is used to make testing easier.  The matching element for the tests `.fsproj` would be something like this:

```xml
<Project>
    <ItemGroup>
        <MyriadSdkGenerator Include="$(MSBuildThisFileDirectory)/../bin/$(Configuration)/netstandard2.1/Generator.dll" />
    </ItemGroup>
</Project>
```

Notice the Include path is pointing locally rather than within the packaged nuget folder structure.

In your testing `fsproj` you would add the following to allow the plugin to be used locally rather that having to consume a nuget package:

```xml
<!-- include plugin -->
<Import Project="<Path to Generator plugin location>\build\Myriad.Plugins.InTest.props" />
```

## Debugging

To debug Myriad, you can use the following two command line options:

* `--verbose` — write diagnostic logs out to standard out
* `--wait-for-debugger` — causes Myriad to wait for a debugger to attach to the Myriad process

These can be triggered from MSBuild by the `<MyriadSdkVerboseOutput>true</MyriadSdkVerboseOutput>` and `<MyriadSdkWaitForDebugger>true</MyriadSdkWaitForDebugger>` properties, respectively.

## Nuget
The nuget package for Myriad can be found here:
[Nuget package](https://www.nuget.org/packages/myriad/).

## How to build and test

1. Make sure you have .Net Core SDK installed - check required version in [global.json](global.json)
2. Run `dotnet tool restore`
3. Run `dotnet fake build`

## Documentation

### How to generate docs

1. Make sure you have .Net Core SDK installed - check required version in [global.json](global.json)
2. Run `dotnet tool restore`
3. Run `dotnet fake build -t Generate docs`

### Working with docs in Watch Mode

1. Make sure you've generated docs as described above
2. `cd docs`
3. `dotnet fornax watch` - this will start local web server hosting documentation regenerating changes on any file edit.

### Publishing docs

1. Docs will be published automatically by GitHub Action into `gh-pages` branch on any pushed git tag


## How to release new version

1. Update [CHANGELOG.md](./CHANGELOG.md) by adding new entry (`## [0.X.X]`) and commit it.
2. Create version tag (`git tag v0.X.X`)
3. Run `dotnet fake build -t Pack` to create the nuget package and test/examine it locally.
4. Push the tag to the repo `git push origin v0.X.X` - this will start CI process that will create GitHub release and put generated NuGet packages in it
5. Upload generated packages into NuGet.org

### Also see
* [Applied Metaprogramming with Myriad And Falanx](https://7sharp9.github.io/fsharp/2019-04-24-applied-metaprogramming-with-myriad/)
* [Myriad Intro](https://7sharp9.github.io/fsharp/2019-11-06-myriad-intro/)
