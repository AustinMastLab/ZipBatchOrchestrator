# ZipBatchOrchestrator (AWS Step Function)

## Purpose
An AWS Step Functions state machine that builds the ZIP file for a large export (for example, a Zooniverse export) in parallel parts, so no single Lambda runs into its time or memory limits.

## Workflow
1. **Trigger**: Started by `App\Services\Actor\Zooniverse\ZooniverseZipTriggerService` when an export has more files than `services.aws.zip_threshold` (8,000). Smaller exports skip the state machine and go straight to `BiospexZipCreator` through SQS.
2. **SplitFiles** (Pass state): sets four fixed file ranges: 0–4,999, 5,000–9,999, 10,000–14,999 and 15,000–19,999.
    - Together they cover at most 20,000 files, the maximum expedition size (`config.expedition_size` in BIOSPEX). Raising that limit needs a change here first.
    - Every run uses all four ranges. For a range past the last file, `BiospexZipCreator` reports "No files found", which BIOSPEX ignores.
3. **MapState** (Map state): runs **BiospexZipCreator** once per range, up to four at a time. Each run zips its range of images into a partial ZIP in S3.
4. **MergeZips**: runs **BiospexZipMerger**, which combines the partial ZIPs into the final archive (`finalKey`), deletes the parts, and reports `zip-ready` to BIOSPEX through SQS (`updatesQueueUrl`).

The state machine calls both Lambdas without an alias, so it always runs their latest uploaded code (`$LATEST`) in every environment. See BIOSPEX issue #514 for the plan to pass an environment alias.

## Inputs/Outputs
- **Inputs (JSON)**:
    - `queueId`: ID of the export queue record.
    - `processDir`: The directory in S3 containing the source images.
    - `s3Bucket`: Target S3 bucket.
    - `totalSize`: Combined size of all images.
    - `fileCount`: Total number of images.
    - `updatesQueueUrl`: SQS URL for status reporting.
    - `finalKey`: The destination S3 key for the merged ZIP.
- **Outputs**:
    - S3: The merged ZIP at `finalKey`.
    - SQS: `zip-ready` from `BiospexZipMerger`, and a message from each `BiospexZipCreator` part, sent to `updatesQueueUrl`.

## Related Components
- **Laravel Service**: `App\Services\Actor\Zooniverse\ZooniverseZipTriggerService` (starts this state machine).
- **Laravel Command**: `App\Console\Commands\SqsListenerExportUpdate` (`export:listen`, receives the status messages).
- **Lambda**: `BiospexZipCreator` (creates each partial ZIP).
- **Lambda**: `BiospexZipMerger` (merges them into the final ZIP).

`BiospexBatchCreator` is not part of this state machine; it splits finished exports into downloadable batches in a separate flow.

## Configuration
- **ARN**: `arn:aws:states:us-east-2:147899039648:stateMachine:ZipBatchOrchestrator`
- **Region**: `us-east-2`
- **Definition**: `step-function.json`, updated with `aws stepfunctions update-state-machine` or in the AWS console.
