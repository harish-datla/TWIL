# Digital Filters

A digital filter shapes frequencies by computing each output sample from recent inputs and outputs:

```text
y[n] = b0x[n] + b1x[n−1] + … − a1y[n−1] − a2y[n−2] − …
```

FIR filters have no feedback, are stable, and can have exactly linear phase, but a sharp response can require many taps. IIR filters use feedback and achieve a response cheaply, but can be unstable and have nonlinear phase. A one-pole low-pass such as `y[n] = 0.1x[n] + 0.9y[n−1]` lets slow changes through while suppressing fast ones.

## RBJ biquad

A biquad is a second-order IIR section with two poles and two zeros. The RBJ Audio EQ Cookbook supplies coefficients for low/high/band-pass, notch, peak, and shelf filters. Cascade biquads for higher-order filters instead of using one high-order section; coefficient quantization is safer. Keep feedback states away from denormals with FTZ/DAZ or equivalent handling.

```cpp
struct Biquad {
    float b0,b1,b2,a1,a2, x1=0,x2=0,y1=0,y2=0;
    void setLowpass(float cutoff, float q, float rate) {
        float w = 2.0f * 3.14159265f * cutoff / rate;
        float c = std::cos(w), alpha = std::sin(w) / (2.0f*q);
        float a0 = 1.0f + alpha;
        b0 = ((1-c)*0.5f)/a0; b1 = (1-c)/a0; b2 = b0;
        a1 = (-2*c)/a0; a2 = (1-alpha)/a0;
    }
    void process(float* samples, int count) {
        for (int i = 0; i < count; ++i) {
            float x0 = samples[i];
            float y0 = b0*x0 + b1*x1 + b2*x2 - a1*y1 - a2*y2;
            x2=x1; x1=x0; y2=y1; y1=y0; samples[i]=y0;
        }
    }
};
```

At 48 kHz, a 1 kHz low-pass with Q=0.707 has approximately `b0=b2=0.00392`, `b1=0.00783`, `a1=-1.8153`, and `a2=0.8310`. A positive Q is stable in exact arithmetic; high-Q, low-frequency sections are more sensitive to rounding.
