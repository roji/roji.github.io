---
title: C# 8 Nullable Reference Types, old TFMs and Multitargeting
permalink: /nullable-reference-types-with-old-tfms
modified: 2020-01-04
comments: true
---
C# 8.0 finally brought us nullable reference types (NRTs), which us to annotate our reference types as non-nullable and get compiler warnings for code that may be in violation. As libraries and applications in the .NET ecosystem opt into this feature, C# code will get safer and more self-documenting, as it's immediately clear which variables can hold null and which can't. [Here are the C# docs for NRTs](https://docs.microsoft.com/en-ca/dotnet/csharp/nullable-references), and you may also want to check out [this blog post to get started](https://devblogs.microsoft.com/dotnet/try-out-nullable-reference-types/).

There's one problem though: C# 8.0 is only supported when targetting at least .NET Core 3.0 or .NET Standard 2.1, so if your project has to target an older TFM (say, .NET Standard 2.0 or even .NET Framework), you can't officially use this feature. Unfortunately, some of us maintain software that can't always target the newest shiny thing, but we'd still like to get the benefits of NRTs. No problem! As this is a compiler-only feature without any runtime requirements, there's nothing *really* preventing you from turning it on in your csproj:

```xml
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>8.0</LangVersion>
    <Nullable>enable</Nullable>
  </PropertyGroup>
```

And this will actually work! Until it doesn't, that is... In some rare cases, things won't work as they should when targeting the old TFM:

```c#
static string Foo(string? s)
{
    Debug.Assert(s != null);
    return s;
}
```

Since `s` is a nullable string, using it in a non-nullable context will generate a warning. Now, let's say that we know that in this particular context, `s` cannot be null, and wish to assert that. This code will compile just fine on recent TFMs, since the compiler knows that if `Debug.Assert` returns successfully, `s` can't be null. However, when targeting an older TFM, this code will generate a warning. To be fair, this is a relatively rare corner case: most NRT code does compile correctly even on older BCLs.

A way around this is to target a newer TFM where NRTs are fully supported, and simply disable nullability on older one; in effect, we'll be using the newer TFM to do the compiler verifications that our code is null-correct. So we can simply modify our csproj to do the following:

```xml
  <PropertyGroup>
    <TargetFrameworks>netstandard2.0;netstandard2.1</TargetFrameworks>
    <LangVersion>8.0</LangVersion>
  </PropertyGroup>

  <PropertyGroup Condition=" '$(TargetFramework)' != 'netstandard2.0' ">
    <Nullable>enable</Nullable>
  </PropertyGroup>
```

Great! There's just one problem - our build will now generate warning CS8632 - The annotation for nullable reference types should only be used in code within a '#nullable' annotations context - since in the older TFM we're using the NRT feature without having it turned it on. No problem, we can just ignore that warning for the old TFM by adding the following:

```xml
<PropertyGroup Condition=" $(Nullable) != 'enable' ">
  <NoWarn>$(NoWarn);CS8632</NoWarn>
</PropertyGroup>
```

That's it. You now have a project targeting two TFMs, with NRTs enabled on the newer TFM and disable on the older. Happy fun nullificating your projects!