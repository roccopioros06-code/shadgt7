# AMD (RX 6900 XT): horizontal stripes in PS4 Pro 4K mode — cause and fix (FIX-AMD-006)

## Symptom

With the PS4 Pro checkerboard picture (3200×1800, "PS4 Pro, 4K TV" or any mode with the
checkerboard box ticked) on an RX 6900 XT (Adrenalin 26.10.1, Windows), the picture shows
horizontal dashes on the menu map and in races: 8×4-pixel blocks, the bottom halves of 8×8 tiles,
painted with the fog / sky colour. They come and go (some sessions start clean), but show up in
almost every session, and are much worse on near geometry (cockpit, car body). Not seen on NVIDIA.

## Cause

The atmosphere pass (fs `0x2b49e735`, vs `0x2dcc657a`) and a compute pass before it
(cs `0xa7035d98`) sample the checkerboard's 2-sample depth (`0x1021200000`, 1920×2160
D32SfloatS8Uint, AMD mixed attachment samples with the 1-sample colour) through an MSAA T#
(type 14, sample 0).

On this AMD driver, that depth read anywhere outside the depth unit comes back as "far" in some
8×4 blocks, so the atmosphere pass paints only fog or sky there:

- read by the texture units in the GENERAL layout (as the renderer does): stripes;
- a `vkCmdCopyImage` of it to another 2-sample depth image, sampled instead: many more stripes;
- a depth-unit resolve of it (`resolveMode = SAMPLE_ZERO`, as the frame dump already does):
  whole and correct.

Things that did not help (each built and tested in game):

- a copy of the scene for the atmosphere pass, which samples the colour target it draws into
  (a real feedback loop: `VK_EXT_attachment_feedback_loop_dynamic_state` is missing on this
  driver, so `attachment_feedback_loop` is off);
- GENERAL → DEPTH_STENCIL_READ_ONLY_OPTIMAL → GENERAL transitions around the reads;
- making the depth image without `MUTABLE_FORMAT` / `EXTENDED_USAGE`;
- `VK_EXT_image_compression_control` (not exposed by this driver).

## Fix (FIX-AMD-006)

Before a draw or dispatch that samples a multisampled depth target (one it doesn't also draw
into), the renderer:

1. resolves sample 0 into a 1-sample depth image with an empty dynamic-rendering pass
   (`resolveMode = SAMPLE_ZERO`);
2. draws that image into an R32 colour image with the depth's sample count, with the existing
   `fs_tri.vert` + `ms_image_blit.frag` (non-MSAA variant), so every sample holds sample 0's depth;
3. rebinds the draw's / dispatch's sampled views of the depth to that colour image.

The translated shaders declare their images with `Depth = 0`, so the R32 colour view fits the
same descriptor. Afterwards the list's pipeline, bindings and dynamic state are invalidated, and
an open pass is resumed without its clears.

It is on where the device lacks `VK_EXT_attachment_feedback_loop_*` (i.e. AMD), and
`RYGT_MSAA_DEPTH_EXPAND=0` / `=1` forces it off / on everywhere.

## Results

RX 6900 XT, Ryzen 7 5800X, 1440p, every PS4 Pro option on, FSR 3.1:

- menu map, day race, sunset / night race, cockpit, mirrors, rear view: no stripes;
- `RYGT_MSAA_DEPTH_EXPAND=0` (same build): stripes back;
- frame rate unchanged (about 56 fps in the same scenes before and after).

## Patch

Against `3e2b8d76d2c38b5da09aadf5b8be7ff76d7f6d64`:

- [`patches/amd-msaa-depth-expand.patch`](amd-msaa-depth-expand.patch): FIX-AMD-006 only.
- [`patches/amd-fix7.patch`](amd-fix7.patch): the build tested in game, which also carries the
  scene copy for the atmosphere pass (FIX-AMD-003) and diagnostic log lines.
