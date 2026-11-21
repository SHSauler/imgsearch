# People (face grouping)

People group faces across images and videos by identity, like Collections
group media by hand. Whole-image similarity search cannot do this: two photos
of the same person in different places have distant global embeddings.

## Data model (migration 17)

- `persons(id, uuid, name, created_at)`: a cluster of faces. `name`
  defaults to neutral `Person N`; the backend always keeps a UUID.
- `face_detections(id, image_id, person_id NULLABLE, box_*, embedding_blob,
  confidence, created_at)`: one detected face. Faces reference image rows
  only; video membership is derived through `video_frames`, so frame faces
  surface as their video (same rule as collections: save the video, not its
  derived frame). `person_id` is nullable so detector output can wait in an
  unassigned pool before clustering assigns it.

Deletion: deleting a person keeps its faces as unassigned (reassignable);
deleting media cascades to its faces. Membership removes nothing else.

## API

All routes use the existing `/api/` authentication and JSON error format.

| Method | Route | Notes |
| --- | --- | --- |
| GET | `/api/people` | `{people:[...]}`, newest first |
| GET | `/api/people?summary=true` | Adds `face_count` + up to 4 newest face-image covers |
| POST | `/api/people` | `{name}` optional; omitted yields neutral `Person N` auto-name, `201` |
| GET/PATCH/DELETE | `/api/people/{id}` | Rename/retire; names are 1–100 chars, no control chars |
| GET | `/api/people/{id}/items` | `{items,total,item_count}`, deduplicated media newest-face-first, NSFW-filtered before pagination like collections |
| POST | `/api/people/merge` | `{from_ids, into_id}`: moves faces, deletes sources |
| POST | `/api/people/{id}/split` | `{face_ids, name?}`: moves faces to a new person, `201` |
| GET | `/api/faces` | `?person_id=` or `?unassigned=true`, with thumbnails |
| POST | `/api/faces/{id}/reassign` | `{person_id}` or `{person_id:null}` |

Face creation belongs to the background detection worker (planned); the API
only clusters, corrects, and reads.

## Pipeline (planned)

Detect (multi-face, e.g. SCRFD-class) → align → identity embedding (ArcFace
class, 512-d) → greedy nearest-neighbor match to existing persons above a
cosine threshold, else new `Person N`. Runs as a new `index_faces` job kind
after embedding, on image rows and sampled video frames, via the existing
ONNX Runtime dependency. VLM annotators (Gemma/Qwen) stay on captions: they
have no stable identity metric.
