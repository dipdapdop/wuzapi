# Redeployment Instructions

See the building instructions in the README.md directly in the repository, as they may have changed.
- [Here](https://github.com/asternic/wuzapi)

This software is very susceptible to changes in upstream dependencies.
This is not an officially supported product by WhatsApp or Meta, so it may break at any time.
Typically it will break every two months or so around the WhatsApp release cycle.

It makes sense to build the project on the Raspberry Pi 4 itself, as I am guessing there is only limited ability to cross-compile.
However the Pi is an Arm64 architecture, so it might be possible to do it on a Mac (M series) or other Arm64 system.

The build process is quite long, and may take a couple of hours on the Raspberry Pi 4.

## Endpoint for the Admin Web Interface
- http://priscilla.local:9001

## Steps to ensure we have a working system

- SSH into the device (Raspberry Pi 4)
```bash
ssh shaun@priscilla.local
```

- Please ensure that we have a working `cloudflared` tunnel running
  - This is for the CLoudflare Tunnel
    - Cloudflare Dashboard > Nick's account > Zero Trust > Networks > Tunnels`
    - `tunnel name: wagwan-service-tunnel`
      - Check with `docker ps` to see if the `cloudflared` container is running
        - If not, start it with:
          ```bash
          docker run -d cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <check-gasops-secrets>
          ```
- Stop the existing WuzAPI Docker containers
  - See what's running first: 
    - ```bash
      docker ps
      docker-compose down
      ``` 

- Get current with the git repository
  - ensure that you are on the `dev` branch
  - make sure the main branch is synced with the remote repository (upstream)
  - push any changes into your fork (origin) on the remote
  - merge those changes into the `dev` branch with care.

I am pretty sure that we don't need to download the latest WhatsMeow dependency ourselves.
See instructions in the WuzAPI README if it turns out that we do.

The dependencies are managed with `go mod`, so we should be able to just run `docker-compose up --build`, 
and it will download the dependencies automatically, and build them into the container.

(I've preinstalled `go` tooling on the Raspberry Pi 4, so you should be able to use the `go get` package manager
to retrieve the WhatsMeow dependency.)

- Build and start the WuzAPI Docker containers
  - ```bash
    docker-compose up --build -d
    ```
  - Check that the containers are running with `docker ps`
  - Check the logs with `docker logs <container_id>` if there are any issues.

- Check the WuzAPI Web Admin interface
  - http://priscilla.local:9001
  - Login with the WuzAPI WEB_ADMIN_TOKEN from the `.env` file here, or in the shared configuration document.
  - Check that all of the WhatsApp connections are working.
  - If not, you may need to re-scan the QR code with your WhatsApp mobile app.

----

See: 
- [WuzAPI GitHub Repository](https://github.com/asternic/wuzapi)
- [WhatsMeow GitHub Repository](https://github.com/tulir/whatsmeow)
