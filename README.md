# crossfader

The Event-Driven Flow:

INFRA (not services, just things that exist)
- Kafka        > the bus
- Postgres     > job state + playlists
- MinIO / S3   > the actual audio files

SERVICES (4)

1. api  (FastAPI)
   - POST /jobs {track_urls: [a, b]}
   - writes job row: status=queued, tracks=[a,b]
   - publishes  -> download.requested  (one per track)
   - returns    job_id immediately
   - GET /jobs/{id}  -> reads status from Postgres (frontend polls this)

2. downloader
   - consumes   <- download.requested {job_id, track_id, url}
   - yt-dlp -> uploads audio to S3
   - updates track row: status=ready, s3_key=...
   - publishes  -> track.downloaded {job_id, track_id, s3_key}

3. mixer  (ML engine, separate repo)
   - consumes   <- track.downloaded
   - BARRIER: checks Postgres — are ALL tracks for this job ready?
       - no  -> ack and do nothing
       - yes -> pull both from S3, crossfade, upload result
   - publishes  -> mix.completed {job_id, s3_key}

4. frontend
   - POST job, then poll GET /jobs/{id} until status=done
   - plays result from S3 presigned URL

TOPICS (3)
- download.requested
- track.downloaded
- mix.completed