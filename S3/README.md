# S3 Object Versioning
- Definition (Versioning): S3 Versioning keeps multiple variants (“versions”) of the same object key in a bucket, so you can restore older data if someone overwrites or deletes something.

**Bucket states**

An S3 bucket can be in one of three versioning states:
1. Unversioned (default)
2. Versioning-enabled
3. Versioning-suspended

> Important irreversible property: **once you enable versioning, the bucket can never return to the “unversioned” state (you can only suspend).**

### What happens on PUT/overwrite

#### In a versioning-enabled bucket:
* Uploading an object to an existing key creates a new version; the older one remains stored as a noncurrent version.
* S3 assigns each new version a unique version ID and returns it (e.g., via x-amz-version-id).

#### In a versioning-suspended bucket:
* New uploads typically get the special version ID null (behavior becomes closer to “classic overwrite”), but old versions remain in the bucket. (AWS models this via “Suspended → new objects get version ID null.”)

### What happens on DELETE

> In a versioning-enabled bucket, a normal delete (without specifying a version) does not remove prior data.Instead:
* S3 creates a **delete marker** which becomes the current **version** for that **key**.
* The older versions still exist (they’re just not returned by a normal GET anymore).

#### Resulting behavior:
* GET object (no version specified) returns the “current version.” If the current version is a delete marker, the object appears deleted.
* To “undelete,” you delete the delete marker, which makes the previous version current again.
### “Permanent delete” in a versioned bucket
* To truly remove data, you must delete a specific version (delete by version ID).Versioning exists specifically so “delete” and “overwrite” aren’t immediately destructive.

### Cost and lifecycle reality (what you pay for)
Versioning often increases storage cost because:
* Every overwrite keeps older bytes as noncurrent versions.
* Deletes may add delete markers while keeping data.

### S3 Lifecycle can expire:
* Noncurrent versions via NoncurrentVersionExpiration (permanent removal).
* Delete markers in certain configurations (separate docs cover managing delete markers).

---

# MFA Delete (what it protects, and what it doesn’t)
> Definition (MFA Delete): An S3 bucket setting (stored alongside versioning config) that requires MFA for the most destructive actions:

* Permanently deleting object versions
* Changing the bucket’s versioning state
### Hard constraints (non-negotiable AWS rules)
* Only the bucket owner (root account of the AWS account that created the bucket) can enable MFA Delete.
* You cannot enable MFA Delete in the AWS Console. You must use AWS CLI or the API.
* MFA Delete configuration is set through the same API as versioning (`PutBucketVersioning`) and requires the `x-amz-mfa` header / MFA parameters.

### What MFA Delete actually blocks
With MFA Delete enabled:
* A **user** (even with broad IAM permissions) cannot permanently delete a version unless the request is **authenticated with the root user + MFA**.
* A user also can’t flip versioning settings (Enabled/Suspended) without `root+MFA`.
### What MFA Delete does NOT do
* It does not stop someone from performing a “normal delete” that creates a delete marker (making the object appear gone) unless they were trying to remove specific versions permanently.
* It does not replace proper IAM least privilege, logging, backups, or replication. It’s a narrow control: “**prevent irreversible destruction**.”

### Big operational downside (brutal truth)
AWS explicitly notes: you cannot use MFA Delete with Lifecycle configurations.
That makes MFA Delete awkward for many real systems, because versioned buckets often rely on Lifecycle rules to control storage growth.

### How they work together
If you enable Versioning + MFA Delete:
* Accidental overwrite → older version still exists (recoverable).
* Accidental delete → delete marker created; you can remove the marker to restore.
* Malicious/accidental “wipeout” attempt → permanent deletion of versions requires root + MFA.

---
For more please visit [docs.aws.amazon.com/AmazonS3/versioning-workflows](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html)

---

# How S3 Uploads Work

**How uploads to Amazon S3 work**, including:
- **Single PUT (PutObject)**
- **Multipart Upload**
- **S3 Transfer Acceleration** (and why it can help remote workers uploading large datasets)

---

## What “uploading to S3” means

At the protocol level, an S3 upload is an **authenticated HTTPS request** to an S3 endpoint that includes:

