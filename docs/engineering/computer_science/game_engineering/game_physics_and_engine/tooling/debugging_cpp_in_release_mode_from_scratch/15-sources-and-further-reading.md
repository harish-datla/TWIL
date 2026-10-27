# Sources & further reading

Numbered citations refer to the superscripts above. Primary references first, practitioner resources second.
A note on originality

The prose, code samples, CSS, and interactive widgets on this page are original writing. The shadow-memory mechanism in §8's ASan widget follows the published AddressSanitizer paper [6]; the shadow encoding (chunk fully-addressable / partial / poisoned) is the actual encoding ASan uses. The DWARF call-frame description in §6 is the standard mechanism documented in the DWARF 5 specification [7]. The happens-before model behind §11's TSan visualization is the one described in [27]. The release-vs-debug example in §1 is a standard demonstration of an optimizer exploiting signed-overflow UB, repeated in many compiler-engineering talks.
