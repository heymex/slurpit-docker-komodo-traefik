# slurpit-docker-komodo-traefik
Compose files to use with Komodo, to deploy Slurpit network discovery behind a Traefik reverse proxy with Cloudflare DNS validation for TLS certs.

I had some struggles getting Slurpit (slurpit.io) working in my preferred Docker-Komodo-Traefik stack.  This is just a simple repo with how I got it working, as well as splitting out credentials etc. into an env file instead of leaving them in the compose file.  Bad developer!  Bad!  No treats for you!

Mainly it was re-focusing on docker instead of podman, env stuff, and fixing some junk with nginx in the portal container not playing nice with the extra layer of reverse proxy from Traefik.

Have fun!  If it doesn't work for you, fix it.  There are enough AI tools out there that you can fix it without being a developer.
