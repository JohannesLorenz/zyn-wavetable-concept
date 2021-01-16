# Wavetable communication

This explains how RT and non-RT threads communicate when new waves for the
wavetable need to be generated.

## Wavetable layout

A wavetable has the following contents:

```
  ooooooo 1D-Tensor (frequeny values in Hz)      --
                                                  |
  ooooooo 1D-Tensor (semantic values, e.g.        | "wavetable scales"
                     random seeds or              |
                     basewave function params)   --

  ooooooo 1D-Tensor (float buffer = 1 wave)      --
  ^                                               |
  |                                               | "wavetable data"
  o...ooo 2D Tensor (Tensor1s for each freq)      | or just
  ^                                               | "waves"
  |                                               |
  o...ooo 3D Tensor (Tensor2s for each semantic)  |
          with ringbuffer semantics               |
          (MW produces, AD note consumes)        --
```

## Sequence diagram

The following (not necessarily UML standard) diagram explains
how and when new wavetables are being generated. There are two
entry points, marked with an "X".

```
    MW (non-RT)                                                 ADnote (RT)
    |                                                           |
    |   wavetable-relevant data changed                         |
    |<---X                                                      |
    |                                                           |
    - calculate new scale 1D tensors                            |
      (if required)                                             |
    |                                                           |
    |--/path/to/ad/voice/wavetable-params-changed:T:F:Tbb:Fbb-->|
    |      Inform ADnote that data relevant for wavetable       |
    |      creation has changed                                 |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - optional 1D scale Tensors (bb)                     |
    |                                                           |
    - suppress further "wavetable-params-changed"               |
    |                                                           |
    |                          - swap scales if transmitted     |
    |                          - resize wavetable if scales     |
    |                            have been transmitted          |
    |<-----free:sb----------------------------------------------|
    |      Free old frequencies                                 |
    |<-----free:sb----------------------------------------------|
    |      Free old semantics                                   |
    |                                                           |
    - free them                                                 |
    |                            Master periodically checks if  |
    |                            ADnote still has enough        |
    |                            wavetables (ringbuffer read    |
    |                            space)                    X--->|
    |                                                           |
    |<-----request-wavetable:sTiibbi:sFiibbi--------------------|
    |      Inform MW that new waves can be generated            | 
    |      - path of ADvoice (sT/sF)                            |
    |      - wavetable ringbuffer                               |
    |        write position + space (ii)                        |
    |        (write position = semantic index)                  |
    |        (space = how many new Tensor2's are needed)        |
    |      - 1D scale tensor ptrs                               |
    |        (semantics, freqs) (bb)                            |
    |      - Presonance boolean (i)                             |
    |                                                           |
    - stop suppressing further "wavetable-params-changed"       |
    - store wavetable request in queue                          |
    - handle wavetable requests in next cycle                   |
      to generate waves                                         |
    |                                                           |
    |------/path/to/ad/voice/set-waves:Tib:Fib----------------->|
    |      pass new waves to ADnote                             |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - write position (=semantic index) (i)               |
    |      - 2D Tensor for given semantic (b)                   |
    |                                                           |
    |                          - swap all Tensor1s from the     |
    |                            passed Tensor2 with the        |
    |                            Tensor1s that have been        |
    |                            consumed previously            |
    |                          - increase ringbuffer write pos  |
    |                            by 1 (since waves for one      |
    |                            semantic have been inserted)   |
    |                                                           |
    |<-----free:sb----------------------------------------------|
    |      recycle the Tensor2 which includes the now           |
    |      unused (because of swap) Tensor1s                    |
    |                                                           |
    - free the Tensor2 (includes freeing                        |
      contained Tensor1s)                                       |
```