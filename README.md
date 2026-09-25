# Tor Browser - Docker Image Based on Debian

This Docker image provides a secure, isolated environment to run the <a href="https://www.torproject.org/" target="_blank">Tor Browser</a> _(opens in new tab)_.  
Tor Browser is designed to help protect your privacy online by anonymizing your web traffic and shielding your identity from network surveillance and traffic analysis. This image is built on top of **"Debian:bookworm-slim"** base image, ensuring a stable and secure environment.

## Key Features:

- **Tor Browser**: Pre-installed and configured to run over the Tor network for anonymous browsing.
- **Debian Base**: Built on top of Debian bookworm-slim image.
- **Isolated Environment**: Ensures that the Tor Browser runs in a sandboxed environment.

---

**⚠️ To run this script succesfuly - need to use VPN or Tor**

[?] Why? - If scripts try to pull Tor Browser directly via ISP (clear net) and do NOT use VPN or so - might have an error, as at the time this project was created, Tor Project was refusing users pulling browser via clear net!

---

## Requirements:

- **Podman** (recommended) - In the .sh file below `podman` is used instead of `docker`.    
- **Docker** (compatible) - Ensure to replace `podman` with `docker` in the .sh file if you want to use it instead.  
- **X11 Server** -- You need to have an **X11** server running for the graphical interface.  

⚠️ _It does not work on **Wayland !**_

>_(to check what you using - run: `echo $XDG_SESSION_TYPE` on Linux)_ 
- Use VPN or Tor connection.
> _On clear net will not be possible to pull/download Tor-browser, as the Tor project does not allow it !_  

## Usage:

👣 Create a folder **"tor-browser-debian"**   
👣 Create two files in the same folder: **"Dockerfile"** & **"run-tor-browser.sh"** with the code below...

---------------

#### Dockerfile:

```
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    libasound2 \
    libdbus-glib-1-2 \
    libgtk-3-0 \
    libx11-xcb1 \
    libxt6 \
    libpci3 \
    xz-utils \
    ffmpeg \
    pulseaudio \
    apt-utils \
    curl \
    libgl1-mesa-glx \
    libegl1-mesa \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /home/torbrowser

RUN LATEST_VERSION=$(curl -sSL https://www.torproject.org/dist/torbrowser/ | \
    grep -oP 'href="([0-9]+\.[0-9]+\.[0-9]+)/"' | \
    grep -oP '[0-9]+\.[0-9]+\.[0-9]+' | sort -V | tail -n 1) && \
    echo "Latest Tor Browser version: $LATEST_VERSION" && \
    curl -sSL -o /home/torbrowser/tor.tar.xz \
      https://www.torproject.org/dist/torbrowser/${LATEST_VERSION}/tor-browser-linux-x86_64-${LATEST_VERSION}.tar.xz && \
    tar xJf /home/torbrowser/tor.tar.xz && \
    rm -f /home/torbrowser/tor.tar.xz

RUN useradd -ms /bin/bash torbrowser
RUN chown -R torbrowser:torbrowser /home/torbrowser

USER torbrowser
WORKDIR /home/torbrowser/tor-browser

ENTRYPOINT ["./start-tor-browser.desktop", "--verbose"]
```

------------------

#### run-tor-browser.sh:

```
#!/bin/bash
set -e

cd "$(dirname "$0")"

IMAGE="localhost/dockerfos/tor-browser-debian:latest"

# Build the Podman image if it doesn't exist
if ! podman image exists "$IMAGE"; then
    echo "Building Podman image for Tor Browser..."
    podman build -t "$IMAGE" .
else
    echo "Image already exists, skipping build."
fi

# Grant X11 access
xhost +local:$(id -un)

# PulseAudio paths
PULSE_SERVER="unix:/run/user/$(id -u)/pulse/native"
PULSE_SOCKET="/run/user/$(id -u)/pulse/native"
PULSE_COOKIE="$HOME/.config/pulse/cookie"

# Run Tor Browser container
echo "Running Tor Browser inside Podman..."
podman run -it --rm \
    --network="slirp4netns:allow_host_loopback=false" \
    -e DISPLAY="$DISPLAY" \
    -e PULSE_SERVER="$PULSE_SERVER" \
    -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
    -v "$PULSE_SOCKET:$PULSE_SOCKET:rw" \
    -v "$PULSE_COOKIE:/home/torbrowser/.config/pulse/cookie:ro" \
    "$IMAGE"

# Revoke X11 access after container exits
echo "Revoking Podman access to X11 server..."
xhost -local:$(id -un)
```

--------------

👣 Make **"run-tor-browser.sh"** executable:

```
chmod +x run-tor-browser.sh
```

👣 No need to build the image - just run the script:

```
./run-tor-browser.sh
```

---------


## Security Considerations:

- This container is designed to run in a sandboxed environment with minimal exposure to the host system.

- It's recommended to run this image using non-privileged user permissions and limit access to sensitive directories.

## Disclaimer:

This Docker image is provided **as-is**, without any warranties or guarantees of any kind. Use of this image is **entirely at your own risk**.

The developer assumes **no responsibility** for:

- Any issues, damage, or data loss resulting from the use of this image.
- Legal implications, security vulnerabilities, or other risks related to its usage, including but not limited to the use of the Tor network in your jurisdiction.
- Compliance with local laws, network restrictions, or any regulatory requirements.

By using this image, you agree that the developer will not be held liable for any consequences or actions that arise from its usage. Please ensure you have reviewed all relevant legal considerations and understand the potential risks involved before deploying or using this image.
