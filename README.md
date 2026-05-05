# Cost-Aware Hockey Video Transfer and Transcoding Evaluation on AWS

## Main project report

## Project summary

This project was built to mimic a real FloSports-style media storage and transfer problem. In the actual workflow, large sports video assets are stored in Amazon S3 rather than sitting on a local machine. I recreated that pattern by uploading hockey game footage into S3, organizing it with input and output prefixes, and testing whether AWS could handle most of the heavy lifting before the files were moved to an external hard drive.

The problem started with a very practical constraint. There were around 1,200 hockey games, and each game was roughly 10 GB. That puts the total source footage close to 12 TB.

```text
1,200 games × 10 GB per game = about 12 TB
```

The client-provided hard drive was only 6 TB. Downloading all 1,200 original games one by one was not realistic because the files would exceed the hard drive capacity, take a long time to download manually, and repeatedly consume local desktop space during the transfer process. The manual process would also require constant cleanup on the local machine just to make space for the next batch of files.

The initial idea was to compress the videos first, then move the smaller versions to the hard drive. If each 10 GB game could be reduced to around 4.5 GB, then 1,200 games would fit much closer to the 6 TB drive capacity.

```tex
1,200 games × 4.5 GB compressed size = about 5.4 TB
```

That was the reason compression was considered. The goal was not just to make smaller files. The goal was to avoid a slow, manual, storage-heavy workflow where videos had to be downloaded locally, compressed locally, moved to the hard drive, deleted from the desktop, and repeated again and again.

Instead, the proposed workflow was to use AWS for the expensive operational part of the process:

Amazon S3 source footage
→ AWS Elemental MediaConvert compression
→ Amazon S3 compressed output
→ AWS CLI transfer directly to external HDD


This would mimic how FloSports-related media assets are stored in S3, keep the workflow cloud-centered, reduce dependency on local desktop storage, and only use the local machine as the place where the external hard drive is connected.

The technical test worked. A 10 GB hockey video was uploaded to S3, transcoded with AWS Elemental MediaConvert using HEVC/H.265, and reduced to about 4.5 GB. The compressed file played correctly in VLC, and the download time was shorter than the original file.

The cost result changed the final decision. The MediaConvert job cost about $12 for one video and took close to 2.45 hours. At 1,200 games, that would make compression financially unrealistic.

```text
1,200 games × $12 per MediaConvert job = about $14,400
```

At that point, buying another 6 TB hard drive was clearly cheaper than compressing every video through MediaConvert.

The final recommendation was to avoid bulk compression and use direct S3-to-HDD transfer with the AWS CLI. This still solved an important part of the workflow: it avoided downloading files through the browser one by one, skipped unnecessary local desktop storage usage, allowed transfer directly to the external hard drive, and could be resumed or repeated using AWS CLI commands.

The main outcome of the project was not that compression became the final solution. The main outcome was that I tested an AWS-native media workflow, measured the real cost, and changed the architecture based on evidence.

"Screenshot to include: S3 bucket showing input/ and output/ folders"

"Screenshot to include: MediaConvert completed job details showing codec, output path, and job status"

"Screenshot to include: AWS Billing or Cost Explorer showing MediaConvert charge"

## Problem I was trying to solve

The original manual option was inefficient.

The files were too large to treat like normal downloads. With 1,200 games at about 10 GB each, the total volume was around 12 TB. The available hard drive was 6 TB, so the full original dataset could not fit on the drive at once. Downloading everything manually would also have created several operational problems.

* Each game would need to be downloaded one by one or in small batches
* The local desktop would temporarily need enough free space to hold large video files
* Files would need to be moved from the desktop to the external drive
* The desktop would then need to be cleared before downloading the next batch
* Any interruption would require manual checking and restarting
* The process would be slow, repetitive, and easy to lose track of

The more scalable idea was to keep the workflow as close to S3 as possible. Since the source data was already modeled as being stored in S3, I wanted to test whether S3, MediaConvert, and AWS CLI could form a better pipeline.

The target workflow was:

```text
S3 input location
→ AWS-managed processing
→ S3 output location
→ direct transfer to HDD using AWS CLI
```

