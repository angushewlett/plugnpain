# plugnpain
Questions you should ask of any proposed audio plug-in API

An incomplete list of things that have proven to be painful during the design, development and use of native audio plug-in APIs, 1997-2023.

Compiled by Angus Hewlett angushewlett@gmail.com with comment, collaboration, contribution and plagiarism from others.

Document is open to revision, feel free to send PRs.

These questions and pain points are not intended as criticism of any specific API or set of design choices, but if you're designing or evaluating such an API, they are things which should be considered. Even if the considered answer is, "*shrug*, that's not in scope."

1. What is a plug-in, and how should the relationship between host, API and plug-in be framed at the level of design philosophy? Are your plug-ins subordinate to the host application, with the API/SDK enforcing this, or are they peers whose communication is mediated via your framework? Something more like the Eurorack hardware metaphor, or just 0.25in jacks to wire everything up?

2. How do you handle metadata, fast scanning, multi-shells.. can a single system package (component, bundle, DLL etc.) serve up more than one plug-in object? Is that list of things static (within a given version release / binary checksum / last-modified date) or dynamic? Are different connection topology variants of the same device (mono, stereo, surround) different objects, or different configurations of the same object?

3. How does your API cleanly handle error conditions that might prevent packages loading or instantiating objects, and capture and communicate those states in a way which is non-blocking and compatible with headless / command-line operation? Do you have guidelines to ensure that non-compliant communication of such error conditions (modal licensor dialogs during scan) constitutes a hard FAIL?

4. How do plug-in and host negotiate I/O semantics, both at instantiation and during runtime? Not just the number and type of I/O "pins", but the meaning of those pins, and valid (input, output) pairings/combinations/constrints. (Or: do you intentionally handball this to users, as audio jacks and XLR connectors did for many a year.)

5. What are your editor (GUI) lifetime semantics relative to your main object; what are the allowed (m <-> n) configurations of plug-in and editor; to what extent are process separation and thread separation enforced? Is all transfer of state between audio object and GUI object mediated via the host, and how are potentially larger and more complex chunks of state shared (sample library information, DRM/licensing state etc..)

6. How does API manage screen resolution negotiation, DPI management & realtime resizing?

7. How does the API differentiate save-state chunks vs presets or programs - these are similar but not the same thing (often a plug-in might save extra information in a “host save” compared to a preset). Are MIDI program changes supported (realtime or deferred), is there a concept of edit buffers, what rules are applied to visible state consistency for state-change operations applied in a non-realtime and realtime thread?

8. How is preset enumeration defined? Are banks and indices a thing, who is responsible for maintaining preset databases, is there some sharing of "container" file format and metadata?

9. What is the approach to memory management for variable-sized plug-in output data, particularly the Save Chunk and realtime MIDI output?

10. When to MIDI-bind parameters and when not to; how to clearly distinguish volatile “performance” data from persistent “parameter change” data. (For example, a plug-in is not expected to save pitch bend wheel state in a preset. Should mod-wheel performance data, or poly aftertouch, or some MPE gesture, be considered to “dirty” the state of a project/preset, and to flag the undo bit?). Clear responsibility between plug-in and host for controller binding, but in a way that doesn’t interfere with MIDI (or MPE) controller data that is being used for performance rather than parameter editing. What are the rules for state-change visibility when these messages may arrive on different threads?

11. Does your API precisely specify threading rules, in a way that makes it no harder than necessary to write correct lock-free code, or at least to allow the realtime part of plug-in audio engines to be implemented in a lockfree or near-lockfree manner?

12. Does your API have a concept of (static or dynamic sized) multi-timbrality for plug-ins? Is the plug-in object a single logical target/component with linear indexing of parameters, or is there some kind of addressing hierarchy (homogenous or heterogenous), per the concept of elements/scopes in Audio Units?

13. How is buffer sizing behaviour, timing of buffer delivery etc. specified? (note USB audio cards and their habit of pulling unevenly sized buffers). Fixed size, variable size, powers of two, isochrony...

14. What is the spec for buffer sizing and clock behaviour during preroll, at loop start and end points, audio tail handling (after end of timeline, and after playback stop).

15. When should plug-ins flush different bits of the audio engine (think soft flush for looping vs MIDI All Notes Off vs hard flush with buffer reallocation)

16. How is cooperative multi-threading supported at audio thread level, how do plug-ins and hosts to negotiate on core allocation, pinning etc. to avoid situations where plug-ins are loading the CPU heavily and not reporting it, fighting one another for cores etc., note bigcores vs littlecores. Can the API ensure that all threads are properly accounted for (to avoid multithreaded hosts and multithreaded plug-ins contending for the same secondary core)

