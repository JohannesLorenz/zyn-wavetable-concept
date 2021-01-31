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
          with ringbuffer semantics (see below)   |
          (MW produces, AD note consumes)        --
```

Ringbuffer layout:

```
  required space holder (to make r=w unambiguous)                      wrap
        ||                                                               |
        vv               |--reserved for write--|                        v
  ...---||--can be read--|                      |--can be reserved next...
  ...oooooooo............oooo...................oooo......................
         |               |                      |
         r               w_delayed              w              <- pointers
```

## Sequence diagram

The following (not necessarily UML standard) diagram explains
how and when new wavetables are being generated. There are two
entry points, marked with an "X".

```
    MW (non-RT)                                                 ADnote (RT)
    |                                                           |
    |   wavetable-relevant params changed                       |
    |<---X                                                      |
    |                                                           |
    |  If "wavetable-params-changed" is not suppressed:         |
    |--/path/to/advoice/wavetable-params-changed:T:F:Tibb:Fibb->|
    |      Inform ADnote that data relevant for wavetable       |
    |      creation has changed                                 |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - if changed params affect scales:                   |
    |        unique timestamp of parameter change (i)           |
    |        calculate and send 1D scale Tensors (bb)           |
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
    |                          - increase ringbuffer's reserved |
    |                            write space (by increading `w`)|
    |                                                           |
    |<request-wavetable:sTiiibbi:sFiiibbi:iiiTiiibbi:iiiFiiibbi-|
    |      Inform MW that new waves can be generated            | 
    |      - path of OscilGen (s or iii is voice path, T/F is   |
    |        OscilGen path)                                     |
    |      - if 1D scale Tensors were passed:                   |
    |          parameter change timestamp                       |
    |        else                                               |
    |          0 (parameter change timestamp is implicitly the  |
    |             one of the latest parameter change which      |
    |             ADnote observed)                              |
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
    |      If MW has not observed any parameter change after    |
    |      the parameter change time of this request:           |
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
    |                          - increase ringbuffer's write    |
    |                            pointer `w_delayed` by 1       |
    |                            (since reserved waves for one  |
    |                            semantic have now been         |
    |                            inserted)                      |  
    |                                                           |
    |<-----free:sb----------------------------------------------|
    |      recycle the Tensor2 which includes the now           |
    |      unused (because of swap) Tensor1s                    |
    |                                                           |
    - free the Tensor2 (includes freeing                        |
      contained Tensor1s)                                       |
```
