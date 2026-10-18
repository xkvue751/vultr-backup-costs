# cloud backup storage: what 1 TB really costs, the egress trap most guides skip, and how to set up an S3-compatible backup target with Vultr

Cloud backup storage is the quiet half of a good backup plan: a remote bucket holding an encrypted copy of your data, ideally untouched until the day everything goes wrong. Most articles on the topic quote a storage price per terabyte and stop there. That skips the two things that actually decide whether a backup target works: what it costs to pull your data back out, and whether ransomware that lands on your PC can reach into the bucket and delete the copy.

The short answer: for personal and small-business backups up to a few terabytes, an S3-compatible bucket with flat monthly pricing and bandwidth included is the sweet spot. Vultr's Object Storage sits right there — $6/month for 1 TB of storage with 1 TB of bandwidth included, compatible with Duplicati, restic, and rclone, and no contract attached. If you need vendor-enforced immutability for compliance reasons, object stores that support S3 Object Lock do that job natively; more on that trade-off below.

Here are the numbers, the traps, and the setup steps.

> **Short on time?** Vultr Object Storage: 1 TB for **$6/month**, 1 TB of bandwidth included, S3-compatible, billed month to month. 👉 [See Vultr's current Object Storage plans and prices](https://www.vultr.com/products/object-storage/?ref=7268283). If hard vendor-side WORM storage is a requirement, Backblaze B2 and AWS S3 both support Object Lock natively — the rest of this guide explains when that matters.

## Vultr Object Storage: plans and pricing

Vultr currently sells object storage in three flat monthly tiers, and every tier includes bandwidth equal to its storage capacity. That second part matters more than it sounds, because most cloud providers bill downloads separately — more on that in a moment.

| Plan | Storage | Included bandwidth | Monthly price | Sign up |
| --- | --- | --- | --- | --- |
| 1 TB | 1 TB | 1 TB | **$6** | [Sign up for the 1 TB tier](https://www.vultr.com/products/object-storage/?ref=7268283) |
| 5 TB | 5 TB | 5 TB | **$20** | [Sign up for the 5 TB tier](https://www.vultr.com/products/object-storage/?ref=7268283) |
| 20 TB | 20 TB | 20 TB | **$60** | [Sign up for the 20 TB tier](https://www.vultr.com/products/object-storage/?ref=7268283) |

All three tiers speak the S3 API, so any backup tool that can write to S3-compatible storage can write to these. Capacity comes in fixed chunks rather than a soft per-GB slide: if a bucket outgrows 1 TB, you move up a tier or split the archive across buckets. Pricing is month to month, and you can destroy a subscription whenever you like.

A few other Vultr products slot into a backup setup if you'd rather build your own target:

- **Block Storage** — mounted disks at $0.10/GB/month for NVMe or $0.05/GB/month for HDD. Useful as the storage behind a dedicated backup server. 👉 [Check Vultr Block Storage pricing](https://www.vultr.com/products/block-storage/?ref=7268283)
- **Cloud Compute** from $6/month (1 vCPU, 1 GB RAM, 25 GB SSD, 1 TB bandwidth) — enough to run restic, Duplicati, or Borg as a small backup server.
- **Snapshots** at $0.05/GB/month for whole-server images, plus scheduled automatic backups on compute instances billed at 20% of the instance price.

For the typical "get my offsite copy off my premises" job, though, the object storage tiers are the whole story.

## What the same terabyte of cloud backup storage costs elsewhere

On the big three clouds, hot object storage runs roughly $18–23 per TB per month as of mid-2026; backup-focused providers cluster between $6 and $15. Here's how the main options line up for a standard hot-tier backup bucket:

| Provider | Storage per TB/month | Egress on restore | Immutability features |
| --- | --- | --- | --- |
| Vultr Object Storage | $6 (1 TB tier) | 1 TB/month included, then $0.01/GB | Handled by your backup tool |
| Amazon S3 Standard | ~$23 | $0.09/GB after 100 GB free | Object Lock, versioning, fine-grained IAM |
| Azure Blob (Hot) | ~$18 | $0.087/GB after 100 GB free | Immutable policies, soft delete, RBAC |
| Backblaze B2 | ~$6.95 | Free up to 3× stored volume, then $0.01/GB | Object Lock, versioning, scoped app keys |
| Cloudflare R2 | $15 | No egress charges | — |

The hyperscalers get cheaper as you go colder: S3 Glacier Deep Archive and Azure Blob Archive sit around $1/TB/month, with Google's archive tier near $1.20. For data you genuinely never touch, that's a fair price. The catches show up in the fine print: Standard-IA carries a $0.01/GB retrieval fee and roughly double the request rates of Standard, and minimum storage durations bite — 30 days on Standard-IA, 90 on Glacier Flexible, 180 on Deep Archive. Delete early and you pay as if the data had stayed the whole term.

Backup software makes this worse than most people expect. Tools like Duplicati run retention pruning and compaction jobs that read and rewrite old backup volumes, which is exactly the access pattern the cheap tiers punish. The workable approach: keep your active backup set on a hot or cool tier with no retrieval fee, and if you want archival pricing, let the provider's lifecycle rules move *old* data down while turning off compaction for that job.

One more line item people miss with managed services: AWS Backup bills restores and cross-region data transfers separately from storage, and cold-storage restores cost more than warm ones. The sticker price on the bucket is only the beginning of the bill.

## The egress trap: what a restore actually costs

Egress is what you pay to download your own data, and it lands on your worst day — a ransomware rebuild, a dead NAS, a deleted folder you notice too late.

The math on the hyperscalers is unforgiving. Amazon charges $0.09/GB for data leaving S3 to the public internet (the first 100 GB per month is free), and Azure is nearly identical at $0.087/GB. Restoring 5 TB after a ransomware event costs about $450 on either platform, on top of the outage itself.

Vultr's model handles this differently: each tier includes a monthly bandwidth allowance equal to its storage size, and overage beyond that is $0.01/GB. A full restore of a 1 TB bucket fits inside what the $6 plan already covers. That has a practical side benefit — you can run a genuine test restore, the single most skipped step in backup hygiene, without generating a surprise line item. Backblaze B2 is generous here too, with free egress up to 3× your stored volume; beyond that it's $0.01/GB. Either way, both beat a four-figure restore bill.

## Ransomware protection: two layers, one honest caveat

The backup target has to survive the machine it protects being fully compromised. Modern ransomware reliably hunts for backup software on the machine it infects, reads the stored credentials, and deletes the remote copy before encrypting anything. Client-side encryption doesn't help there — an attacker doesn't need to read your backups, only delete them.

Two layers of defense, and you want both:

1. **Least-privilege credentials.** Generate the S3 key pair from the Object Storage subscription itself and never reuse your main Vultr API key for backup jobs. The narrower the credential, the less a compromised machine can do with it. The same principle applies everywhere: scoped IAM on S3, bucket-restricted application keys on B2, chrooted users on SFTP.
2. **Versioning the backup tool can enforce.** Since Vultr's object storage doesn't advertise S3 Object Lock support, plan for immutability at the tool level. rclone's `--backup-dir` flag parks deleted and overwritten files in a separate directory. restic and Duplicati keep their own version history, so old snapshots can't be pruned without the client's configuration. For belt-and-suspenders, keep a second copy somewhere else entirely — a rotated offline disk counts.

If vendor-enforced WORM storage is a hard requirement — some compliance regimes effectively demand it — the honest answer is that Object Lock-capable providers fit that job better. Decide which requirement is actually yours before picking a bucket.

## Setting up Vultr as a backup target

The whole job is three steps: create an account, deploy a bucket, point your backup software at it. The bucket configuration takes minutes; the only fiddly part is getting the endpoint and keys right the first time.

1. 👉 [Create a Vultr account](https://bit.ly/VulTR) — pay-as-you-go, no commitment needed to poke around.
2. In the customer portal, deploy Object Storage in a region near you (Vultr runs 32 data center locations across six continents). Note the S3 endpoint it gives you — for example, New Jersey deployments use hostnames like `ewr1.vultrobjects.com` — and generate the S3 access and secret keys for that subscription.
3. **rclone:** run `rclone config`, create a remote of type `s3`, leave the provider generic, and paste in the custom endpoint plus your keys. Verify with `rclone lsd remote:` — you should see your bucket listed.
4. **restic:** initialize straight against the endpoint, e.g. `restic -r s3:ewr1.vultrobjects.com/your-bucket init`.
5. **Duplicati:** choose S3-compatible as the destination and enter Vultr's endpoint as the custom server URL. Duplicati encrypts before upload, so the keys never leave your machine.
6. Within the first month, do one full test restore. On the included-bandwidth plans this costs nothing extra, and it's the only way to find out your restore path works before you need it.

## When cloud backup storage on Vultr is the wrong pick

No single target fits every job, and pretending otherwise helps nobody. A few cases where you should look elsewhere:

- **Compliance-grade immutability.** If you need storage the provider itself refuses to delete or overwrite for a fixed period, use an Object Lock-capable store (AWS S3 and Backblaze B2 both offer it). Vultr's object storage doesn't advertise that feature.
- **Set-and-forget whole-PC backup.** Unlimited per-computer consumer backup services exist for around $99/year, and if that describes your need exactly, they're less work than a bucket.
- **A decades-long cold archive.** The ~$1/TB archive tiers from the big clouds are cheaper per month than any hot storage — just respect the 90–180 day minimum durations, watch retrieval fees, and switch off compaction on archive jobs so your backup tool doesn't fight the tier's economics.
- **Under 2 TB on a single computer.** Consumer storage plans around $100/year for 2 TB are simpler, and free allowances like MEGA's 20 GB cover tiny collections. One warning travels with these: a synced folder is a mirror, and deletions propagate. Version history is the only thing standing between you and that; check its length before trusting any consumer cloud with your only offsite copy.

## Frequently asked questions

### Is the $6 Object Storage plan really $6?

Yes. The monthly price covers the bucket, 1 TB of storage, and 1 TB of bandwidth. Vultr doesn't itemize per-request charges on Object Storage the way AWS does for S3, so the main variable cost is bandwidth past the included allowance at $0.01/GB.

### What happens when my backup grows past 1 TB?

You upgrade to the next tier ($20 covers 5 TB with 5 TB of bandwidth) or split data across buckets. At the 5 TB tier the effective price is $4/TB — still a fraction of hyperscaler hot storage, and roughly even with Backblaze B2 at current list prices.

### Can I use it with Duplicati, restic, or rclone?

Yes. All three speak S3-compatible storage with a custom endpoint, and all three encrypt client-side, so the provider holds ciphertext it can't read. That's what you want in a backup target.

### Do I pay every time I download my backups?

You consume the tier's monthly bandwidth allowance first. A full restore of a 1 TB bucket fits inside the $6 plan's included 1 TB. Past the allowance, it's $0.01/GB.

## Picking in 30 seconds

- Up to 1 TB of backups → the $6 tier, full stop.
- Growing toward 5 TB → the $20 tier; the per-TB price actually improves as you scale.
- Hard requirement for vendor-enforced immutability → an Object Lock-capable provider instead.
- Archive-only data you'll rarely touch → a cold tier, with compaction off and the minimum-stay rules understood.

Before your first real backup: generate the restricted S3 keys from the Object Storage subscription, never your master API key, and put one test restore on the calendar for week two. 👉 [Compare Vultr's Object Storage tiers before you run your first backup job](https://www.vultr.com/products/object-storage/?ref=7268283).
