# Separate predictor errors from sampler errors

Test in stages. First run the sampler with exact targets on a known distribution, like this page's mixture. Next test the trained predictor at individual noise levels. Only then judge generated samples. A correct sampler can't make up for a wrong time index, wrong data normalization, or a prediction in the wrong convention.

This page's widgets are split the same way. The posterior widget is exact Bayes arithmetic, the line trainer fits one fixed dataset, and the sampler runs with a known exact predictor. Testing the parts separately catches errors that a good-looking final animation can hide.
A good-looking sample proves little

A sample can look plausible and still fail to match a specific original, ignore the requested condition, or cover only part of the data. Measure the property you need; the denoising formula guarantees none of these.
