Saturation gain normalization [is a lot better now](https://github.com/tote-bag-labs/valentine/pull/50)


As described in the attached issue, turning Saturation up worked as intended,
but also made the output quieter. This is because of the way gain was being
normalized. This is because of the overly simplistic normalization used in saturation
processing.

Here's how we were doing it:

y = f(x * gain) / gain

This works in that it keeps the signal from getting too loud and hence requiring manual
gain adjustment when turning up Saturation. This doesn't actually work because the
the function is, of course, nonlinear. The function's output won't increase as much as
gain does. Dividing the output of this function by gain, then, will make the output
quieter.

What we need is for the gain compensation to ease up for larger gain inputs.

The following works a lot better:

y = f(x * gain) / f(gain)

I probably saw this in a paper or book before and forgot to apply it.
And then, when I was pondering this issue, I asked copilot to write me a function
that would find the correct compensation gain for an inverse hyperbolic sine.

The answer was basically what's shown above.

There's also some formatting changes in here, mostly to make the code more readable.
