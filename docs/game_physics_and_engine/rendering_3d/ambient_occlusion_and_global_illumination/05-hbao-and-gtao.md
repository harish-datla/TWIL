# HBAO and GTAO

SSAO counts random samples and only loosely estimates the directional AO integral. Horizon-based AO (HBAO) treats the depth buffer as a height field, marches in several directions, finds the steepest horizon angle, and integrates the blocked portion. It is more geometrically faithful than random counting.

GTAO (ground-truth ambient occlusion) is a horizon method tuned against a ray-traced AO reference, with a fitted multi-bounce term that reduces single-bounce over-darkening. “Ground truth” means matching a Monte Carlo AO reference, not solving global illumination.

Both remain screen-space. They still miss off-screen and hidden geometry; they improve AO fidelity without adding light transport or color bleeding.