This would avoid using the local desktop as the main processing or staging layer. The desktop would only be used to run AWS CLI commands and connect the external hard drive.

The first version of the solution included MediaConvert because compression seemed like the most logical way to solve the hard drive capacity issue. If compression could reduce each game from 10 GB to about 4.5 GB, the entire 1,200-game set would be close to 5.4 TB and could fit on the 6 TB drive.

That assumption was technically correct but financially wrong.

The file size reduction was good. The processing cost was not.

## Why I pitched compression first

Compression was a reasonable first idea because the hard drive capacity problem was real.

The available drive was 6 TB, but the original footage was around 12 TB. Without compression, there were only a few options:

* Use more than one hard drive
* Buy a larger hard drive
* Transfer only part of the dataset
* Compress the videos before storing them on the drive

The compression path looked attractive because it could potentially solve both the capacity issue and the transfer-time issue. Smaller files would move faster, take up less space on the hard drive, and be easier to handle later.

HEVC/H.265 was selected because it is designed for better compression efficiency than older codecs such as H.264. The expectation was that MediaConvert could reduce file size while keeping game footage usable for playback and review.

The expected benefits were:

* Reduce each file from around 10 GB to a smaller playable version
* Fit more games onto the 6 TB hard drive
* Reduce download time from S3 to the hard drive
* Avoid using local desktop storage for compression work
* Let AWS handle the processing instead of a personal machine
* Create a repeatable cloud workflow that could later be automated

The test confirmed that the video could be compressed successfully. The 10 GB source became roughly 4.5 GB, which was close to the kind of reduction needed.

But the cost of that compression was too high to use across the full dataset.

## What I actually built and tested

I created an S3 bucket named:

```text
mediaconvert-hockey
```

The bucket was organized using prefixes:

```text
s3://mediaconvert-hockey/input/
s3://mediaconvert-hockey/output/
```

This directory-style layout was intentional. It was meant to mimic a real S3 media storage pattern where source files and processed outputs are separated by prefix.

The input prefix stored the original hockey video. The output prefix stored the transcoded video created by MediaConvert.

I uploaded one hockey game file of approximately 10 GB into the input folder:

```bash
aws s3 cp /path/to/original-game.mp4 s3://mediaconvert-hockey/input/
```

After upload, I created a MediaConvert job with these settings:

```text
Codec: HEVC / H.265
Container: MP4
Audio: AAC
Input: s3://mediaconvert-hockey/input/
Output: s3://mediaconvert-hockey/output/
```

The output was around 4.5 GB. I downloaded the output and tested playback locally using VLC because VLC supports HEVC playback.

The download timings were:

```text
Compressed 4.5 GB file: about 6 minutes
Original 10 GB file: about 15 minutes
```

This confirmed that compression improved file size and download time. It also confirmed that the compressed output was usable.

The issue was cost. The MediaConvert charge was about $12 for one video, and the processing time was close to 2.45 hours. That made the compression workflow unsuitable for the full 1,200-game dataset.

"Screenshot to include: terminal showing aws s3 cp upload command"

"Screenshot to include: terminal showing aws s3 cp download command"

"Screenshot to include: VLC playback of compressed HEVC output"

## AWS services and concepts used

### Amazon S3

S3 was used as the object storage layer. The bucket held both the raw source file and the compressed output file.

The important S3 concepts in this project were:

* Bucket creation
* Prefix-based organization using input/ and output/
* Object upload
* Object download
* S3 URI paths
* Data transfer out from S3 to a local hard drive
* Storage cost versus transfer cost separation

The biggest practical takeaway was that S3 storage cost was not the main issue because storage was already covered. The relevant cost was the transfer and the optional processing cost.

### AWS CLI

The AWS CLI was used to move files between S3 and the local machine or external hard drive.

Example upload command:

```bash
aws s3 cp /path/to/game.mp4 s3://mediaconvert-hockey/input/game.mp4
```

Example download command:

```bash
aws s3 cp s3://mediaconvert-hockey/output/game-compressed.mp4 /Volumes/HDD/hockey/game-compressed.mp4
```