- **Bucket + key** (object name/path)
- **Payload** (the object bytes)
- **Headers** describing the object (content-type, metadata, encryption, storage class, etc.)
- **Auth** using **SigV4** (IAM user/role credentials) or a **pre-signed URL**

The simplest upload API is **PutObject** (“single PUT upload”).  
A **single PutObject** call can upload an object **up to 5 GB**.

When S3 accepts an upload, it stores the object and returns metadata such as:
- **ETag**
- **VersionId** (if bucket versioning is enabled)

---

## Single PUT Upload (PutObject)

### When to use
- Small/medium objects
- Stable connections
- Simple client logic

### How it works
1. Client sends **one** `PUT` request with the full object payload.
2. S3 authenticates/authorizes (IAM + bucket policy checks).
3. S3 stores the object and returns success.

### Key properties
- **Max size per single PUT:** **5 GB**
- If the network fails near the end, you typically must **retry the whole upload**
- Limited ability to **parallelize** throughput

### ETag note (important)
For many single PUT uploads, the **ETag often matches the MD5** of the payload (but do not treat that as guaranteed in all encryption/config scenarios).

---

## Multipart Upload (best for large files)

Multipart upload splits a large object into **parts**, uploads parts independently (often **in parallel**), then tells S3 to assemble them into one final object.

### Why it exists (benefits)
- **Reliability:** retry only the failed part, not the entire file
- **Speed:** parallel uploads can increase throughput significantly
- **Works better over unstable networks**

### Multipart limits you must follow
- **Max parts:** 10,000
- **Part size:** **5 MiB to 5 GiB**
  - **Last part** can be smaller than 5 MiB
- **Max object size:** ~50 TB (AWS commonly documents ~48.8 TiB / 50 TB range)

### Multipart lifecycle (4 main API calls)

#### 1) CreateMultipartUpload
- Starts the multipart upload and returns an **UploadId**
- UploadId identifies the “in-progress upload session”

#### 2) UploadPart (repeat per part)
- Upload parts numbered **1..N** (order doesn’t matter for upload)
- Each successful part returns an **ETag** for that part
- You can upload parts **concurrently**

#### 3) CompleteMultipartUpload
- You send S3 the ordered list of `{PartNumber, ETag}`
- S3 concatenates the parts (in part-number order) into the final object

#### 4) AbortMultipartUpload (important)
- Cancels an in-progress multipart upload
- Prevents ongoing storage charges for uploaded parts (and cleans up)

### Billing reality
Until you **Complete** or **Abort**, uploaded parts may remain stored and can incur charges.
Always abort failed uploads (or use lifecycle rules to clean up incomplete multipart uploads).

### ETag note (important)
For multipart uploads, the final object’s **ETag is NOT the MD5 of the full object**.
It’s derived from the part ETags (a composite value).

---

## S3 Transfer Acceleration

Transfer Acceleration is a bucket feature that lets clients upload to a special **accelerate endpoint**.
Traffic goes:
**Client → nearest AWS edge location (CloudFront) → AWS backbone → S3 bucket region**

This can reduce latency and improve throughput for long-distance uploads.

### Endpoint format
- `bucket-name.s3-accelerate.amazonaws.com`
- (Dualstack variant exists for IPv6)

### Bucket name constraint
- Bucket name must be DNS-compliant and **must not contain periods (`.`)**

---

## Why Transfer Acceleration helps remote workers uploading large datasets

Remote workers often experience:
- unpredictable public internet routing
- congestion and packet loss
- long geographic distance to the bucket’s region

Transfer Acceleration can help by:
- getting the upload onto AWS’s backbone sooner (via nearby edge)
- reducing variability and often improving speed for **large files** and **far distances**

### When it might not help much
- Workers are already close to the S3 region
- Uploads are lots of tiny files (latency dominates)
- Corporate VPN/proxy is the real bottleneck

### Best practice: test before paying
AWS provides a **Transfer Acceleration speed comparison tool** to estimate whether acceleration helps for your users.

---

## Quick decision rules

- **Small files + stable network:** use **Single PUT (PutObject)**
- **Large files (GBs) or unreliable networks:** use **Multipart Upload**
- **Remote/global workforce uploading large objects to one region:** use **Multipart + Transfer Acceleration**, then validate with the speed test tool


