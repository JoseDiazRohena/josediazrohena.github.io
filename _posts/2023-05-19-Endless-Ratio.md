---
layout: post
---

I had a low moment trying to resize the buttons used to change the 
`Ratio` parameter and started thinking of ditching the chooser box
and using a knob instead.

Ratio, Knee, Threshold
======================

Valentine originally used a knob for the `Ratio` parameter. It was 
stepped, allowing for 5 descrete settings. A mentor gave me feedback on this. 
He didn't like using a knob for non-continuous controls. He preferred buttons, or even better, 
a continuous control.

At the time, I chose the former because I was pretty set on the idea of restricting
choice. I thought it would be easier to use Valentine if the user didn't have to think about
the difference between `5:1` and `6.21:1` compression—they could choose from ratios that I
had found to be useful. Adding a continuous control would also change how the compressor
could sound or be complex to implement in a way that didn't do so. This was because Valentine's 
threshold and knee are set according to the `Ratio` setting.

| Ratio   | Knee     | Threshold       |
| :-----: | :------: | :-------------: |
|   4:1   |  6.0 dB   |     -18.0 dB     |
|   8:1   |  3.84 dB   |     -14.0 dB     |
|   12:1   |  2.16 dB   |     -13.0 dB     |
|   20:1   |  .96 dB   |     -12.0 dB     |
|   1000:1   |  0.0 dB   |     -10.0 dB     |

These ratios weren't arbitrary—I got the general idea from an 1176 manual I 
found online.

<p align="center">
  <img src="/assets/images/1176_transfer_curve.png" alt="A plot of a compressor's transfer curve." />
  <p style="text-align: right;"><em>UREI 1176 Manual, 1989</em></p>
</p>

As compression ratio increases, the range over which the compressor transitions
to applying the full ratio (knee) decreases. The threshold at which the compressor
starts acting on the signal also increases. I thought this could sound interesting,
so I decided to try it. I liked the result, and became invested in keeping Valentine's
sound from changing too much in this respect.

<p align="center">
  <img src="/assets/images/valentine_screenshot_box.png" alt="A screenshot of Valentine's UI" />
</p>

Implementing the ratio parameter using buttons was a definite improvement—it made the parameter more clear
and allowed the user to get to any of the settings without progressing through the whole range. But
it introduced some UI problems. I was having trouble getting the text on the ratio chooser
but large enough to be readable while making the overall component make sense in the context
of Valentine's other components. Also, the chooser box, to be honest,
looked a bit _rudimentary_. At least more so than the knobs, which look ok.

<p align="center">
  <img src="/assets/images/valentine_screenshot_box_better.png" alt="A screenshot of Valentine's UI. The ratio box has
  has been resized, making the text larger." />
</p>

I eventually made the ratio box a little larger in [this pull request](https://github.com/tote-bag-labs/valentine/pull/50),
helping at least with the legibility issue. By then I had already resolved to try a continuous control. 



Don't Overthink It
==================

My first concern about implementing a continuous ratio parameter was ensuring that doing so
would not change the compressor's sound for settings that are presently available. That would
mean making sure that the knee and threshold values were correctly set. Originally, this was
done by determining which of the ratio settings was selected, and using that to index into
corresponding arrays holding the knee and threshold values.

```cpp
// ValentineAudioProcessor::parameterChanged

ratioIndex = static_cast<size_t> (newValue);
ffCompressor->setRatio (ratioValues[ratioIndex]);
ffCompressor->setKnee (kneeValues[ratioIndex]);
ffCompressor->setThreshold (thresholdValues[ratioIndex]);

```

If I was to implement a continuous ratio control, then, I would need to ensure that
the corresponding knee and threshold values were applied at the original ratio settings.
I took for granted that these values should change smoothly between the original settings.
I spent a lot of time asking Chat GPT about different kinds of spline interpolation, getting seemingly
plausible answers that seemed really complicated for the task.

I realized I was overthinking it, especially given that I might not actually
keep the change. Better for getting things done: let go of the existing
ratio/knee/threshold relationship and linearly remap `Ratio` to
the relevant range. The values weren't arbitrary, but they hadn't been chosen
with systematic listening.

| Ratio   | Knee     | Threshold        |
| :-----: | :------: | :--------------: |
|   1:1   |  7.0 dB   |     -20.0 dB    |
|   ...   |  ...      |     ...         |
|   1000:1   |  0.0 dB   |     -10.0 dB |


In implementing our new ratio parameter, I opted to expand the range all the
way down to `1:1`, making Valentine just a distortion. I chose starting values
for knee and threshold that would result in values matching the previous settings
at the previous starting ratio (`4:1`) as a starting point. I made further adjustments
to the knee and threshold ranges, listening for how the compression changed over
the ratio parameter's range.

Maybe Overthink It A Little Bit
===============================

I had gotten all of this done with not so much pain, and implementing a partial
solution got me on track to an implementation I was a lot happier with. For me, using a continuous control
didn't make Valentine more complicated, as I had feared. It seemed the opposite,
since I could now concentrate on listening to how changing `Ratio` affected the
sound with one control gesture. One can do this with eyes closed. It sounded good, 
but the output got a bit louder as ratio increased in a way that I didn't love. Overall
I felt the change was a net benefit.

I thought that there had to be a straightforward way to at least ensure
that, for the settings that were previously available, Valentine would sound the same.
I realized that I was already remapping the full ratio range to the full knee and threshold ranges
and could easily ensure that our knee and threshold were the right value at key
ratios by treating the distance between these points as smaller, connected ranges to remap to.
For example, we can find the knee for a ratio of 4.5:1 by remapping from
`4.0` - `8.0` to `6.0` - `3.84`. I could try this with linear interpolation, upgrading to something
more involved if necessary.

```cpp
/** Remaps a value from one range of the other, using sections corresponding to
 *  the upper and lower bounds of the input value as found in the input range.
 */
template <typename ValueType, size_t ArraySize>
ValueType piecewiseRemap (const std::array<ValueType, ArraySize> inputSegments,
                          const std::array<ValueType, ArraySize> outputSegments,
                          const ValueType input)
{
    const auto [lowerBound, upperBound] = findNearestIndices (inputSegments, input);
    // clang-format off
    const auto normalizedInput = (input - inputSegments[lowerBound])
                               / (inputSegments[upperBound] - inputSegments[lowerBound]);
    // clang-format on

    return outputSegments[lowerBound]
           + (outputSegments[upperBound] - outputSegments[lowerBound]) * normalizedInput;
}
```

This resulted in a sound that I was much happier with, especially when adjusting
the parameter. It looks better too.

<p align="center">
  <img src="/assets/images/valentine_screenshot_knob.png" alt="A screenshot of Valentine's UI. The ratio box has been replaced with a knob" />
</p>


Here's the [pr](https://github.com/tote-bag-labs/valentine/pull/58).
