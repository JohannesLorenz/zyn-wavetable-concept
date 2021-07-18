# Problems

* Disabling WT modulation on a running note which was started with WT modulation
  - requires to recompute OscilGen::get
  - requires to somehow keep the table still in memory (complicated reference counts?)
* Enablind WT modulation on a running note requires the note to still wait for MW to generate the note (=> complicated code)
* Enabling/Disabling WT modulation on a running note can lead to jumps/clips (Like with every other modulation, but for other modulations, we must stay backward compatible: Changing between them shall change how the ADnote is being played)

=> We try to solve all this by disabling transitions (between WT and non-WT) and let them lead to a quick fade out instead.

# Additions to ADnote

* A bool if the initial modulation type was WT or not
* Two backup buffers for the currently used wavetable buffers (explained later)
* The backup LERP coefficient for the currently used WT modulation (explained later)

# The effective modulation type

* If none or both "initial" and "current" modulation types are WT: The current modulation type (which we read every time from ADnoteParameters)
* Else: the initial

=> This makes sure that "the effective modulation is WT <=> the initial modulation is WT" (i.e. WT-ness is kept during lifetime of every note), but still allows switching between FM, PM, RM ...

# When the modulation type changes while a note is alive

* If current and initial modulation differ by WT-ness (i.e. one is WT, the other is not WT):
  - Initiate a fade out (256 samples at 48 kHz), and
    * If the current modulation is WT (i.e. the initial is not WT), don't do anything: simply keep using the buffer from OscilGen::get (from the CTOR).
    * Else, if the current modulation is not WT (i.e. the initial is WT), backup the currently used two buffers and the LERP factor

# If fadeout is active

* If the initial modulation is WT, keep playing the backup buffers with the backup LERP coefficient (since the table might be gone already)
* Otherwise, just a normal fadeout

