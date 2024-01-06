# 1️⃣🐝🏎️ The One Billion Row Challenge

.NET implementation of https://github.com/gunnarmorling/1brc

Note that his implementation supports `\r\n` line endings. All numbers here are from Windows, where the input file is 13.7GB vs 12.8GB. It's generated by the upstream Java code so this is an implicit requirement for x-plat already.

**EARLIER PERFORMANCE RESULTS ARE NOT COMPARABLE**
I was measuring the time inside the app. People are afraid of the startup time, but I found that exiting is slow. With `time` the result is `~4.9 sec`. It would still be the top for the first two days (when the best reported was 12+), now it's around 2/3.

~~This code runs in **3.98 sec** on `6 cores i5-12500/64GB RAM (30GB free)/Firecuda 530` on Windows. With only `\n` line endings it takes **3.67 sec**. The numbers are ~same with AOT.~~

The top Java version reported as _7.99 sec_ by @royvanrijn takes **4.75 sec**. (Regardless on line endings) on my machine.

The second Java version reported as _9.625 sec_ fails on `\r\n` with segfault. With only `\n` endings it takes **5.00 sec**.

~~I tried to install Graalvm and set JAVA_HOME/PATH as instructed, restarted IDEs and consoles as instructed, but I cannot tell if it worked or not. I think it's against the spirit on the task to use a custom JVM. But I do not see a big difference, maybe it's not so good on Windows.~~

## Results

Below is the evolution of results with each commit. The time show is measured inside the app. Using `time` command adds up to `0.9 sec` (mostly for existing).

**First attempt**

Mmap + paralell using Span API and some unsafe tricks to avoid Utf8 parsing until the very end.

```
Processed in 00:00:10.6978618
Processed in 00:00:10.8473143
Processed in 00:00:10.9107262
Processed in 00:00:10.9733218
Processed in 00:00:10.5854176
```

**Some micro optimizations**

```
Processed in 00:00:09.7093471
```

Float parsing is ~57%, dictionary lookup is ~24%. Optimizing further is about those two things. We may use `csFastFloat` library and a specialized dictionary such as `DictionarySlim`. However the goal is to avoid dependencies even if they are pure .NET.

It's near-perfectly parallelizable though. On 8 cores it should be 33% faster than on 6 that I have. With 32GB RAM the file should be cached by an OS after the first read. The first read may be very slow in the cloud VM, but then the cache should eliminate the difference between drive speeds.


**Use naive double parsing**

If we can assume that the float values are well formed then the speed almost doubles.

```
Processed in 00:00:05.5519479
```

**Optimized double parsing with fallback**

No assumptions are required if we fallback to the full .NET parsing implementation on any irregularity.

```
Processed in 00:00:05.2944041
Processed in 00:00:05.3489315
```

**Cache powers of 10, inline summary.init**

```
Processed in 00:00:04.7363095
Processed in 00:00:04.8472097
Processed in 00:00:04.8235814
Processed in 00:00:04.7163938
```


**Microoptimize float parsing, but keep it general purpose**

```
Processed in 00:00:04.4547973
Processed in 00:00:04.5303938
Processed in 00:00:04.5125394
```

**Optimize hash function**

See comments in Utf8Span.GetHashCode

```
Processed in 00:00:04.2237865
Processed in 00:00:04.2524434
Processed in 00:00:04.2688423
```


**Use specification to use int parsing and branchless min/max**

Processed in 00:00:03.9916535
Processed in 00:00:03.9897462
Processed in 00:00:03.9810353