For the final no-compression approach, the direct command would be:

```bash
aws s3 cp s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ --recursive
```

For a safer large transfer, I would use a dry run first:

```bash
aws s3 cp s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ --recursive --dryrun
```

Then run the actual transfer:

```bash
aws s3 cp s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ --recursive
```

For a resumable transfer, I would use sync:

```bash
aws s3 sync s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/
```

This is the better command for a large transfer because it avoids re-copying files that already exist at the destination.

This direct S3-to-HDD approach worked better than manually downloading files through a browser or staging them on the local desktop first. The local machine still runs the AWS CLI command, but the destination is the mounted hard drive, so the desktop does not become the main storage bottleneck.

### AWS Elemental MediaConvert

MediaConvert was used to test whether cloud-based transcoding could reduce video size enough to justify itself.

The service successfully converted the video from a larger source file into a smaller HEVC MP4 file. The technical result was good, but the cost result was not suitable for this use case.

What this project showed about MediaConvert:

* MediaConvert is useful for file-based video transcoding
* It can create smaller playable outputs from large source videos
* HEVC/H.265 can significantly reduce file size
* HEVC has compatibility trade-offs compared with H.264
* MediaConvert pricing can become expensive when scaled across many long videos
* Processing time matters operationally, especially when one job takes multiple hours
* A successful technical test can still be rejected if the cost model does not work

This was one of the main learning points of the project. MediaConvert did exactly what it was supposed to do, but the pricing did not fit this particular transfer-and-archive problem.

### IAM and service access

MediaConvert needed permission to read input files from S3 and write outputs back to S3. This introduced the idea of service roles and least-privilege access.

For a production version, the role would need permissions such as:

* Read access to the input S3 prefix
* Write access to the output S3 prefix
* Permission for MediaConvert to assume the service role
* Logging permissions if CloudWatch logging is enabled

Even though this project was small, it still followed the same access pattern used in production media workflows: storage is separated from processing, and the processing service needs explicit access to the storage layer.

### EventBridge and notification design

Because MediaConvert jobs can take a long time, manually checking the console is not a good operational workflow.

For this project, the MediaConvert job took close to 2.45 hours. A more production-ready version would use EventBridge to listen for MediaConvert job state changes and trigger a notification when the job completes or fails.

The logical event flow is:

```text
MediaConvert job status changes
→ EventBridge rule catches COMPLETE or ERROR status
→ SNS topic or Lambda target is triggered
→ Email or operational notification is sent
```

The event pattern would look conceptually like this:

```json
{
  "source": ["aws.mediaconvert"],
  "detail-type": ["MediaConvert Job State Change"],
  "detail": {
    "status": ["COMPLETE", "ERROR"]
  }
}
```

This is the right monitoring pattern for long-running MediaConvert jobs. I would only describe it as implemented if the EventBridge rule and notification were actually created. Otherwise, it belongs in the report as a production extension or next version design.

"Screenshot to include: EventBridge rule pattern for MediaConvert COMPLETE or ERROR events, if implemented"

"Screenshot to include: SNS email subscription confirmation, if implemented"

### CloudWatch and billing monitoring

CloudWatch and AWS billing tools were relevant for two different reasons.

CloudWatch is useful for operational monitoring, such as job errors, queue backlog, and job activity. AWS billing tools are useful for understanding whether the architecture is financially acceptable.

For this project, the most important monitoring output was the billing result. The MediaConvert charge revealed that the compression approach would not scale economically.

A budget alert is useful for this kind of experiment because video services can become expensive quickly. A practical AWS Budget would notify me when the account crosses a small threshold during testing.

Example budget thresholds for a learning project:

```text
Alert at $5 actual spend
Alert at $10 actual spend
Alert at $20 forecasted spend
```

This is not just a cost-saving feature. It is part of responsible cloud engineering.

"Screenshot to include: AWS Budget alert configuration"

"Screenshot to include: Cost Explorer filtered to Elemental MediaConvert"

## Cost analysis

The original test produced these numbers:

