Reloop Mixtour Pro
==================

-  `Manufacturer's product page <https://www.reloop.com/reloop-mixtour-pro>`__
-  `Forum thread <TODO-add-mixxx-forum-thread-link>`__

.. TODO: set the version this mapping is first shipped in
.. versionadded:: 2.6

Compatibility
-------------

The Reloop Mixtour Pro is a class-compliant USB MIDI and audio device, so it works
without any special drivers on GNU/Linux, macOS and Windows.

.. note::
   The mapping switches the controller into its advanced ("Serato") mode on startup
   by sending two SysEx messages from ``init()``. This is required for the performance
   pads' mode picker and the mode/shift button combinations to work. No other software
   needs to have run first.

Sound card setup
----------------

The controller has a built-in audio interface with a main output and a headphone
output. Assign them under **Preferences > Sound Hardware** (Main and Headphones
outputs). Verify the channel numbers for your unit.

Mapping description
-------------------

The controller is a 2-channel surface that controls **four decks**: hold a
:hwlabel:`MODE` button and press the corresponding Track Load button to flip the left
side between decks 1 and 3, or the right side between decks 2 and 4. Button and pad
LEDs are blue for decks 1/2 and red for decks 3/4.

Decks
~~~~~

=========================================================  ================================================================
Control                                                    Function
=========================================================  ================================================================
Play button                                                Play / pause the deck.
:hwlabel:`SHIFT` + Play                                    Temporary pitch bend up.
:hwlabel:`MODE` + Play                                     Raise the musical key.
:hwlabel:`SHIFT` + :hwlabel:`MODE` + Play                  Nudge tempo up by 0.1 BPM.
:hwlabel:`CUE` button                                      Set / jump to the temporary cue point.
:hwlabel:`SHIFT` + :hwlabel:`CUE`                          Temporary pitch bend down.
:hwlabel:`MODE` + :hwlabel:`CUE`                           Lower the musical key.
:hwlabel:`SHIFT` + :hwlabel:`MODE` + :hwlabel:`CUE`        Nudge tempo down by 0.1 BPM.
:hwlabel:`SYNC` button                                     Toggle :ref:`beat sync <sync-lock>`.
:hwlabel:`SHIFT` + :hwlabel:`SYNC`                         Match the musical key to the other deck.
:hwlabel:`MODE` + :hwlabel:`SYNC`                          Double the current loop size.
:hwlabel:`SHIFT` + :hwlabel:`MODE` + :hwlabel:`SYNC`       Set / move the loop out point.
:hwlabel:`IN-OUT-EX` button                                1st press: loop in. 2nd: loop out (loop active). 3rd: exit loop.
:hwlabel:`SHIFT` + :hwlabel:`IN-OUT-EX`                    Toggle a beat loop.
:hwlabel:`MODE` + :hwlabel:`IN-OUT-EX`                     Halve the current loop size.
:hwlabel:`SHIFT` + :hwlabel:`MODE` + :hwlabel:`IN-OUT-EX`  Set / move the loop in point.
=========================================================  ================================================================

Mixer
~~~~~

===============================  ==================================================================
Control                          Function
===============================  ==================================================================
Line faders                      Channel volume.
:hwlabel:`CROSSFADER`            Crossfade between the left and right channels.
3-band EQ knobs                  High / mid / low for the channel.
:hwlabel:`FILTER` knob           Filter (QuickEffect): low-pass anticlockwise, high-pass clockwise.
Channel gain                     Channel gain (trim).
Master volume                    Master output volume.
Headphone volume                 Headphone (cue) volume.
Headphone mix                    Headphone cue / master balance.
:hwlabel:`HEADPHONE CUE` button  Toggle headphone cue (PFL) for the channel.
===============================  ==================================================================

Browser
~~~~~~~

