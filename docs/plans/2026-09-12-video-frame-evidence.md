# Video frame evidence

Status: design and staged implementation; video ingestion currently transcribes
only the audio track. This plan does not claim video RAG parity.

## Evidence contract

Retain the original video and derive bounded PNG frames using an explicitly
configured local ffmpeg/ffprobe pair. Choose the first non-attached video stream.
Inventory decoded presentation timestamps rather than deriving time from FPS.
For a fixed sampling interval, select the first decoded frame at or after each
requested playback time, recording both requested time and actual presentation
time. Duplicate choices coalesce, with all requested times retained. Never
invent a final frame when a sampling request exceeds the last decoded timestamp.

Record original video SHA-256, stream index, decoded frame ordinal, original PTS,
playback-relative timestamp, displayed dimensions, PNG SHA-256, policy revision
and decoder identity. Preserve exact rational stream time base and integer PTS
where available. Reject missing, nonfinite or nonmonotone timestamp inventories.
Autorotation affects displayed pixels and dimensions, not source attribution.

Bound input bytes, duration, decoded inventory bytes and frame count, selected
frame count, per-frame/total PNG bytes, pixels and one shared wall deadline.
Use owned subprocess groups and cancellation cleanup. No shell, network inputs,
automatic executable discovery, model downloads or implicit inference. Reject
exhausted budgets; do not silently truncate and imply full completion.

## Stages

1. Implement and test deterministic sampling and checked PNG decoding. Tests use
   native locally generated constant/VFR videos, nonzero time origins, rotation,
   no-audio sources, sparse frames, duplicate choices and malformed inventories.
2. Add explicit frame OCR with exact UTF-8 spans and displayed-frame boxes.
   Retain frame-specific provenance and distinguish empty OCR from failed work.
   Current ParsedDocument requires text: an all-empty OCR result must remain
   refused until the storage contract explicitly supports visual-only sources.
3. Persist source/policy/decoder/OCR-model-bound frame receipts. Recover only
   validated completed frames. Test actual process termination, corrupt receipts,
   changed configuration and provider-free completed replay.
4. Expose checked frame reads from retained sources. Revalidate source/link and
   current host authority after asynchronous work. Forget/unlink/space deletion
   must suppress stale pixels. Reads never call OCR or vision models.
5. Add explicit user-selected vision interpretation of exact retained frames.
   Generated descriptions retain model identity and origin=model_generated;
   they must not enter store_document's extracted_text path as quotations.
6. Connect frame OCR/retrieval/citations and web UI: actual timestamp, exact frame,
   sampling interval/counts/empty/failed coverage, and original video playback.
   Validate native API and browser behavior, including restart and deletion.

Each stage gets focused tests/type checks and independent review. A partial stage
is not closure of the K04 video or multimodal retrieval gap. Keep audio transcript
behavior compatible; silent video requires frame evidence, not fake transcript.

## Reference behavior inspected, no copied implementation

- LlamaIndex multi_modal_video_RAG notebook samples images separately from audio
  transcript and retrieves both. It motivates independent visual evidence.
- RAGFlow vision code samples chronological video frames and separately supports
  image OCR/description. Neither inspected path establishes our retained-frame
  provenance and recovery contract.
- Official ffprobe documentation: https://ffmpeg.org/ffprobe.html (stream
  specifier V excludes attached pictures; machine-readable frame inventory).

## Prerequisite found during review

Existing image-understanding requests can publish after source removal or host
key revocation during inference. Fix in fix/image-understanding-source-races
before reusing that authorization/publication path for video frames.
