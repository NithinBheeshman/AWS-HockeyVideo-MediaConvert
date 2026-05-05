
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
