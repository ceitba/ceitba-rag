# 0010. Blob storage: Garage behind the S3 API (not MinIO)

- Status: Accepted
- Date: 2026-10-02
- Research: [04](../research/04-ingestion-sync-infra.md)

## Context
MinIO Community Edition lost its console (May 2025), stopped binary/image releases (Oct 2025),
entered maintenance mode (Dec 2025), was archived (2026), and its Docker Hub repos were deleted
(2026-09-11).

## Decision
- Self-host Garage on the VPS; access it only through the S3 API (boto3 / aws-sdk-go-v2) so the
  backend is swappable (RustFS, SeaweedFS, R2, B2).
- Content-addressed keys: `blobs/sha256/<hh>/<sha256>`; OCR JSON under `ocr/<ocr_version>/…`.
- Offsite backup to Backblaze B2.
- Users get our copy only via short-lived presigned URLs (ADR 0013).

## Consequences
Garage is AGPL; used unmodified as a separate service, no impact on this repo's license.
