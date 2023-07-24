---
layout: post
---

When I started development on Valentine, I spent a lot of time making music with it. Mostly, I used it on drum loops, since I was mostly interested in getting cool rythmic effects. As my ideas about the algorithm solidified, I turned to other matters, only listening to make sure I hadn't broken anything. Now that I'm nearing a 1.0 release, I find myself actually using Valentine for music again. Doing so for about half an hour inspired some changes. Most of them were straightforward to implement. One of them threatened to become about a bunch of other things. I just wanted to be able to [turn the output clipper off](https://github.com/tote-bag-labs/valentine/pull/76/)!

I just want to be able to turn the output clipper off
=====================================================

<p align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/iVwUOAZrbjk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</p>

I have "Talisman" loaded into my default Live set. I use it as a stand in for cases where you have the compressor on the master bus or when treating samples. The results were ok but keyboards sounded too distorted for what I was trying to do. Fixing this won't a huge deal, right? Just make a parameter for enabling the clipper. Wrong!


```cpp

    // If only it were so simple...
    if (clipOn.get())
    {
        boundedSaturator->processBlock (highSampleRateBlock);
    }

```

The issue here is not turning the clipper off. It's everything that has to change when the clipper is turned off. This clipper, as noted [in an earlier post](/2023/05/01/57-improving-saturators.html), is implemented using first order antiderivative antialiasing, which adds .5 samples of latency to processing. This was accounted for when calculating the amount latency compensation to apply in aligning the processed and clean buffers. However, latency calculation and compensation setting was being done under the assumption that it couldn't change while processing.


Tactical decision making
========================

This assumption no longer held, so what would have been a simple addition now required a refactor. I now (thought I) needed delays that could be changed smoothly during processing. This implied allowing fractional delay—even for the clean buffer, which would never be actually set to a fractional delay but need to smoothly change between settings. I now also needed to account for the possibility of fractional delays of more than one sample. I wasn't ready for this. 

The processed buffer used a Thiran Allpass to apply a fractional delay of less than a sample. The clean buffer used a simple non-interpolated delay. The Thiran Allpass isn't great for changing during processing, and I didn't have delay line that with encapsulated interpolation. However, JUCE has one, with a handy way of choosing interpolation type to a changing-while-processing-friendly Lagrange interpolation. The only problem was that using the JUCE implementation ran counter to my goal of getting JUCE dsp out of my projects[^1]. There were some other, less pressing problems: this solution wouldn't use the Thiran Allpass implementation I had spent so much time improving. Perhaps more important to my ego—shouldn't I be able to write my own interpolated delay? I already had an interpolator and a delay, after all. All of these concerns didn't match up to the fact that I was interested in fixing latency compensation _now_. I work on Valentine in my spare time, and refactoring my delays to work for this use case could present weeks of delay. The result would be an implementation that would, in the end, sound exactly the same as the JUCE implementation, exist for the time being in a file that already has an unbreakable JUCE dependency, and probably not have as nice of an interface. I decided, then, to get on with it and use the JUCE delays. Which work great.

Another bit of tactics: the function I wrote to change latency compensation on demand is kind of complex. Another goal I have for Valentine is to not add more functionality without also adding tests. In this case, getting the function under test would require refactoring and dependency breaking. I decided to get the feature sooner and file an issue for the added [tech debt](https://github.com/tote-bag-labs/valentine/issues/92).

Overthinking it?
===============

Dependencies aside, refactoring the code to adjust latency during processing was kind of straightforward. Whenever the sample rate changes, `AudioProcessor::prepareToPlay()` is called and there we find the maximum possible processing latency. That value, rounded up, is the latency we will report to the host. It's also the amount of delay we will apply to the clean buffer.

```cpp
    // Find the maximum possible processing delay, and round up. That will be
    // the unprocessed buffer delay and the delay we report to host.
    // This maximum delay should only change if the oversampling latency called—
    // e.g. when the sample rate changes and prepareToPlay is called.
    const auto maximumDelay =
        overSamplingLatency + maximumSaturatorLatency + maximumClipperLatency;
    cleanBufferDelay = static_cast<int> (std::ceil (maximumDelay));
```

Then, we find the delay we need to apply to the processing buffer by finding the processing latency based on current settings and subtracting that value from the total maximum latency. We'll do this whenever the processing graph changes in such a way that the processing latency changes.

```cpp
    // Use saturator enablement to find current latency
    const auto currentClipperLatency = clipOn.get()
                                     ? maximumClipperLatency : 0.0f;
    const auto currentSaturatorLatency =
        saturateOn.get() ? maximumSaturatorLatency : 0.0f;
    const auto currentDelay =
        overSamplingLatency + currentSaturatorLatency + currentClipperLatency;

    // We need to delay the processed signal by the difference between its
    // processing latency and the delay applied to unprocessed signal
    const auto processBufferDelay = cleanBufferDelay - currentDelay;
```

Since the clean buffer delay is always an integral number of samples, I actually could have kept the its delay implementation as it was. But I went through a few iterations before figuring out that I didn't actually need to change that delay time (whoops)[^2]. As for the processed delay, we use the amount we just calculated to set a new delay value (with value smoothing, of course).


Did it make a difference? 
=========================

While I was doing all this work, I had an evil though: "It's just half a sample? Who cares about half a sample? Who will hear the difference". I will, I think. I didn't direcly A/B the changes, but the following makes me feel that it's the right call.

- Given how much time I spent fixing my Allpass implementation when I realized it wasn't correctly delaying the processed buffer. In doing that, I _did_ A/B test this change, determing that I could hear subsample differences in delay between the processed and clean buffers. It would make sense to continue caring about accuracy in this respect. 

- I've been playing around with drums and am finding it easy to dial in a blend between really smashed processing and clean signal while retaining transient content. "Punch", they call it. This indicates to me that the status quo is resulting in good phase alignment between the processed and clean buffers.

- Not accounting for this difference doesn't scale. What happens when I decide to make the saturation processing optional as well? Or implement variable oversampling? 

And now we have some nice extra functionality: output clipping can be enabled to prevent overs if we have a slow attack and are really hitting an otherwise unprocessed signal, or if we aren't compressing much and simply want tanh flavor (lol). For tamer material, we can leave it off and avoid unnecessarily reducing transients.

Why I added an enable button to Saturation as well
================================================== 

This was kind of the tip of the iceberg, as I eventually found myself adding a similar control for [Saturation](https://github.com/tote-bag-labs/valentine/pull/85).


<p align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/Z-7-4r3J7BA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</p>

While performing the changes needed to make output clipping optional, I realized that they weren't quite enough. Some of the distortion I was hearing was due to saturation processing, which was always on. I attempted to resolve this by lowering the gain fed into the Saturator at its lowest parameter setting. Compressing a beat that samples the track above made me realize that I had to go further. Since compression is in our case achieved by increasing gain, always-on Saturation processing would make Valentine unuseable. That clean keyboard produced a haze of distortion that just sounded bad. So I made Saturation optional.

I don't have much to say about it. Thanks to the work I had already done, it was easy.



[^1]: There's nothing wrong with JUCE dsp besides taking on an unwanted dependency. 
[^2]: I learned this when trying an implementation that changed the clean buffer delay if changes to the processing graph would cause the maximum system delay to, when rounded up, change. I then called `AudioProcessor::setLatencySamples(newLatency)` to keep the host up to date. The result caused awful clicks and pops, which I traced to the fact that `prepareToPlay()`, with all the non-realtime safe stuff going on in there, was being called during processing. It turned out that this was because I was calling `setLatencySamples()` during processing. At first I thought "well, how do I report that the total latency changed?". The answer is: don't change the total latency.


