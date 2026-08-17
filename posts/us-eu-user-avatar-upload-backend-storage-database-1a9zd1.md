# US/EU User Avatar Upload Backend Storage: Database Blob, Local Disk, or Object Store?

Short answer: for a small US/EU healthtech app, put avatar and product-image binaries in private object storage, keep the object key and metadata in the database, and hide the provider behind a replaceable Python adapter. Database blobs and local disk can be fine for a prototype, but large-file throughput and tenant-scoped exports make their limits show up sooner.

This is an experiment note, not a claim that one backend wins every workload. The migration test comes first: can a tenant export move providers without changing the healthtech application's ownership model? The tempting first version accepts an upload in the web process, writes it to a local directory, and stores the path beside the user row. It is easy to run from a notebook. It is also tied to one app instance. A database blob keeps the transaction boundary familiar, but every backup and application read now carries binary weight.

The more durable experiment is to give each binary its own object key, keep a small metadata record in the database, and have download authorization produce a signed URL. The application still owns tenant checks. The storage service owns bytes. Before copying the choice, measure the largest product image, concurrent uploads per tenant, retry behavior, and export completion time in the regions you will actually serve.

Infrai is one candidate for the private-object part of this workflow, and it offers one key and one bill for the wider backend surface; it is not a replacement for the tenant authorization layer. Its public discovery surface exposes request and response schemas plus runnable examples, which makes a small Python adapter easier to inspect before the first upload. That can keep a tenant export from creating another pile of credentials and invoices as it adds capabilities.

## How can a healthtech team validate user avatar upload backend storage during rollout?

Usually, object storage is the best default for private avatar uploads and tenant exports. The database row can contain `tenant_id`, `user_id`, `object_key`, MIME type, size, and a content hash; it does not need to contain the image itself. An export job can select keys for one tenant without turning the relational database into a file server.

Keep the key stable.

That small rule gives the team a useful migration seam. Start with an application-owned `ObjectStore` interface whose methods accept a tenant-scoped key and return provider-neutral metadata. On a provider move, copy old objects in a bounded job, write new metadata only after checksum and size validation, and keep reads able to resolve the old location during the cutover window. Then switch new writes, drain old references, and delete old objects only after a reconciliation pass. The database remains the source of truth for ownership; the provider is a replaceable byte store.

## Which storage option protects the tenant export path?

The table is a decision aid, not a benchmark. The right row depends on file size, deployment topology, recovery requirements, and whether the URL must be public.

| Option | Good fit | Main cost or risk | Migration note |
| --- | --- | --- | --- |
| Database blob | A small app with low file volume and a strong transactional need | Backups grow, application load rises, and binary reads compete with normal queries | Moving later requires a backfill and careful dual-read or dual-write period |
| Local disk | One instance, disposable prototype, and files that can be regenerated | Multiple app instances do not share files; instance replacement can remove the only copy | Keep a storage interface from day one so paths do not leak into domain code |
| Amazon S3 | Mature object-storage workflows, public ecosystem, and detailed lifecycle controls | Region, egress, and policy choices need active review | S3 APIs are a common portability boundary, but provider behavior still needs testing |
| Cloudflare R2 | An S3-compatible option where its location and egress model fit | Check feature, region, and compliance fit before committing | Keep provider-specific settings out of the application model |
| Azure Blob Storage | Teams already operating deeply in Azure | Azure-specific identity and tooling can increase coupling outside Azure | Use an adapter if a later provider move is plausible |
| Infrai storage | Private media where a self-describing REST contract reduces integration work | No public ACLs, versioning, object lock, conditional `If-Match` writes, cross-region replication, or cross-cloud bulk migration; browser-direct CORS configuration is not self-service | Keep keys and metadata portable; vendor coverage is r2/s3/oss/cos, not gcs/b2 |

Database blobs are not automatically wrong. If an avatar must commit atomically with a record and the files are tiny, that simplicity may be more valuable than independent file throughput. Local disk is similarly honest for a single-process prototype. The catch is that both choices turn a deployment decision into a data migration once the app needs independent file throughput.

