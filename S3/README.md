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
For more please visit [docs.aws.amazon.com/AmazonS3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html)