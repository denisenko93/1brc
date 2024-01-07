# 1️⃣🐝🏎️ The One Billion Row Challenge

.NET implementation of https://github.com/gunnarmorling/1brc

> If you want to compare results on the same hardware, open a PR with your implementation. Add it to a directory `1brc_githugusername`. This directory must contain a dotnet project `1brc.csproj` (not renamed) and your code. You program must accept the first argument as the path to the measurements file.
>
> This should work from the repo dir:
> dotnet build 1brc_githugusername/1brc.csproj -c Release
> dotnet run --project 1brc_githugusername/1brc.csproj -c Release --no-build -- path/to/the/file.txt


Note that his implementation supports `\r\n` line endings. The numbers in the Evolution section below are from Windows, where the input file is 13.7GB vs 12.8GB. It's generated by the upstream Java code on Windows so this is an implicit requirement for x-plat already.

## Results

_The performance is measured on an idle 6C/12T Alder Lake CPU fixed at 2.5 GHz (no turbo), 32GB DDR4 3200, Debian 12 in LXC. At least 5 runs, showing the best result._

**Jan 6, 6 PM UTC**
 * The top two Java results (reported as `< 8.0`): **~4.1 sec**
 * This code: **4.59 sec** 🥉
 * Current No 3 Java (`9.625`): **4.75 sec** (Graal JVM), **4.83 sec** (Graal native image)

## Evolution

Below is the evolution of results with each commit. The time shown here is measured inside the app, on Windows. Using `time` command adds up to `0.5-0.9 sec`. 

#### First attempt

Mmap + paralell using Span API and some unsafe tricks to avoid Utf8 parsing until the very end.

```
Processed in 00:00:10.6978618
Processed in 00:00:10.8473143
Processed in 00:00:10.9107262
Processed in 00:00:10.9733218
Processed in 00:00:10.5854176
```

#### Some micro optimizations

```
Processed in 00:00:09.7093471
```

Float parsing is ~57%, dictionary lookup is ~24%. Optimizing further is about those two things. We may use `csFastFloat` library and a specialized dictionary such as `DictionarySlim`. However the goal is to avoid dependencies even if they are pure .NET.

It's near-perfectly parallelizable though. On 8 cores it should be 33% faster than on 6 that I have. With 32GB RAM the file should be cached by an OS after the first read. The first read may be very slow in the cloud VM, but then the cache should eliminate the difference between drive speeds.


#### Use naive double parsing

If we can assume that the float values are well formed then the speed almost doubles.

```
Processed in 00:00:05.5519479
```

#### Optimized double parsing with fallback

No assumptions are required if we fallback to the full .NET parsing implementation on any irregularity.

```
Processed in 00:00:05.2944041
Processed in 00:00:05.3489315
```

#### Cache powers of 10, inline summary.init

```
Processed in 00:00:04.7363095
Processed in 00:00:04.8472097
Processed in 00:00:04.8235814
Processed in 00:00:04.7163938
```

#### Microoptimize float parsing, but keep it general purpose

```
Processed in 00:00:04.4547973
Processed in 00:00:04.5303938
Processed in 00:00:04.5125394
```

#### Optimize hash function

See comments in Utf8Span.GetHashCode

```
Processed in 00:00:04.2237865
Processed in 00:00:04.2524434
Processed in 00:00:04.2688423
```

#### Use specification to use int parsing and branchless min/max

Processed in 00:00:03.9916535
Processed in 00:00:03.9897462
Processed in 00:00:03.9810353


#### Set dictionary capacity

Set dictionary capacity to 10k.

```
`\n` only
Processed in 00:00:03.2530659
Processed in 00:00:03.1561451

`\r\n`
Processed in 00:00:03.3463769
Processed in 00:00:03.3641962
Processed in 00:00:03.3762491
```


#### Manual SIMD: find boundaries and simplify parsing

Use AVX2-optimized `IndexOf`.

Find boundaries for `;`, `.` and `\n`.

Optimize/simplify int parsing between `;`and `.` and use a single digit that is always after `.`.

```
Processed in 00:00:03.0687066
Processed in 00:00:03.0819551
```