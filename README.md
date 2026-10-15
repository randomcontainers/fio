# fio

Container images with [fio](https://github.com/axboe/fio), the Flexible I/O Tester, which runs read and write workloads against files and block devices and reports bandwidth, IOPS and latency. fio is compiled from its release tag on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when fio publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the fio project. Report problems with the image in this repository and fio bugs at [axboe/fio](https://github.com/axboe/fio/issues) or on the fio mailing list.

## Quick start

Measure random 4 KiB reads and writes on a 1 GiB file in the current directory for 60 seconds:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/fio \
  --name=randrw --filename=fio.dat --size=1G --rw=randrw --bs=4k \
  --ioengine=libaio --iodepth=32 --direct=1 --runtime=60 --time_based
```

fio writes `fio.dat` first, then runs the job and prints bandwidth, IOPS and latency percentiles for reads and writes. The file stays in the directory afterwards; pass `--unlink=1` to have fio delete it. The results describe the file system behind the mounted directory, so mount a directory on the disk or volume you want to measure. Without a mount, `/work` is on the container's own file system.

Job files work the same way. With `--output-format=json`, fio writes results that a script can read:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/fio \
  --output-format=json --output=result.json job.fio
```

The [fio documentation](https://fio.readthedocs.io/en/latest/fio_doc.html) describes every option. Running the image without arguments prints fio's command-line options, `--cmdhelp=<option>` describes a job option, and `--enghelp` lists the I/O engines.

## I/O engines

`psync` is the default engine on Linux. `fio --enghelp` lists the others in the image, among them `libaio`, `io_uring`, `posixaio`, `mmap`, `sg`, `mtd` and `net`.

Use `--direct=1` with `libaio`. For buffered I/O, Linux does the work inside the submit call, so `--iodepth` has no effect.

Docker's default seccomp profile has blocked the io_uring system calls since Docker 25.0, so jobs with `--ioengine=io_uring` or `io_uring_cmd` fail to start. Run them with a seccomp profile that allows `io_uring_setup`, `io_uring_enter` and `io_uring_register`, or without seccomp filtering:

```sh
docker run --rm --security-opt seccomp=unconfined --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/fio --name=uring --filename=fio.dat --size=1G --rw=randread --bs=4k \
  --ioengine=io_uring --iodepth=32 --direct=1 --runtime=60 --time_based
```

In Kubernetes, whether the `RuntimeDefault` seccomp profile allows io_uring depends on the container runtime.

The NUMA options `numa_cpu_nodes` and `numa_mem_policy` need the memory policy system calls, which Docker's default seccomp profile allows only with `CAP_SYS_NICE`. Add `--cap-add SYS_NICE` for jobs that use them.

## Block devices

To measure a disk or partition directly, pass it to the container with `--device` and name it in `--filename`. The image runs as UID 1000, which usually cannot open a block device, so run as root:

```sh
docker run --rm --user 0:0 --device /dev/nvme1n1 ghcr.io/randomcontainers/fio \
  --readonly --name=seqread --filename=/dev/nvme1n1 --rw=read --bs=1M \
  --ioengine=libaio --iodepth=16 --direct=1 --runtime=30 --time_based
```

`--group-add "$(stat -c %g /dev/nvme1n1)"` in place of `--user 0:0` gives UID 1000 the device's group, which is enough where that group can read and write it. A write or trim job overwrites what is on the device. `--readonly` makes fio refuse such jobs, so keep it on read tests.

## Client and server

`fio --server` listens on TCP port 8765 and runs the jobs a client sends it, so one client can start the same job on several hosts at once and collect the results. The server has no login. Any client that can reach its port can run any command in the container, as the container's user and with access to the mounted directory: job options such as `exec_prerun` and the `exec` engine run programs, and so does `--trigger-remote`. Publish the port only on an address of a network you trust.

Start a server on each host, with the directory to test mounted at `/work` and the host's address on the trusted network in place of `<address>`:

```sh
docker run -d --name fio-server -p <address>:8765:8765 --user "$(id -u):$(id -g)" -v /mnt/data:/work \
  ghcr.io/randomcontainers/fio --server
```

Then run a job file against it from the client, with the server's name or IP address in place of `<server>`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/fio --client=<server> job.fio
```

Repeat `--client=<server> job.fio` for each server, or pass `--client=hosts.list` with one server per line. The client sends the job file to the server, so every job option must be in it.

## What is in the image

- `fio` in `/usr/local/bin`, with the `libaio` engine, the NUMA options, compressed logs (`log_store_compressed`, read back with `--inflate-log`) and arithmetic in option values, such as `size=(8*1024*1024)`.
- `fio-genzipf`, which shows how a zipf, pareto or normal `random_distribution` spreads I/O over a data set.
- `fio-btrace2fio`, which turns a blktrace capture into a job file with one job per process that imitates its block sizes, read/write mix, randomness, queue depth and rate. fio replays the trace itself with `--read_iolog=<trace>`.
- `fio-dedupe`, which counts duplicate blocks in a file or device.
- `fio-verify-state`, which prints a verify state file.

Not included:

- gfio, the GTK client.
- The scripts that plot or parse fio's logs, which need gnuplot or Python: `fio_generate_plots`, `fio2gnuplot`, `fiologparser.py`, `fiologparser_hist.py`, `fio-histo-log-pctiles.py` and `fio_jsonplus_clat2csv`.
- `genfio`, a bash script that writes job files. The Alpine image has no bash, and the Ubuntu image leaves `genfio` out as well so that both images have the same programs.
- The engines that need libraries the image does not have, such as `http` (S3 and WebDAV), `rbd`, `rdma`, `nfs`, `nbd`, `libiscsi` and `xnvme`.
- The man page.

The configure options and configure's results for the CPU, zlib, libaio, libnuma, MTD, lex/yacc and `-march` checks are in `/usr/local/share/randomcontainers/fio/buildinfo`.

## Default or slim

fio's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a fio release such as `3.43`. `<major>` is its shorter form, `3`, and follows the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current fio version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

fio creates its test files, logs and verify state files in the working directory, so the mounted directory has to be writable for the user the container runs as. In Kubernetes, a volume that belongs to root needs `securityContext.fsGroup`, or a `runAsUser` that owns it.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new fio release and are rebuilt when the base image changes. An image can carry your job files, so a run needs only the volume to test:

```dockerfile
FROM ghcr.io/randomcontainers/fio:slim-ubuntu@sha256:...
COPY jobs/ /usr/local/share/fio-jobs/
```

Runs of that image then take a job by its path, such as `/usr/local/share/fio-jobs/randrw.fio`. To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "fio"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new fio releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/fio/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages fio needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/fio:latest \
  --repo randomcontainers/fio --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/fio:latest --format '{{ json .SBOM }}'
```

fio's GitHub releases carry only Windows installers, so the build clones the `fio-<version>` tag of [axboe/fio](https://github.com/axboe/fio). It stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The tags are signed with PGP, but the build does not check the signatures. The commit is also in `/usr/local/share/randomcontainers/fio/buildinfo`.

## Updates

The project checks the tags of [axboe/fio](https://github.com/axboe/fio) every 15 minutes and follows the 3.x series. A release is picked up once its tag is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t fio:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

fio is licensed under the GNU General Public License, version 2 (GPL-2.0-only), which covers every file that does not state other terms. Some of the files compiled into the image do: the CRC32, CRC32C, CRC-T10DIF, SHA-256, SHA-512 and SHA-3 code, the red-black tree, the FIFO code and the MTD library are GPL-2.0-or-later, xxHash is BSD-2-Clause, and `lib/ieee754.c` is in the public domain. The image's license label is `GPL-2.0-only AND GPL-2.0-or-later AND BSD-2-Clause`. `/usr/local/share/randomcontainers/fio/licenses/` holds fio's `COPYING` and `MORAL-LICENSE` (the author's requests to users, beyond the license) and the notices of xxHash and `lib/ieee754.c` (`NOTICE.xxhash`, `NOTICE.ieee754`). The Ubuntu and Alpine packages in the image, such as libaio, libnuma and zlib, keep their own licenses. The SBOM lists them.

The corresponding source for each image:

- fio: every version has a GitHub release in this repository, named `v<version>`, with `fio-<version>.tar.gz`, an archive of the commit that was compiled. The build applies no patches. `/usr/local/share/randomcontainers/fio/source` names the upstream commit and links to that release.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