17. Can non-atomic MIDI messages (NRPNs, SysEx streams, MPE note-ons) be packaged as an atomic event?

18. Is there a system for preloading / caching programs/presets, so that large state-changes can be applied without interrupting streaming/playback? 

19. Does the API provide event-based audio clocks (“sync pulses”) rather than polling / callback-based clocks? Due to PDC (plugin delay compensation), not all plug-ins in the same logical graph pull / process-buffer are necessarily running in the same sampletime context. Is there a specification for clock updates during looping playback, particularly when loop boundaries are not aligned with natural audio-callback buffer boundaries)?

20. What is specified in relation to resource management / contention and OS access inc lifetime/cleanup? To what extent does the host try to provide/replace/mediate "operating system" services? (see also a11y/i8n/l8n and security context topics below).
    
21. What services are provided in relation to parameter layout semantics (e.g. control surface mapping “automap”, “scopes” in AU, Avid's page tables)?

22. Does the API provide any affordances for accessibility, internationalisation, localisation? (One valid answer: “we don’t solve this, use the OS services”, but should at least be a conscious decision).

23. What support is built-in for wrappers/bridges supporting out-of-process operation (application to application), plug-ins operating either as a separate process individually, or in a collection of plug-ins that’s isolated from the main host data model & GUI?

24. cf threading topic above, to what extent does your API specify mutexing / serial access (for non audio thread) - for many non-trivial plug-ins there are effectively two “serial” contexts (“realtime thread” and “everything else”) from an API perspective - even lightweight operations (get parameter value) are potentially unsafe in a dynamically configurable plugin. Is it better to put this responsibility (locking-correctness) on the host side given that there are fewer hosts with, on average, more competency in writing lock-correct code?

25. Data model state changes, ordering, visibility: both the UI thread (loadState, guiSetParameter…) and audio thread (parameter change event, program change event) can potentially update the master state. What is the correct behaviour for a plug-in in the event of possibly conflicting messages? Where does the master “state” live (“DSP object”, “GUI object”, or another object which lives on the UI thread but is not the GUI)? How is the DSP object kept updated in cases where the audio thread isn’t active?

26. “Parameters going through the host” (notification between plug-in’s GUI and DSP). Inevitably, in all but the most prescriptive APIs, there will be some aspects of state-change / state-messaging which do not pass through the host - even in a fully separated API, GUI and DSP will end up passing file URLs or other shared resource and then using them to transfer state. In which exact circumstances is “messages via the host” better, and why?

27. What support if any is provided for “idle” UI time management - plug-in timer callbacks vs host “idle” timer callbacks etc.?

28. Is there specific support for background thread management for plug-in worker tasks (low and high priority) - as distinct from multi-core audio processing (see point 15 above)? Low priority tasks e.g. preset database re-indexing, medium priority e.g. visualisers/analysers, high priority e.g. disk streaming?

29. Does the API enforce (or not) best practice in the design of plug-in state serialization, e.g. structure, versioning, portability.. a linear list of (0..1) parameters may be too basic for a lot of use cases, but binary opaque chunks are binary and opaque.

30. Is there any mechanism for dealing with partial non-compliance during a transition from legacy APIs and hosts - sometimes it is better for things to work imperfectly as long as people know it’s gonna be imperfect. MIDI has been imperfect for almost 40 years…

31. How is MIDI 2 supported?  Is native/raw MIDI2 data stream available by default, or only as a special-case, or not at all? As a large & complex spec it seems to have potential ramifications far larger than just “give me the data and let me get on with it”.

32. Rules around audio i/o buffer identities (the raw in-memory objects). Can in and out buffers be aliased / the same? Can two input (or output) buffers be the same? Can buffers in the frame be null pointers? What’s considered permissible if anything, other than normalised floats in-range? Denormals etc.?

33. Does your API require C++ ABI compatibility? (Beware, keep things simple, use C ABI).

34. RTL object compatibility, both c++ (std:: containers) and c (FILE* and other handles). Don't assume plugin and host will be linked to same RTL instance or even the same version.

35. Security contexts. How does your API manage potential differences in security context requirements (e.g. mic and camera access) between host and plug-in? Is there any mechanism for a plug-in to share state (both in memory and via persistent filesystem objects) between instances of itself that may be loaded in different application/host security contexts? For example making the same user-downloaded presets available in different applications.
    
36. Binary platform nuances - universal binaries and Rosetta on OS X. ARM64X vs ARM64EC vs emulation for Windows-ARM. What does the API specify and are there specific transitional arrangements?

37. Session portability and device substitution. Is there a mechanism whereby a component can proxy/stand-in for another when reloading old sessions? Can devices implementing this API exchange data with the other plug-in APIs in wider usage?