```text
Original file size: about 10 GB
Compressed file size: about 4.5 GB
Reduction: about 55%
MediaConvert processing time: about 2.45 hours
MediaConvert charge: about $12
```

At first, a 55% size reduction looked useful. It meant the full dataset could theoretically fit closer to the 6 TB hard drive size.

```text
1,200 original games × 10 GB = about 12 TB
1,200 compressed games × 4.5 GB = about 5.4 TB
```

That would have solved the hard drive capacity problem. But the processing cost made the approach unrealistic.

Estimated compression cost if every game was transcoded first:

```text
1,200 games × $12 = about $14,400
```

Estimated direct S3-to-HDD transfer cost for the original files:

```text
About $1,060 before tax
```

The comparison made the decision clear. Compressing everything would cost far more than simply buying another 6 TB hard drive and transferring the original videos directly.

Final decision:

```text
Do not compress all 1,200 videos.
Use direct S3-to-HDD transfer with AWS CLI.
Buy or use additional HDD capacity if needed.
```

## Why direct S3-to-HDD transfer was the better final choice

Direct transfer was better for this workload because the files already existed in S3 and the company was already paying for storage. There was no need to create a second compressed copy of every game if the goal was simply to move the files to physical storage.

The direct transfer approach had these advantages:

* No MediaConvert processing cost
* No 2.45-hour processing wait per game
* No extra compressed copy stored in S3
* No HEVC compatibility concern
* No repeated browser-based manual downloads
* Less dependency on local desktop storage
* Transfer can target the mounted external hard drive directly
* AWS CLI sync can resume or skip files that already exist
* Multiple transfers can be run in a controlled way if the source folders are split logically

The final workflow became:

```text
FloSports-style S3 source location
→ AWS CLI bulk copy or sync
→ External HDD
→ File count and size validation
```

For a one-time archive transfer, this is more appropriate than building a full transcoding pipeline.

## Final architecture

The final architecture is intentionally simple because the cost analysis showed that simple was better.

```text
Amazon S3
  source prefix containing original game files

AWS CLI
  aws s3 sync or aws s3 cp --recursive

Local machine
  runs the command
  external HDD is mounted as destination

External HDD
  stores downloaded games directly

Validation
  compare file count
  compare total transferred size
  spot-check video playback
```

For a production-style version, I would add:

```text
AWS Budgets
  email alert for unexpected spend

CloudWatch / EventBridge
  optional monitoring for any future MediaConvert jobs

S3 Inventory or object listing
  validation of total expected files

Transfer logs
  local record of completed downloads
```

Step Functions would only make sense if the workflow became more complex, for example if each video required validation, metadata extraction, conditional transcoding, retry logic, and final database updates. For this project, Step Functions would be over-engineering.

## Commands used or recommended

Upload one file to S3:

```bash
aws s3 cp /path/to/game.mp4 s3://mediaconvert-hockey/input/game.mp4
```

Download one compressed output file:

```bash
aws s3 cp s3://mediaconvert-hockey/output/game-compressed.mp4 /Volumes/HDD/hockey/game-compressed.mp4
```

Download one original file directly:

```bash
aws s3 cp s3://mediaconvert-hockey/input/game.mp4 /Volumes/HDD/hockey/game.mp4
```

Preview a bulk transfer without copying anything:

```bash
aws s3 cp s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ --recursive --dryrun
```

Bulk copy all games:

```bash
aws s3 cp s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ --recursive
```

Recommended resumable approach:

```bash
aws s3 sync s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/
```

Log transfer output:

```bash
aws s3 sync s3://mediaconvert-hockey/input/ /Volumes/HDD/hockey-games/ > transfer-log.txt 2>&1
```

Count local files after transfer:

```bash
find /Volumes/HDD/hockey-games/ -type f | wc -l
```

Check local folder size:

```bash
du -sh /Volumes/HDD/hockey-games/
```

List S3 objects:

```bash
aws s3 ls s3://mediaconvert-hockey/input/ --recursive --summarize
```

## Parallel transfer note