============================================  =========================================================
Control                                       Function
============================================  =========================================================
:hwlabel:`BROWSE` encoder (turn)              Scroll up / down the library.
:hwlabel:`BROWSE` encoder (press)             Load / open the selected item.
:hwlabel:`SHIFT` + :hwlabel:`BROWSE` (press)  Go back one step.
:hwlabel:`MODE` + :hwlabel:`BROWSE` (turn)    Beat-snapped seek through the track (4 beats per detent).
Track Load button                             Load the selected track to the deck.
:hwlabel:`SHIFT` + Track Load (left / right)  Move focus left / right between library panes.
:hwlabel:`MODE` + Track Load button           Switch the side between decks 1/3 (left) or 2/4 (right).
============================================  =========================================================

FX
~~

===================================  ============================================================================
Control                              Function
===================================  ============================================================================
FX paddle                            Engage Effect Unit 1 on the deck (push down = momentary, lock up = latched).
:hwlabel:`SHIFT` + FX paddle         Backspin the deck.
:hwlabel:`DRY/WET` knob              Effect Unit 1 dry/wet mix (affects both decks).
FX parameter button (left / right)   Adjust the focused effect's parameter down / up.
:hwlabel:`SHIFT` + parameter button  Select the previous / next effect.
===================================  ============================================================================

Performance pads
~~~~~~~~~~~~~~~~

Hold a :hwlabel:`MODE` button and press the pad printed with the desired mode to
select it for that deck. Press :hwlabel:`SPLIT` to control both decks at once (the
four left pads control the left deck, the four right pads the right deck); otherwise
all eight pads control the active deck.

Hot Cue Mode
^^^^^^^^^^^^

============================  ==================================================================
Control                       Function
============================  ==================================================================
Pad 1 - 8                     Set and trigger :term:`hotcue` 1 - 8 (lit in the hotcue's colour).
:hwlabel:`SHIFT` + Pad 1 - 8  Clear :term:`hotcue` 1 - 8.
============================  ==================================================================

Auto Loop Mode
^^^^^^^^^^^^^^

=========  =========================================================
Control    Function
=========  =========================================================
Pad 1 - 8  Toggle a beat loop of 1/4, 1/2, 1, 2, 4, 8, 16, 32 beats.
=========  =========================================================

Bounce Loop Mode
^^^^^^^^^^^^^^^^

==================  ========================================
Control             Function
==================  ========================================
Left pads (1 - 4)   Beatjump backward by 4, 8, 16, 32 beats.
Right pads (5 - 8)  Beatjump forward by 4, 8, 16, 32 beats.
==================  ========================================

Sampler Mode
^^^^^^^^^^^^

======================  ===========================================
Control                 Function
======================  ===========================================
Pad 1 - 8               Load (if empty) or play sampler 1 - 8.
:hwlabel:`SHIFT` + Pad  Stop a playing sample; eject a stopped one.
======================  ===========================================

Instant FX Mode
^^^^^^^^^^^^^^^

================  =========================================================================
Control           Function
================  =========================================================================
Pad 1 - 8 (hold)  Apply Quick Effect chain preset 2 - 9 while held; releases to the filter.
================  =========================================================================

Configure the eight effects under **Preferences > Effects > Quick Effect chain
presets**: preset 1 is the deck's default filter, presets 2 - 9 are the Instant FX
pads in order.

Pitch Cue Mode
^^^^^^^^^^^^^^

============================  =================================================
Control                       Function
============================  =================================================
Pad 1                         Reset to the original key.
Pad 2 - 8                     Set the key to +1 .. +7 semitones.
:hwlabel:`SHIFT` + Pad 1 - 8  Jump to :term:`hotcue` 1 - 8 (switch cue points).
============================  =================================================

Saved Loops Mode
^^^^^^^^^^^^^^^^

============================  ==========================================
Control                       Function
============================  ==========================================
Pad (empty slot)              Save the current loop to the slot.
Pad (saved slot)              Recall: jump to and enable the saved loop.
:hwlabel:`SHIFT` + Pad 1 - 8  Clear the saved loop.
============================  ==========================================

Controls not included in this mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  **Neural Mix buttons** (no Mixxx equivalent).
-  :hwlabel:`SHIFT` + :hwlabel:`HEADPHONE CUE` (Crossfader FX, no Mixxx equivalent).
-  **Neural Mix** pad mode (not yet implemented).

