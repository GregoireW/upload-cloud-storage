# upload-cloud-storage

The `upload-cloud-storage` GitHub Action uploads files to a [Google Cloud
Storage (GCS)][gcs] bucket.

Paths to files that are successfully uploaded are set as output variables and
can be used in subsequent steps.

**This is not an officially supported Google product, and it is not covered by a
Google Cloud support contract. To report bugs or request features in a Google
Cloud product, please contact [Google Cloud
support](https://cloud.google.com/support).**

## Prerequisites

-   This action requires Google Cloud credentials that are authorized to upload
    blobs to the specified bucket. See the [Authorization](#authorization)
    section below for more information.

-   This action runs using Node 16. If you are using self-hosted GitHub Actions
    runners, you must use runner version [2.285.0](https://github.com/actions/virtual-environments)
    or newer.

## Usage

> **⚠️ WARNING!** The Node.js runtime has [known issues with unicode characters
> in filepaths on Windows][nodejs-unicode-windows]. There is nothing we can do
> to fix this issue in our GitHub Action. If you use unicode or special
> characters in your filenames, please use `gsutil` or `gcloud` to upload
> instead.

### For uploading a file

```yaml
jobs:
  job_id:
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
    - id: 'checkout'
      uses: 'actions/checkout@v4'

    - id: 'auth'
      uses: 'google-github-actions/auth@v2'
      with:
        workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
        service_account: 'my-service-account@my-project.iam.gserviceaccount.com'

    - id: 'upload-file'
      uses: 'google-github-actions/upload-cloud-storage@v2'
      with:
        path: '/path/to/file'
        destination: 'bucket-name/file'

    # Example of using the output
    - id: 'uploaded-files'
      uses: 'foo/bar@v1'
      env:
        file: '${{ steps.upload-file.outputs.uploaded }}'
```

The file will be uploaded to `gs://bucket-name/file`

### For uploading a folder

```yaml
jobs:
  job_id:
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
    - id: 'checkout'
      uses: 'actions/checkout@v4'

    - id: 'auth'
      uses: 'google-github-actions/auth@v2'
      with:
        workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
        service_account: 'my-service-account@my-project.iam.gserviceaccount.com'

    - id: 'upload-folder'
      uses: 'google-github-actions/upload-cloud-storage@v2'
      with:
        path: '/path/to/folder'
        destination: 'bucket-name'

    # Example of using the output
    - id: 'uploaded-files'
      uses: 'foo/bar@v1'
      env:
        files: '${{ steps.upload-folder.outputs.uploaded }}'
```

## Destination Filenames

If the folder has the following structure:

```text
.
└── myfolder
    ├── file1
    └── folder2
        └── file2.txt
```

### Default Configuration

With default configuration

```yaml
- id: 'upload-files'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: 'myfolder'
    destination: 'bucket-name'
```

The files will be uploaded to `gs://bucket-name/myfolder/file1`,`gs://bucket-name/myfolder/folder2/file2.txt`

Optionally, you can also specify a prefix in destination.

```yaml
- id: 'upload-files'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: 'myfolder'
    destination: 'bucket-name/myprefix'
```

The files will be uploaded to `gs://bucket-name/myprefix/myfolder/file1`,`gs://bucket-name/myprefix/myfolder/folder2/file2.txt`

### Upload to bucket root

To upload `myfolder` to the root of the bucket, you can set `parent` to false.
Setting `parent` to false will omit `path` when uploading to bucket.

```yaml
- id: 'upload-files'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: 'myfolder'
    destination: 'bucket-name'
    parent: false
```

The files will be uploaded to `gs://bucket-name/file1`,`gs://bucket-name/folder2/file2.txt`

If path was set to `myfolder/folder2`, the file will be uploaded to `gs://bucket-name/file2.txt`

Optionally, you can also specify a prefix in destination.

```yaml
- id: 'upload-files'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: 'myfolder'
    destination: 'bucket-name/myprefix'
    parent: false
```

The files will be uploaded to `gs://bucket-name/myprefix/file1`,`gs://bucket-name/myprefix/folder2/file2.txt`

### Glob Pattern

You can specify a glob pattern like

```yaml
- id: 'upload-files'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: 'myfolder'
    destination: 'bucket-name'
    glob: '**/*.txt'
```

This will particular pattern will match all text files within `myfolder`.

In this case, `myfolder/folder2/file2.txt` is the only matched file and will be uploaded to `gs://bucket-name/myfolder/folder2/file2.txt`.

If `parent` is set to `false`, it wil be uploaded to `gs://bucket-name/folder2/file2.txt`.

## Inputs

-   `path` - (Required) The path to a file or folder inside the action's
    filesystem that should be uploaded to the bucket.

    You can specify either the absolute path or the relative path from the
    action:

    ```yaml
    path: /path/to/file
    ```

    ```yaml
    path: ../path/to/file
    ```

-   `destination` - (Required) The destination for the file/folder in the form
    bucket-name or with an optional prefix in the form bucket-name/prefix

    ```yaml
    destination: bucket-name
    ```

    In the above example, the file will be uploaded to gs://bucket-name/file

    ```yaml
    destination: bucket-name/prefix
    ```

    In the above example, the file will be uploaded to
    gs://bucket-name/prefix/file

-   `gzip` - (Optional) Upload file(s) with gzip content encoding, defaults to
    true.

    ```yaml
    gzip: false
    ```

    In the above example, the file(s) will be uploaded without `gzip`
    content-encoding

-   `resumable` - (Optional) Enable resumable uploads, defaults to true.

    ```yaml
    resumable: false
    ```

-   `predefinedAcl` - (Optional) Apply a predefined set of access controls to
    the file(s).

    ```yaml
    predefinedAcl: projectPrivate
    ```

    In the above example, project team members get access to the uploaded
    file(s) according to their roles.

    Acceptable values are: `authenticatedRead`, `bucketOwnerFullControl`,
    `bucketOwnerRead`, `private`, `projectPrivate`, `publicRead`. See [the
    document](https://googleapis.dev/nodejs/storage/latest/global.html#UploadOptions)
    for details.

-   `headers` - (Optional) Set object metadata.

    ```yaml
    headers: |-
      content-type: application/json
      x-goog-meta-custom-field: custom-value
    ```

    In the above example, file `Content-Type` will be set to `application/json`
    and custom metadata with key `custom-field` and value `custom-value` will be
    added to it.

    Settable fields are: `cache-control`, `content-disposition`,
    `content-encoding`, `content-language`, `content-type`, `custom-time`. See
    [the
    document](https://cloud.google.com/storage/docs/gsutil/addlhelp/WorkingWithObjectMetadata#settable-fields;-field-values)
    for details.

    All custom metadata fields must be prefixed with `x-goog-meta-`.

-   `parent` - (Optional) Whether parent dir should be included in GCS
    destination, defaults to true.

    ```yaml
    parent: false
    ```

-   `glob` - (Optional) Glob pattern.

    ```yaml
    glob: '*.txt'
    ```

-   `concurrency` - (Optional) Number of files to simultaneously upload,
    defaults to 100.

    ```yaml
    concurrency: 10
    ```

-   `process_gcloudignore` - (Optional) Process a `.gcloudignore` file present
    in the top-level of the repository. If true, the file is parsed and any
    filepaths that match are not uploaded to the storage bucket. Defaults to
    true.

    ```yaml
    process_gcloudignore: true
    ```

-   `project_id` - (Optional) Google Cloud project ID to use for billing and API
    requests. By default, this is extracted from the running environment.

    ```yaml
    project_id: 'my-project'
    ```

## Outputs

List of successfully uploaded file(s).

For example:

```yaml
- id: 'upload-file'
  uses: 'google-github-actions/upload-cloud-storage@v2'
  with:
    path: '/path/to/file'
    destination: 'bucket-name/file'
```

will be available in future steps as the output "uploaded":

```yaml
- id: 'publish'
  uses: 'foo/bar@v1'
  env:
    file: '${{ steps.upload-file.outputs.uploaded }}'
```

## Authorization

There are a few ways to authenticate this action. The caller must have
permissions to access the secrets being requested.

### Via google-github-actions/auth

Use [google-github-actions/auth](https://github.com/google-github-actions/auth)
to authenticate the action. You can use [Workload Identity Federation][wif] or
traditional [Service Account Key JSON][sa] authentication.

```yaml
jobs:
  job_id:
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
    - id: 'auth'
      uses: 'google-github-actions/auth@v2'
      with:
        workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
        service_account: 'my-service-account@my-project.iam.gserviceaccount.com'

    - uses: 'google-github-actions/upload-cloud-storage@v2'
```

### Via Application Default Credentials

If you are hosting your own runners, **and** those runners are on Google Cloud,
you can leverage the Application Default Credentials of the instance. This will
authenticate requests as the service account attached to the instance. **This
only works using a custom runner hosted on GCP.**

```yaml
jobs:
  job_id:
    steps:
    - id: 'upload-file'
      uses: 'google-github-actions/upload-cloud-storage@v2'
```

The action will automatically detect and use the Application Default
Credentials.

[gcs]: https://cloud.google.com/storage
[wif]: https://cloud.google.com/iam/docs/workload-identity-federation
[sa]: https://cloud.google.com/iam/docs/creating-managing-service-accounts
[nodejs-unicode-windows]: https://github.com/nodejs/node/issues/48673
