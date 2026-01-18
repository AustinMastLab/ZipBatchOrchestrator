# ZipBatchOrchestrator (AWS Step Function)

## Purpose
An AWS Step Function workflow that orchestrates the creation of multiple ZIP files for large exports (e.g., Zooniverse). It prevents Lambda timeout and memory issues by splitting a massive image collection into smaller, manageable batches.

## Workflow
1. **Trigger**: Invoked by `App\Services\Actor\Zooniverse\ZooniverseZipTriggerService` when an export exceeds the `zip_threshold` (defined in Laravel config).
2. **BiospexBatchCreator**:
    - Downloads the source manifest/archive.
    - Splits the manifest into smaller chunks (Batch size: 2000).
    - Uploads batch CSVs and individual image files to a temporary location.
3. **Map State (Parallel Processing)**:
    - Iterates over the list of created batches.
    - For each batch, it invokes the **BiospexZipCreator** Lambda.
4. **BiospexZipCreator**:
    - Takes a batch manifest and creates a partial ZIP file (e.g., `filename-part1.zip`).
    - Uploads the partial ZIP to S3.
5. **BiospexZipMerger**:
    - Once all parallel batch tasks are complete, this Lambda merges the partial ZIPs into a single final archive.
6. **Completion/Callback**:
    - The final result is uploaded to the destination S3 key.
    - A success/failure notification is sent back to the Laravel application via SQS (`updatesQueueUrl`).

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
    - S3: Final merged ZIP file.
    - SQS: Success/Failure notification sent to Laravel.

## Related Components
- **Laravel Service**: `App\Services\Actor\Zooniverse\ZooniverseZipTriggerService` (Triggers this State Machine).
- **Lambda**: `BiospexBatchCreator` (Splits the manifest).
- **Lambda**: `BiospexZipCreator` (Creates batch ZIPs).
- **Lambda**: `BiospexZipMerger` (Finalizes the export).

## Configuration
- **ARN**: `arn:aws:states:us-east-2:147899039648:stateMachine:ZipBatchOrchestrator`
- **Region**: `us-east-2`