The final direct transfer workflow can be made faster by splitting files into groups and running separate AWS CLI transfers to the mounted hard drive. This should be done carefully so the local network, hard drive write speed, and AWS transfer limits are not overloaded.

A safer approach is to split the transfer by prefix or folder, for example:

```bash
aws s3 sync s3://mediaconvert-hockey/input/batch-01/ /Volumes/HDD/hockey-games/batch-01/
aws s3 sync s3://mediaconvert-hockey/input/batch-02/ /Volumes/HDD/hockey-games/batch-02/
```

This avoids manually downloading games one by one while still keeping the workflow understandable and recoverable.

## What I learned

This project helped me understand that AWS design is not only about making a service work. It is about deciding whether the service should be used at all.

The MediaConvert result was technically successful. The video compressed from about 10 GB to about 4.5 GB and played correctly. But the job cost about $12 and took close to 2.45 hours. At 1,200 games, this would have created an estimated transcoding cost of about $14,400, which was not acceptable for this use case.

The better solution was to keep the original videos and move them directly from S3 to an HDD using the AWS CLI. This made better use of the existing S3 storage setup, avoided unnecessary processing, and kept the workflow cost-aware.

The biggest learning was that optimization has to be measured against the actual bottleneck. Compression reduced file size, but cost became the bottleneck. Once that was clear, the architecture changed.

The project also gave practical exposure to:

- S3 bucket and prefix organization
- AWS CLI file movement
- MediaConvert job configuration
- HEVC/H.265 compression behavior
- S3 input and output workflows
- Service-role style access between AWS services
- AWS billing investigation
- Budget alert planning
- EventBridge-style job notification design
- Cost-based cloud architecture decisions

## Final conclusion

The project started with a compression idea and ended with a cost-aware transfer decision.

The manual option was to download around 1,200 games one by one, manage local desktop space, move files to a 6 TB hard drive, delete local copies, and repeat the process. That would have been slow and operationally messy.

The compression option was tested because the original dataset was around 12 TB and the available drive was only 6 TB. MediaConvert proved that a 10 GB game could be reduced to about 4.5 GB, which would make the dataset fit much closer to the hard drive limit.

But the $12 cost for a single MediaConvert job made the full compression plan too expensive. At 1,200 games, it would cost about $14,400 just to process the videos. Buying additional hard drive capacity was cheaper than compressing everything in AWS.

The final solution was to use direct S3-to-HDD transfer through the AWS CLI. This avoided unnecessary transcoding, kept the original files intact, skipped browser-based manual downloading, and reduced the local desktop’s role to simply running the CLI while the external hard drive acted as the destination.

The value of the project is not that I forced a complicated AWS architecture. The value is that I tested an AWS media service, measured the real cost, rejected the expensive option, and selected a simpler architecture based on evidence.

That is the kind of decision-making that matters in real cloud engineering.





## What screenshots to include

Use screenshots where they prove something real happened. Avoid uploading videos, private data, account IDs, credentials, or internal company information.

* "S3 bucket showing input/ and output/ prefixes"
* "Uploaded 10 GB object inside the S3 input prefix"
* "MediaConvert job settings showing HEVC/H.265, MP4, and AAC"
* "MediaConvert job status showing COMPLETE"
* "S3 output prefix showing the compressed 4.5 GB file"
* "VLC playback of the compressed output"
* "AWS Billing or Cost Explorer showing the MediaConvert charge"
* "Terminal showing AWS CLI upload or download command"
* "AWS Budget alert configuration, if created"
* "EventBridge rule for MediaConvert job state change, if created"


Recommended references:

* AWS Elemental MediaConvert pricing
* AWS Elemental MediaConvert user guide
* AWS MediaConvert EventBridge events
* AWS MediaConvert job status change events
* AWS MediaConvert queues and parallel job processing
* AWS VOD automation watchfolder pattern using S3, Lambda, MediaConvert, CloudWatch Events, and SNS
* Amazon S3 pricing
* AWS CLI `s3 cp` command reference
* AWS CLI `s3 sync` command reference
* AWS Budgets notification documentation
* GitHub documentation for adding locally hosted projects using Git
* GitHub Markdown formatting documentation
