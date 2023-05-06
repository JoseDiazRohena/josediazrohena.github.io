---
layout: post
---

Editorial note
==============
You may notice that some of these posts are numbered. Those numbers correspond to the PR number in the [Valentine repo](https://github.com/tote-bag-labs/valentine). I'm going to stop doing that because it's a bit confusing.


Custom Fonts
============

[#43 - Improve custom font handling](https://github.com/tote-bag-labs/valentine/pull/43) landed before I started this blog, but I think it's an interesting topic. And probably the first thing I've shared that could be of direct use to you.

With help from [baconpaul](https://github.com/baconpaul), Valentine was finally building and running in Linux. However, it failed strict pluginval validation. Juce's leak checker was asserting around leaks of `glyph` objects. My first thought was that it had something to do with the code shown below. Valentine uses a custom typeface which is created and then held as a `static` variable. `static` was used here in order to ensure that the typeface is only created once in a context where we don't have much control over when the font may be requested (or otherwise instantiated).

```cpp
// Valentine LookAndFeel.cpp

include "BinaryData.h"

namespace
{
const juce::Font getCustomFont()
{
    static auto typeface = juce::Typeface::createSystemTypefaceFor (BinaryData::MontserratMedium_ttf,
                                                                    BinaryData::MontserratMedium_ttfSize);
    return juce::Font (typeface);
}
}
```

Globals Are Annoying
====================

I did't have control over when this function is invoked because the of how I coupled my custom font implementation to singleton and singleton-esque objects. When do they come into existence? Who is responsible for that? Who knows?

This is how it works. We override `juce::LookAndFeel::getTypefaceForFont` in our look and feel. Our overriden function calls `getCustomFont()`. But `getTypefaceForFont` only ever gets called for the _default_ look and feel, which is owned by `juce::Desktop`. For all this to work, we need to, then, set our look and feel as the default. This introduces some problems for us. As with any object with global scope, it is hard to reason about when the look and feel will be destroyed. Especially when we consider `juce::Desktop` is itself a global object. Honestly, the fact that the class is called `Desktop` and the fact that it is a Singleton object should be enough of an indication that we shouldn't be doing this.

But there's more! Since the typeface we created has `static` storage duration, it's also hard to reason about when it is deleted—especially when several instances of the plugin are open. It's possible (probable) that the `Font` was merely being deleted after Juce's leak checker does its thing. The logic would be much easier to understand if we didn't rely on complex library code with global state. And of course, if we could say for sure when our object would be destroyed in relation to other potentially dependent objects.

On top of that, doing it this way allows us (me, sorry) to write more unclear code. Consider this:

```cpp
juce::Font LookAndFeel::getLabelFont (juce::Label& l)
{
    return { static_cast<float> (l.getHeight()) };
}
```

It's impossible to understand from this code how our custom font is applied. This is because it relies on a `juce::Font` constructor that uses eventually calls our `getTypefaceForFont` function.

Implementing a proper custom font class allows us to write this more clearly.

```cpp
juce::Font LookAndFeel::getLabelFont (juce::Label& l)
{
    const auto fontHeight = static_cast<float> (l.getHeight());
    return fontHolder.getFont("MontserratMedium_ttf").withHeight (fontHeight);
}
```

Comparing these two examples you might also see something else: the old version doesn't have a way of using multiple fonts.

A Custom Font Holder
====================

I put the "custom fonts how" question to the [Audio Programmer discord](https://www.theaudioprogrammer.com/discord), and [Eyal Amir](https://www.modalics.com/) came to the rescue with an implementation of a font holding class that is resource efficient, has clear object lifetimes, and allows for accessing multiple fonts in a straightforward manner.

```cpp
// tbl_font.h

#include "BinaryData.h"

#include <juce_gui_basics/juce_gui_basics.h>

#include <map>

#pragma once

namespace tote_bag
{

/** A class that holds our custom fonts.
 */
class FontHolder
{
    public:
    /** Gets a Font from the map, creating it if it doesn't exist.
     */
    juce::Font getFont(const juce::String& fontName);

    private:
    std::map<juce::String, std::unique_ptr<juce::Font>> fonts;
};

} // namespace tote_bag
```

```cpp
// tbl_font.cpp
#include "tbl_font.h"

namespace tote_bag
{

juce::Font FontHolder::getFont(const juce::String& fontName)
{
    auto fontIt = fonts.find(fontName);
    if (fontIt == fonts.end())
    {
        int size = 0;
        const auto resource = BinaryData::getNamedResource (fontName.toRawUTF8(), size);

        // Make sure you've passed the correct font name to this function.
        // If you're sure you've done that, make sure you've added the font
        // to your assets dir and BinaryData files.

        // NOTE: I just found a bug in my implementation. This should probably return juce::Font{} after
        // asserting. Otherwise, this could crash a release build. Then again, it seems *very* hard
        // to not notice that you weren't about to get the Font resource in debug...
        jassert(resource != nullptr);

        auto typeface = (juce::Typeface::createSystemTypefaceFor (resource,
                                                                  static_cast<size_t>(size)));

        auto font = std::make_unique<juce::Font> (typeface);

        const auto[newFontIt, inserted] = fonts.insert({fontName, std::move(font)});
        if(!inserted)
        {
            jassertfalse;
            return juce::Font{};
        }

        fontIt = newFontIt;
    }

    return *fontIt->second;
}

} // namespace tote_bag
```

It's a simple class. Fonts are held in a map, and instantiated as a unique pointer the first time they are requested via `getFont`. All you need is the font name, which is defined in the `BinaryData` file created when adding the Font to a Juce project. `FontHolder` is owned by our look and feel class—so we know when it and the fonts it holds will be destroyed. No leaks!
