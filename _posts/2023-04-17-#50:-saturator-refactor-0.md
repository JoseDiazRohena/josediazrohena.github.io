Saturation gain normalization [is a lot better now](https://github.com/tote-bag-labs/valentine/pull/50)

As described in the attached issue, turning the saturation parameter up worked as intended, but also made the output quieter. This is because of the way the saturation output was being adjusted to account for input gain. This is because of the overly simplistic normalization used in saturation processing.

Here's how we were doing it:

`y = f(x * gain) / gain`

This works in that it keeps the signal from getting too loud without manual gain adjustment. This doesn't actually work because the function we are doing gain compensation for is, of course, nonlinear. The function's output won't increase as much as the input gain does. Dividing the output of this function by gain, then, will make the output quieter for large gain inputs. Not good for encouraging the user to turn up the gain!

What we need is for our gain compensation to ease up for larger gain inputs.

The following works a lot better, resulting in a small amount of output gain
increase:

`y = f(x * gain) / f(gain)`

I probably saw this in a paper or book before and forgot to apply it. And then, when I was pondering this issue, I asked copilot to write me a function that would find the correct compensation gain for an inverse hyperbolic sine. The answer was basically what's shown above.

Implementation Details
======================
This post isn't named "Getting Gain Normalization Right" or something like that because, as pointed out above, the solution wasn't terribly interesting. Implementing it raised some interesting questions about how to correctly implement the fix.

As shown above, we find the correct compensation by running the saturation function for the gain we are applying to the signal. In our case, this has to be done on a per sample basis because the gain actually passed into whatever saturator being used depends on the given input. We may want asymmetry. And so we run into a problem already present in this implementation:

```cpp
switch (saturationType)
    {
        case Type::inverseHyperbolicSine:
            return inverseHyperbolicSine (inputSample * gain) * compensationGain<inverseHyperbolicSineTag> (gain);

        case Type::sineArcTangent:
            return sineArcTangent (inputSample, gain);
/../
```

When I first implemented this saturation class, I did so naively. This switch executes in the saturation class's `processSample` function. This can be bad for [cache performance](https://blog.cloudflare.com/branch-predictor/). I think I'm going to resolve this by refactoring `Saturation` into a templated class. But that's a big project for now. So let's at least not add even more branches (at runtime) for figuring out which gain compensation function to call.

```cpp
  struct inverseHyperbolicSineTag {};
  struct hyperbolicTangentTag {};

  template <typename SaturationType, typename FloatType>
  auto compensationGain(FloatType inputGain)
  {
    if constexpr (std::is_same<SaturationType, inverseHyperbolicSineTag>::value)
    {
      return static_cast<FloatType>(1.0) / inverseHyperbolicSine(inputGain);
    }
    else if constexpr (std::is_same<SaturationType, hyperbolicTangentTag>::value)
    {
      return static_cast<FloatType>(1.0) / hyperbolicTangent(inputGain);
    }
    else
    {
      static_assert(tote_bag::type_helpers::dependent_false<SaturationType>::value, "Unsupported saturation type.");
    }
  }
```

This might not be the best approach for a full refactor: maybe some kind of templated saturation base class (inheriting from a processing class as part of a big [CRTP refactor?](https://en.cppreference.com/w/cpp/language/crtp)) would be easier to read and faster to compile. But for now, I think this is Good Enough.

Benchmarking
============
This is all driven by performance concerns. But they are entirely hypothetical. One thing I'm interested in doing is benchmarking this code before the refactor.
