# ESPHome-NonRoot

[ESPHome][esphome-link] docker container that supports non-root operation.

## License

Licensed under the [MIT License][license-link]  
![GitHub License][license-shield]

## Build

Code and Pipeline is on [GitHub][github-link].  
Docker images are published on [Docker Hub][docker-link].

## Status

[![Last Commit][last-commit-shield]][commit-link]  
[![Workflow Status][workflow-status-shield]][actions-link]  
[![Last Build][last-build-shield]][actions-link]  
[![Docker Latest][docker-latest-version-shield]][docker-link]

## Release Notes

- Version 1.0:
  - Initial public release.

## Usage

### Configuration

- `volumes` :
  - `/config` : Volume mapping to project files, e.g. `/data/esphome/config:/config`.
  - `/cache` (Optional) : Volume mapping to runtime generated content, e.g. `/data/esphome/cache:/cache`.
    - Omitting the `/cache` volume will create an unnamed docker volume.
- `user` (Optional) : Run the container under the specified user account.
  - Use the `uid:gid` notation, e.g. `user: 1001:100`.
    - Get the `uid` : `sudo id -u nonroot`.
    - Get the `gid` : `sudo id -g nonroot`.
  - Use an existing user or create a system account.
    - `adduser --no-create-home --shell /bin/false --disabled-password --system --group users nonroot`.
  - Omitting the `user` option will run under default `root` account.
  - Make sure the container user has permissions to the mapped volumes.
    - `sudo chown -R nonroot:users /data/esphome`
    - `sudo chmod -R ugo=rwx /data/esphome`
