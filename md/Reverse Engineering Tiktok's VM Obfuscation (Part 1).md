> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.nullpt.rs](https://www.nullpt.rs/reverse-engineering-tiktok-vm-1)

> TikTok has a reputation for its aggressive data collection. The platform has implemented various meth......

TikTok has a reputation for its aggressive data collection. In fact, an article published on 22 December 2022 [uncovered how ByteDance spied on multiple Forbes journalists using TikTok](https://www.forbes.com/sites/emilybaker-white/2022/12/22/tiktok-tracks-forbes-journalists-bytedance/?sh=410b113b7da5). While some of the data they collect may seem benign, it can be used to build a detailed profile of each user. Information such as user location, device type, and various hardware metrics are combined to create a unique "fingerprint" that can potentially be used to track a user's activity on and off the app. This data may also be used to prevent their APIs from being utilized in automated scripts by ensuring that the data from the requests seem humanlike.

The platform has implemented various methods to make it difficult for reverse-engineers to understand exactly what data is being collected and how it is being used. Analyzing the call stack of a request made on tiktok.com can begin to paint the picture for us. Let's start by doing a search for the term"food". Upon pressing enter, TikTok sends off a GET request with our search term and some extra telemetry embedded.

The response for this request is exactly what we'd expect: The JSON representation of accounts starting with or containing the keyword food.

Most of the query parameters are self explanatory but there's three that stand out:

*   msToken
*   X-Bogus
*   _signature

Removal of the `_signature` query parameter doesn't seem to have an affect as the request still goes through as expected but removal of any other parameter causes TikTok to give a 0 length response.

How are these parameters generated? Taking a look at the call stack tells us the journey from beginning to end.

The call to window.fetch being located in script `secsdk-lastest.umd.js` tells us that the [fetch](https://developer.mozilla.org/en-US/docs/Web/API/fetch) function has been monkey patched to provide additional functionality but perhaps what's more interesting are the obfuscated function names underneath.

An examination of the `webmssdk.js` script reveals that the code is intentionally made difficult to understand through obfuscation, as evidenced by the following function:

View the fully obfuscated script over at [webmssdk.js](https://sf16-website-login.neutral.ttwstatic.com/obj/tiktok_web_login_static/webmssdk/1.0.0.1/webmssdk.js)

By utilizing the [Babel suite](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md), we are able to parse the source code and manipulate its Abstract Syntax Tree (AST). With this, we can create a simple transformation that reduces complex binary expressions to a single constant. The transformation code appears as follows:

The function, previously obfuscated, now appears in the following form:

Much better, but now we're stuck in ternary hell. We can create another simple transformation to unpack the nested ternary logic and make it more easily understood:

Applying this transformation to function _0x4e353d produces the following result:

We could create more complex transformations to further improve the readability of the obfuscated script, but for the purposes of this article, these two transformations are sufficient.

As you review the script, you may notice recurring patterns. For example, consider these two function calls:

They follow a very similar schema: A function call with 3 parameters:

1.  A string of alphanumeric characters that is not immediately recognizable as to its purpose.
2.  An object containing getters and setters referencing various browser APIs and global variables.
3.  void 0 (a fancy obfuscated way of saying undefined)

An exercise to you: Dump all function calls that meet the criteria listed above

To determine how this string is being used, we need to analyze the function it is being called in.

We can immediately see that the function we deobfuscated earlier is defined within the `_0x8d6b0f` function. Additionally, the argument names have been made more readable for ease of understanding.

The first 16 characters are evenly split into two parts and then converted into an integer from base 16. The result is then compared to two magic constants: `1213091658` and `1077891651`. Applying this logic to our string will result in it passing these checks.

A check for a 00 separator follows immediately after. While we are still unsure of the exact purpose of this string, we have determined how it should start.

Characters 24-34 are divided into parts and used in some bitwise arithmetic that is calculated for the variable `_0x4fab32`. Searching for the use of this variable leads us to a call to the `String#fromCharCode` function, where it is XORed with another variable.

This strongly suggests the use of an [XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher), leading me to conclude that variable `_0x4fab32` is likely the key. Based on this discovery, we can infer the purpose of nearby variables. The decryption process now looks as follows:

With all the necessary pieces in place, we can now isolate this logic and potentially retrieve strings from the previously mentioned long and obfuscated string. I chose to implement this in TypeScript and run it using Node, but the logic can be implemented in any language of your choosing.

If we run our script using the initial bytecode:

We obtain the following output:

Great, we were able to successfully extract all strings from this particular module. We even see the strings `_signature` and `X-Bogus`! If we run our script using the strange string from the second function, we obtain a completely separate set of strings.

This is because each "weird string" is actually bytecode that is interpreted and executed by TikTok's custom virtual machine to perform various tasks. Many modules handle bot protection and fingerprinting in their own ways.

For instance, this module is responsible for managing canvas fingerprinting, in which a user's machine's rendering of an HTML5 canvas element is used to create a fingerprint for them:

Here are the strings for TikTok's WebGL module, which can be used to gather your vendor and other GPU information:

And here's a demo of that in action (This is using JavaScript to pull your GPU information):

This article does not delve into the specifics of how these strings are utilized or how TikTok interprets the rest of the bytecode through its custom virtual machine and various opcodes. If that is something you are interested in, keep an eye out for the second part of this series :)

If you're interested in a full strings dump check out [strings.txt](https://gist.github.com/voidstar0/d4d409321ca0a32e2ffd295b59a9a1df)

[Mastodon (@voidstar@infosec.exchange)](https://infosec.exchange/@voidstar)

[Twitter](https://twitter.com/blastbots)Discord: veritas#0001Email: f @ nullpt.rs