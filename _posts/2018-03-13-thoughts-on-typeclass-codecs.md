---
layout: post
title: Some thoughts on typeclass-based codecs
categories: code
---

There has recently been a bit of discussion on the PureScript slack channel about whether using typeclass-based codecs is a good idea or not. I used to be quite a fan of this approach, but these days I prefer not to use them.

For reference, when I say "typeclass-based codecs", I mean like using the `EncodeJson` and `DecodeJson` classes provided by [argonaut-codecs](https://github.com/purescript-contrib/purescript-argonaut-codecs) in PureScript, or `ToJSON` and `FromJSON` provided by [aeson](https://hackage.haskell.org/package/aeson) in Haskell.

The primary advantage of writing codecs in this style is that they're quick to write and generally make for simple looking definitions. Most of the codec is implicit, as below the top-level structure it can mostly be derived from instances for existing types. This is also part of the problem since it can encourage a form of "primitive obsession" – it's more convenient to write a codec that relies on types with existing instances, rather than creating a model that better represent the domain.

Another consideration is whether the format you're dealing with is something that will need maintaining heading into the future. If your app is writing files it needs to read later (or writing files that another app needs to read later for that matter) then changing the format is a big deal. However, if you're using classes, it can hide that fact - you can change parts of the model and not even have to touch the codecs sometimes, since the compiler will just figure out the new instances that the updated types require.

Codecs silently adapting to changes in the data seems like an advantage, until you find that the app no longer understands the old format when reading and is writing a new backwards-incompatible format. It's true that this can be avoided if you're very disciplined about changes to the model, but given that we're already using a language like PureScript or Haskell, we know that compilers are _much_ better than humans at keeping us on track with situations like this.

As well as inadvertently changing the format yourself, you also have to pay close attention whenever the library you're using is updated, as your serialization format is tied to the instances provided there. If the library alters the codec instances it provides in any way, you're going to have trouble again. This is probably a rare occurrence, but it's a definite possibility.

Classes also enable codecs to be entirely derived based on generics (PureScript) or templating (Haskell) - which just further compounds the problems mentioned here! (When the instance can be reduced to a one-liner  you're more likely to choose types that ensure it's possible, and changing the types will be even less likely to cause you to have to do anything with the codecs).

In summary, Chris Martin recently [put it pretty well](https://twitter.com/chris__martin/status/965761044470796290):

> serialization is boring, so people manage to convince themselves that "I _shouldn't_ write this code" is a consequence of "I don't _want_ to write this code"

It's definitely tedious to be more explicit in writing codecs, but if you're not writing a throwaway prototype, it's turned out not to be worth the trouble that comes later, for me at least.