- `ESPHOME_VERBOSE` : By setting the `ESPHOME_VERBOSE` environment variable ESPHome will be launched with the [`--verbose`](https://esphome.io/guides/cli.html#cmdoption-v-verbose) option.

### Docker Compose

```yaml
volumes:
  esphome_cache:

services:
  esphome:
    image: docker.io/ptr727/esphome-nonroot:latest
    container_name: esphome-nonroot-test
    restart: unless-stopped
    user: 1001:100
    environment:
      - TZ=Americas/Los_Angeles
      - ESPHOME_VERBOSE=true
    network_mode: bridge
    ports:
      - 6052:6052
    volumes:
      - /data/esphome/config:/config
      - esphome_cache:/cache
```

```console
# Launch stack
docker compose --file ./Docker/Compose.yml up --detach
# Open browser http://localhost:6052
# destroy stack
docker compose --file ./Docker/Compose.yml down --volumes
```

### Docker Run

Run ESPHome with custom commandline arguments:  
`docker run ptr727/esphome-nonroot:latest esphome version`

Run a shell in the container:  
`docker run -it --rm --pull always --name Testing ptr727/esphome-nonroot:latest /bin/bash`

## Use Case

- Running containers as non-privileged and as non-root is a docker best practice.
- The official ESPHome [docker container][esphome-docker-link] does not support running as a non-root user.
  - [Issue #3558 : Docker requires root](https://github.com/esphome/issues/issues/3558).
  - [Issue #2752 : Docker image does not allow running rootless](https://github.com/esphome/issues/issues/2752).
  - [Issue #3929 : Not possible to run docker esphome/esphome container - problem with platformio](https://github.com/esphome/issues/issues/3929).
  - [HA Community : Is there a way to run ESPhome in docker with custom UID and GID](https://community.home-assistant.io/t/is-there-a-way-to-run-esphome-in-docker-with-custom-uid-and-gid/668256).
  - Etc.
- This project creates a container that supports the docker `user` option allowing the container to run non-privileged and as non-root.

## Issue Description

- Primary issues is the container environment relies on the executing user being `root` and having `root` container filesystem permissions.
- Refer to the ESPHome `2024.5.5` (current version as of writing) [`Dockerfile`](https://github.com/esphome/esphome/blob/2024.5.5/docker/Dockerfile).
- [`PLATFORMIO_GLOBALLIB_DIR=/piolibs`](https://github.com/esphome/esphome/blob/2024.5.5/docker/Dockerfile#L67) sets the PlatformIO [`globallib_dir`](https://docs.platformio.org/en/latest/envvars.html#envvar-PLATFORMIO_GLOBALLIB_DIR) option to `/piolibs`.
  - `/piolibs` is not mapped to an external volume.
  - `/piolibs` has default permissions and requires `root` write permissions at runtime.
  - The [`globallib_dir`](https://docs.platformio.org/en/latest/projectconf/sections/platformio/options/directory/globallib_dir.html#projectconf-pio-globallib-dir) option has been deprecated.
    - `This option is DEPRECATED. We do not recommend using global libraries for new projects. Please use a declarative approach for the safety-critical embedded development and declare project dependencies using the lib_deps option.`
- [`platformio_install_deps.py`](https://github.com/esphome/esphome/blob/2024.5.5/docker/Dockerfile#L101) installs global libraries using [`pio pkg install -g`](https://github.com/esphome/esphome/blob/2024.5.5/script/platformio_install_deps.py#L58), the [`-g`](https://docs.platformio.org/en/stable/core/userguide/pkg/cmd_install.html#cmdoption-pio-pkg-install-g) option has been deprecated.
  - `We DO NOT recommend installing libraries in the global storage. Please use the lib_deps option and declare library dependencies per project.`
- The [presence](https://github.com/esphome/esphome/blob/2024.5.5/docker/docker_entrypoint.sh#L6) of a `/cache` directory changes `pio_cache_base` to `/cache/platformio`, the default is `/config/.esphome/platformio`
  - `PLATFORMIO_PLATFORMS_DIR="${pio_cache_base}/platforms"`, `PLATFORMIO_PACKAGES_DIR="${pio_cache_base}/packages"`, and `PLATFORMIO_CACHE_DIR="${pio_cache_base}/cache"` is explicitly set as child directories of `pio_cache_base`.
  - It is simpler to set `PLATFORMIO_CORE_DIR` PlatformIO [`core_dir`](https://docs.platformio.org/en/latest/envvars.html#envvar-PLATFORMIO_CORE_DIR) option, and not setting `PLATFORMIO_PLATFORMS_DIR` [`platforms_dir`](https://docs.platformio.org/en/latest/projectconf/sections/platformio/options/directory/platforms_dir.html#projectconf-pio-platforms-dir), `PLATFORMIO_PACKAGES_DIR` [`packages_dir`](https://docs.platformio.org/en/latest/projectconf/sections/platformio/options/directory/packages_dir.html#projectconf-pio-packages-dir), and `PLATFORMIO_CACHE_DIR` [`cache_dir`](https://docs.platformio.org/en/latest/projectconf/sections/platformio/options/directory/cache_dir.html#projectconf-pio-cache-dir) options, that are by default child directories of `core_dir`.
- The [presence](https://github.com/esphome/esphome/blob/2024.5.5/docker/docker_entrypoint.sh#L26) of a `/build` directory sets the `ESPHOME_BUILD_PATH` environment variable, that [sets](https://github.com/esphome/esphome/blob/2024.5.5/esphome/core/config.py#L204) the `CONF_BUILD_PATH` ESPHome [`build_path`](https://esphome.io/components/esphome.html) option, the default is `/config/.esphome/build`.
  - The directory presence detection could override an explicitly set `ESPHOME_BUILD_PATH` option.
- `ESPHOME_DATA_DIR` can be used to set the ESPHome [`data_dir`](https://github.com/esphome/esphome/blob/2024.5.5/esphome/core/__init__.py#L595) intermediate build output directory, the [default](https://github.com/esphome/esphome/blob/2024.5.5/esphome/core/__init__.py#L599) is `/config/.esphome`, or hardcoded to `/data` for the HA addon image.
- [`PLATFORMIO_CORE_DIR`](https://docs.platformio.org/en/latest/envvars.html#envvar-PLATFORMIO_CORE_DIR) PlatformIO [`core_dir`](https://docs.platformio.org/en/latest/projectconf/sections/platformio/options/directory/core_dir.html#projectconf-pio-core-dir) option is not set and defaults to `~/.platformio`.
- [`PIP_CACHE_DIR`](https://pip.pypa.io/en/stable/topics/caching/#pip-cache-dir) is not set and defaults to `~/.cache/pip`.
- `HOME` (`~`) is not set and defaults to e.g. `/home/[username]` or `/` or `/nonexistent` that either does not exists or the executing user does not have write permissions.

## Project Design

- Use the `python:slim` Debian based Python docker image as a base image simplifying use for Python in a container environment.
- Use a multi-stage build minimizing size and layer complexity of the final stage.
- Build [wheel](https://pip.pypa.io/en/stable/cli/pip_wheel/) archives for the platform in the builder stage, and install from the generated wheel packages in the final stage.
- Set appropriate PlatformIO and ESPHome environment variables to store projects in `/config` and dynamic content in `/cache` volumes.

[actions-link]: https://github.com/ptr727/ESPHome-NonRoot/actions
[commit-link]: https://github.com/ptr727/ESPHome-NonRoot/commits/main
[docker-latest-version-shield]: https://img.shields.io/docker/v/ptr727/esphome-nonroot/latest?label=Docker%20Latest&logo=docker
[docker-link]: https://hub.docker.com/r/ptr727/esphome-nonroot
[workflow-status-shield]: https://img.shields.io/github/actions/workflow/status/ptr727/ESPHome-NonRoot/BuildDockerPush.yml?logo=github&label=Workflow%20Status
[github-link]: https://github.com/ptr727/ESPHome-NonRoot
[last-build-shield]: https://byob.yarr.is/ptr727/ESPHome-NonRoot/lastbuild
[last-commit-shield]: https://img.shields.io/github/last-commit/ptr727/ESPHome-NonRoot?logo=github&label=Last%20Commit
[license-link]: ./LICENSE
[license-shield]: https://img.shields.io/github/license/ptr727/ESPHome-NonRoot?label=License
[esphome-link]: https://esphome.io
[esphome-docker-link]: https://hub.docker.com/r/esphome/esphome
