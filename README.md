# `Upload Encrypted Artifact`

[![Test Upload](https://github.com/AlexGidarakos/upload-encrypted-artifact/actions/workflows/test.yml/badge.svg)](https://github.com/AlexGidarakos/upload-encrypted-artifact/actions/workflows/test.yml)

A composite GitHub Action that creates encrypted [artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts). Internally powered by [@actions/upload-artifact](https://github.com/actions/upload-artifact).

- [`Upload Encrypted Artifact`](#upload-encrypted-artifact)
  - [Usage](#usage)
    - [Inputs](#inputs)
    - [Outputs](#outputs)
  - [Examples](#examples)
    - [Upload a single file](#upload-a-single-file)
    - [Upload an entire directory](#upload-an-entire-directory)
    - [Upload using a wildcard pattern](#upload-using-a-wildcard-pattern)
    - [Compression levels explained](#compression-levels-explained)
    - [Reuploading with the same artifact name](#reuploading-with-the-same-artifact-name)
    - [Retention period](#retention-period)
    - [Using outputs](#using-outputs)
      - [Example output between steps](#example-output-between-steps)
      - [Example output between jobs](#example-output-between-jobs)

## Usage

### Inputs

```yaml
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  with:
    # A file, directory or wildcard pattern that describes what to upload
    # Required
    source:

    # A text password from which an AES-256 cipher key is derived and used to encrypt the artifact
    # Required
    password:

    # The level of LZMA compression used when creating the artifact
    # The value can range from 0 (no compression) to 9 (best compression)
    # Values above 4 increase the runtime dramatically with only minimal improvement to the size reduction
    # For large files that are not easily compressed, a value of 1 is recommended for significantly faster execution
    # Optional, default is '3'
    compression-level:

    # Filename of the local artifact before it is uploaded to GitHub
    # Optional, default is 'artifact.tar.7z'
    local-name:

    # Name of the remote artifact in GitHub after the upload
    # Optional, default is 'artifact'
    remote-name:

    # Duration after which artifact will expire in days
    # 0 means using the default value defined in the repository settings
    # Actual values can range from 1 to 90 (public repositories) or 400 (private repositories)
    # Optional, defaults to repository settings
    retention-days:

    # Action behaviour if an artifact with the same name already exists for the same workflow run
    # If true, an artifact with a matching name will be deleted before a new one is uploaded
    # If false, the action will fail if an artifact for the given name already exists
    # Does not fail if the artifact does not exist
    # Optional, default is 'true'
    overwrite:
```

### Outputs

| Name | Description | Example |
| - | - | - |
| `artifact-id` | ID of the artifact, can be used with the GitHub REST API | `1234` |
| `artifact-url` | URL to download the Artifact. Can be used in many scenarios such as linking to artifacts in issues or pull requests. Users must be logged-in in order for this URL to work. This URL is valid as long as the artifact has not expired or the artifact, run or repository have not been deleted | `https://github.com/example-org/example-repo/actions/runs/1/artifacts/1234` |

## Examples

### Upload a single file

```yaml
steps:
  - run: mkdir -p path/to/artifact
  - run: echo "Hello World" > path/to/artifact/hello.txt
  - uses: AlexGidarakos/upload-encrypted-artifact@v1
    with:
      source: path/to/artifact/hello.txt
      password: ${{ secrets.SuperSecret }}
      remote-name: my-artifact
```

### Upload an entire directory

```yaml
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  with:
    source: path/to/artifact/
    password: ${{ secrets.SuperSecret }}
```

### Upload using a wildcard pattern

```yaml
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  with:
    source: path/**/[abc]rtifac?/*
    password: ${{ secrets.SuperSecret }}
```

### Compression levels explained

This action uses the 7z binary to apply LZMA compression to the artifact, with compression levels ranging from 0 (no compression) to 9 (best compression). The default compression level is `3`, but if you are uploading large and highly compressible data, you can try a higher level, e.g. 5.

Values above 4 increase the runtime dramatically with only minimal improvement to the size reduction. For large files that are not very compressible, a value of 0 or 1 is recommended for significantly faster execution.

For instance, if you are uploading random binary data:

```yaml
- name: Create a 1GB file with random bytes
  run: dd if=/dev/urandom of=random.bin bs=1M count=1000
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  with:
    source: random.bin
    password: ${{ secrets.SuperSecret }}
    compression-level: 0  # no compression, very fast
```

### Reuploading with the same artifact name

Once uploaded, artifacts are immutable. Therefore, uploading new data as an artifact with the same name results in deletion of the old artifact and creation of a new artifact with a **different resource ID** in GitHub.

### Retention Period

Artifacts are retained for 90 days by default. You can specify a different retention period using the `retention-days` input:

```yaml
- run: echo "A short-lived artifact" > file.txt
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  with:
    source: file.txt
    password: ${{ secrets.SuperSecret }}
    retention-days: 2
```

A value of 0 means using the default value defined in the repository settings. Actual values can range from 1 to 90 (public repositories) or 400 (private repositories). For more information see [artifact and log retention policies](https://docs.github.com/en/free-pro-team@latest/actions/reference/usage-limits-billing-and-administration#artifact-and-log-retention-policy).

### Using outputs

If an artifact upload is successful, then an `artifact-id` output is available. This ID is a unique identifier that can be used with the  [GitHub REST APIs](https://docs.github.com/en/rest/actions/artifacts).

#### Example output between steps

```yaml
- uses: AlexGidarakos/upload-encrypted-artifact@v1
  id: upload
  with:
    source: path/to/artifact/
    password: ${{ secrets.SuperSecret }}
- name: Output artifact ID
  run: echo 'Artifact ID is ${{ steps.upload.outputs.artifact-id }}'
```

#### Example output between jobs

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      output1: ${{ steps.upload.outputs.artifact-id }}
    steps:
      - uses: AlexGidarakos/upload-encrypted-artifact@v1
        id: upload
        with:
          source: path/to/artifact/
          password: ${{ secrets.SuperSecret }}
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - env:
          OUTPUT1: ${{needs.job1.outputs.output1}}
        run: echo "Artifact ID from previous job is $OUTPUT1"
```
