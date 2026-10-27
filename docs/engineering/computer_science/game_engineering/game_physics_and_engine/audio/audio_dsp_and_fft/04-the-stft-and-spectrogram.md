# The STFT and Spectrogram

The Short-Time Fourier Transform slides a windowed FFT across a signal. Each transform becomes a column in a time × frequency map; displaying magnitude in dB produces a spectrogram. The hop size controls how far the window advances. Hops of 25–50% of the window are common so tapered windows overlap.

Time and frequency resolution trade off. A short window tracks transients accurately but has coarse frequency bins; a long window resolves nearby frequencies but smears events in time. No implementation can maximize both because of the Fourier uncertainty relation. More overlap smooths the display but does not improve frequency resolution.

At 48 kHz, a 1024-sample window spans 21.3 ms and resolves 46.9 Hz. A 256-sample window spans 5.3 ms but resolves only 187.5 Hz. With a 50% hop, the 1024-sample analysis advances 512 samples and produces 93.75 columns per second.
