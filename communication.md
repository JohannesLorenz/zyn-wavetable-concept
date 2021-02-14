# Wavetable communication

This explains how RT and non-RT threads communicate when new waves for the
wavetable need to be generated.

## Wavetable layout

A Tensor3 (3D-Tensor) has the following contents:

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

Wavetable layout:

```
 o "current" ringbuffer (Tensor3)
 o "next" ringbuffer (Tensor3, to be filled by parameter change)
 o timestamp of params for current ringbuffer
 o timestamp of params for next ("to be filled") ringbuffer
 
 o scales for requested (and most of the time for current) ringbuffer
 
 o wavetable mode (e.g. random seeds, basefunc parameter, ...)
```

## Asynchonicity principles

MiddleWare and ADnoteParams are run by 2 different threads at the same
time. This can cause issues, like outdated requests. To avoid this, the
following principles have been set up:

* wavetable requests induced by parameter changes always carry a timestamp
* any side should not rely on the other side handling the asynchonicity
  "well", i.e. it can not be relied on the other side detectig messages
  with outdated timestamps

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
    |--/path/to/advoice/wavetable-params-changed--------------->|
    |  :Ti:Fi:Tibb:Fibb                                         |
    |      Inform ADnote that data relevant for wavetable       |
    |      creation has changed                                 |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - unique timestamp of parameter change (i)           |
    |      - if changed params affect scales:                   |
    |        calculate and send 1D scale Tensors (bb)           |
    |                                                           |
    - suppress further "wavetable-params-changed"               |
    |                                                           |
    |                          - swap scales if transmitted     |
    |                          - mark current ringbuffer        |
    |                            "outdated until                |
    |                            waves for timestamp arrive"    |
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
    | If the wavetable request comes from a parameter change,   |
    | or the current wavetable is not outdated                  |
    |<request-wavetable:sTiiibbi:sFiiibbi:iiiTiiibbi:iiiFiiibbi-|
    |      Inform MW that new waves can be generated            | 
    |      - path of OscilGen (s or iii is voice path, T/F is   |
    |        OscilGen path)                                     |
    |      - In case of parameter change:                       |
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
    |      the parameter change time of this request, and the   |
    |      scale sizes have changed:                            |
    |------/path/to/ad/voice/set-tensor3:Tb:Fb----------------->|
    |      pass new container to ADnoteParams                   |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - 3D Tensor for given semantic (b)                   |
    |                                                           |
    |                          - swap passed Tensor3 with the   |
    |                            wavetable's "next" buffer      |
    |                                                           |
    |<-----free:sb----------------------------------------------|
    |      recycle the Tensor3 which includes the now           |
    |      unused (because of swap) sub-Tensors                 |
    |                                                           |
    - free the Tensor3 (includes freeing                        |
      contained sub-Tensors)                                    |
    |                                                           |
    |      If MW has not observed any parameter change after    |
    |      the parameter change time of this request:           |
    |------/path/to/ad/voice/set-waves:Tiib:Fiib--------------->|
    |      pass new waves to ADnote                             |
    |      - path: osc or mod-osc? (T/F)                        |
    |      - timestamp of inducing parameter change (0 if none) |
    |      - write position (=semantic index) (i)               |
    |      - 2D Tensor for given semantic (b)                   |
    |                                                           |
    |                If the wave does not come from an outdated |
    |                parameter change (OK if from up-to-date    |
    |                parameter change or not from parameter     |
    |                change at all):                            |
    |                          - swap all Tensor1s from the     |
    |                            passed Tensor2 with the        |
    |                            Tensor1s from the "next"       |
    |                            ringbuffer (parameter change)  |
    |                            or with the previously         |
    |                            consumed Tensor1s              |
    |                          - increase respective ringbuffer |
    |                            write pointer `w_delayed` by 1 |
    |                            (since reserved waves for one  |
    |                            semantic have now been         |
    |                            inserted)                      | 
    |                          - in case of parameter change,   |
    |                            if the last semantic has       |
    |                            arrived: swap WT's "next" and  |
    |                            "current" ringbuffers          |
    |                                                           |
    |<-----free:sb----------------------------------------------|
    |      recycle the Tensor2 which includes the now           |
    |      unused (because of swap) Tensor1s                    |
    |                                                           |
    - free the Tensor2 (includes freeing                        |
      contained Tensor1s)                                       |
```
