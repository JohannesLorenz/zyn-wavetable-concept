# Problems

Problems with changing the modulation type (between WT and any non-WT) while a note is running:

* Disabling WT/non-WT modulation on a running note which was started with WT/non-WT modulation
  - requires to recompute the new table (possibly with the random seeds/with the wavetable)
  - requires to somehow keep the old table still in memory (complicated reference counts?)
* Enabling WT/non-WT modulation on a running note requires the note to still wait for MW to generate the note (=> complicated code)
* Enabling/Disabling WT modulation on a running note can lead to jumps/clips (Like with every other modulation, but for other modulations, we must stay backward compatible: Changing between them shall change how the ADnote is being played)

=> We try to solve all this by disabling transitions (between WT and non-WT) and let them lead to a quick fade out instead.

Problems with changing the modulation type "while" the note is started:

* At the time the note is started, it's possible that the UI already requested to change the modulation type, but the table has not been generated yet. This means the modulation type requests a modulation for which no table exists yet.

=> Such notes are still being played with the old modulation type. This means the old table must resist in ADnoteParameters until the new one is "atomically" added. These notes will be faded with the upcoming table transition (see above).

# Additions to ADnote

* A bool if the initial modulation type was WT or not
* Two backup buffers for the currently used random-seed/wavetable buffers (explained later)
* The backup LERP coefficient for the currently used WT modulation (explained later)

# The effective modulation type

* If none or both "initial" and "current" modulation types are WT: The current modulation type (which we read every time from ADnoteParameters)
* Else: the initial

=> This makes sure that "the effective modulation is WT <=> the initial modulation is WT" (i.e. WT-ness is kept during lifetime of every note), but still allows switching between FM, PM, RM ...

# When the modulation type changes while a note is alive

* If current and initial modulation differ by WT-ness (i.e. one is WT, the other is not WT):
  - Initiate a fade out (256 samples at 48 kHz), and
    * If the current modulation is WT (i.e. the initial is not WT), backup the currently used buffer (possibly for the current random seed)
    * Else, if the current modulation is not WT (i.e. the initial is WT), backup the currently used two buffers and the LERP factor

# If fadeout is active

* If the initial modulation is WT, keep playing the backup buffers with the backup LERP coefficient (since the table might be gone already)
* Otherwise, keep playing the current backup buffer