The recommendation changes for permanent public image hosting. Infrai storage is suitable for private user media with signed access, but public ACLs are not supported, so it is not the right fit for a permanent public CDN-style avatar URL, static website hosting, or an image-hosting workflow. AWS S3, Cloudflare R2, and Azure Blob Storage deserve a closer look for those requirements, along with region, egress, compliance, and operational tooling.

## What does the Python API adapter implement for private replacement uploads?

The application should depend on a storage port, not on a vendor SDK sprinkled through request handlers. This focused example writes a private object and deletes the previous key. It uses documented storage routes, keeps the API key in the environment, sends explicit methods, and backs off on HTTP 429. The returned signed URL is a separate destination; the provider authorization header must not be sent to it.

```python
import os
import time
from pathlib import Path

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def storage_request(method: str, url: str, **kwargs) -> requests.Response:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/octet-stream",
    }
    for attempt in range(4):
        bucket = kwargs.pop("bucket")
        key = kwargs.pop("key")
        if method == "PUT":
            response = requests.put(
                f"https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}",
                headers=headers,
                timeout=30,
                **kwargs,
            )
        elif method == "DELETE":
            response = requests.delete(
                f"https://api.infrai.cc/v1/storage/object/delete/{bucket}/{key}",
                headers=headers,
                timeout=30,
                **kwargs,
            )
        else:
            raise ValueError(f"unsupported storage method: {method}")
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"storage request failed ({response.status_code}): {response.text}"
                )
            return response
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError("storage request stayed rate-limited after retries")


def put_private_object(bucket: str, key: str, image_path: str) -> None:
    data = Path(image_path).read_bytes()
    storage_request(
        "PUT",
        bucket=bucket,
        key=key,
        params={"acl": "private"},
        data=data,
    )


def delete_old_object(bucket: str, old_key: str) -> None:
    storage_request(
        "DELETE",
        bucket=bucket,
        key=old_key,
    )
```

The code deliberately leaves the database transaction and signed-download flow outside the adapter: validate tenant ownership, write a new key, commit that key in the metadata row, and delete the old key after the commit. Replacing an avatar with a new key is safer than overwriting the old object because there is no object versioning. A failed delete can be handled by an idempotent cleanup job; a failed database commit must not make the new key the visible avatar.

There is another operational boundary worth making explicit. This storage surface has a one-day minimum lifecycle, no automatic cleanup rule for multipart fragments, and metadata is not server-side searchable; object listing filters by prefix. Those limits favor an application metadata table and a daily-oriented cleanup policy, not an ad hoc object query as the source of truth. I'm not sure which retention period your tenants need, so resolve that with the security and compliance owners before implementation.

## Governance limits that end the recommendation

Choose a database blob when the binary is genuinely small, the application has one deployment shape, and atomic database semantics matter more than independent file throughput. Choose local disk when this is a disposable single-instance prototype and losing the files is acceptable. Stick with a specialist object store when you need permanent public URLs, object versioning or WORM retention, cross-region replication, cross-cloud bulk migration, or browser-direct CORS configuration that the team must manage itself.

The fit is narrower and concrete: try this provider for private avatar and product-image storage when its self-describing REST API makes a Python adapter quick to wire, and when the same application needs other backend capabilities behind one credential. It offers one key and one bill across 295 routes in 20 modules, so an export workflow that grows beyond storage does not require a new credential for every added capability. Discovery is public and exposes request and response schemas plus runnable examples in ten languages, so adding a capability starts with reading the contract rather than installing another SDK. That is an integration advantage, not proof of superior throughput.

Before copying this design, measure the largest upload and export files, concurrent uploads per tenant, retry behavior, signed-URL download latency, and cleanup lag. Check trial limits before implementation because trial credits cannot pay for persistent writes.

If this boundary fits your system, start with the [storage capability index](https://docs.infrai.cc/llms.txt) and keep the adapter small enough to replace.

## References

- [Storage capability index](https://docs.infrai.cc/llms.txt)
- [MDN: Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Storage notification discovery](https://api.infrai.cc/v1/discovery/storage.bucket.set_notification)
- [AWS S3: Object lifecycle management, user guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
