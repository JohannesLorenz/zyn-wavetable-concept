# Advantages/Disadvantages of the tensor implementation

The advantages/disadvantages with same numbers belong together.

## Disadvantages:

1. Higher computation times in non-RT thread:
  1. Precomputing waves that will never be used (e.g. for unused frequencies)
2. Code gets a bit more complicated (more communication for wavetable pointers, wavetable parameters, pasting etc. between RT and non-RT threads)
3. Frequencies may not match exactly the played frequency (though probably without hearable difference for Nyquist-Shannon. It works in PAD synth, too)
4. Some parameters can not be automated in realtime anymore (previously, you could change the base function for every new ADnote in realtime)
5. -

## Advantages:

1. Lower computation times in RT thread:
  1. If no random (default settings), waves can be re-used
2. The tensor implementation makes it easier to implement wavetable modulation (e.g. by modulating the basewave parameter)
  1. OscilGen code can be non RT
3. -
4. -
5. Faster reaction time: FFT/IFFT are done way before noteOn
  1. Especially, playing multiple notes at once is no problem anymore (imagine DAWs and the begin of a bar)
