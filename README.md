# OpenWrt GitHub Actions Shared Workflows

This repository contains shared reusable GitHub Actions workflows to compile OpenWrt tools, toolchain, and package feeds, optimized for macOS runners and standard environments.

---

## S3-Compatible Storage Configuration (e.g. Cloudflare R2, MinIO, AWS S3)

To share the compiled build tools and toolchain outputs between different workflow runs (reducing compile times significantly), the workflow integrates with an S3-compatible object storage provider.

The workflows expect the following secrets to be defined in your GitHub repository/organization:

### Configuration Secrets

| Secret Key | Description | Format / Example |
| :--- | :--- | :--- |
| `s3_endpoint` | The base URL of the S3-compatible service. | **Correct:** `https://<account-id>.r2.cloudflarestorage.com`<br>**Avoid:** `https://<account-id>.r2.cloudflarestorage.com/my-bucket/` *(path components/trailing slashes are auto-stripped but should be avoided)* |
| `s3_bucket` | The name of the S3 bucket to save artifacts/caches into. | `openwrt-cache-bucket` |
| `s3_access_key` | Access key ID for S3 client authorization. | `abc123xyz...` |
| `s3_secret_key` | Secret access key for S3 client authorization. | `secret456key...` |
| `s3_public_url` | *(Optional)* Public read-only URL/CDN pointing to your bucket. Used to speed up toolchain restoration/download via HTTP `curl`. | `https://cdn.my-domain.com` or `https://pub-abc.r2.dev` |

---

### Secret Format Guidance

#### 1. Endpoint URL Format (`s3_endpoint`)
The MinIO Client (`mc`) used in the workflow requires the endpoint URL to be **without resource components** (i.e. no path suffixes or trailing bucket names).
* **Do:** Use `https://<account-id>.r2.cloudflarestorage.com` or `https://s3.us-east-1.amazonaws.com`.
* **Don't:** Include the bucket name in the URL (e.g. `https://<account-id>.r2.cloudflarestorage.com/my-bucket`).
* *Note: The workflow includes automated sanitization regexes to strip trailing paths or slashes, but configuring it cleanly prevents configuration conflicts.*

#### 2. Public Read-Only URL (`s3_public_url`)
If you route downloads through a CDN or a public domain to avoid API request caps, configure `s3_public_url`. When set, the restore step downloads cache archives from:
* `${S3_PUBLIC_URL}/${TOOLS_TAR}` (instead of query-authorized S3 downloads).
* If unset, it falls back to standard S3 endpoint URLs.
