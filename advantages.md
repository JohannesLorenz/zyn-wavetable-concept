# Advantages/disadvantages of the wavetable implementation

This discusses the advantages/disadvantages of using non-RT generated wavetables in ADnote, rather than computing them live in noteOn. While this is a pre-step before implementing wavetable modulation, this page is not (much) about wavetable modulation, but rather tries to answer if the changes to the current ADnote code to use wavetables are justified.

The advantages/disadvantages with same numbers belong together.

## Disadvantages:

1. Higher computation times in non-RT thread:
   - Precomputing waves that will never be used (e.g. for unused frequencies)
2. Code gets a bit more complicated (more communication for wavetable pointers, wavetable parameters, pasting etc. between RT and non-RT threads)
3. Frequencies may not match exactly the played frequency (though probably without hearable difference for Nyquist-Shannon. It works in PAD synth, too)
4. Additional latency to some parameter automation, though automations are still
   possible. e.g. changing the waveform base function may not be reflected on a
   note immediately after the change, but notes a short period of time after the
   change should have the updated waveform shape
5. Introducing random elements is more challenging

## Advantages:

1. Lower computation times in RT thread:
   - If no random (default settings), waves can be re-used
2. The wavetable implementation makes it easier to implement wavetable modulation (e.g. by modulating the basewave parameter)
   - OscilGen code can be non RT
3. (no counterpart)
4. Faster reaction time: FFT/IFFT are done way before `AdNote::noteOn`
   - Especially, playing multiple notes at once is no problem anymore (imagine DAWs and the begin of a bar)
5. Allows for the possibility of selecting multiple anti-aliased wavetables with
   slow, but high dynamic range (in terms of frequency) modulation
6. Avoids non-realtime safe data preparation within `AdNote::noteOn`
