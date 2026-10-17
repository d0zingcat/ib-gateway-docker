# Gateway image publication and deployment

Pushing `master` or manually running **Publish Gateway and deploy** builds both
Gateway channels for `linux/amd64` and `linux/arm64`. TWS is not published by
this workflow. The maintained IBC repository, pinned commit, and TOTP support
come from the channel Dockerfiles.

## Images

Images are published to `ghcr.io/d0zingcat/ib-gateway`:

| Tag | Purpose |
| --- | --- |
| `stable` | Stable channel used by both deployed services |
| `latest` | Latest upstream Gateway channel |
| `stable-sha-<full Git SHA>` | Stable build for a specific repository revision |
| `latest-sha-<full Git SHA>` | Latest build for a specific repository revision |

Each image includes `org.opencontainers.image.revision` and
`org.opencontainers.image.version`. Actions records each published manifest
digest in its run summary. Pin a digest when an immutable reference is needed.

## Dokploy setup

1. Publish the initial images before changing running services. Configure GHCR
   read access in Dokploy if the package is private.
2. Keep the existing paper and live applications, switching their source to
   Docker with image `ghcr.io/d0zingcat/ib-gateway:stable`. Preserve environment
   variables, password mounts, network settings, ports, and runtime settings.
   Remove deployment overrides for Dockerfile-owned version variables so a
   future Gateway upgrade also selects the correct installation directory.
3. Switching the source to Docker stops Git provider builds. Remove any old
   direct Git push hooks for these applications from the repository, and keep
   Dokploy's **Auto Deploy** enabled in Docker mode: this setting also gates
   deployment webhooks. Copy each application's deployment webhook to its own
   GitHub Actions secret: `DOKPLOY_PAPER_DEPLOY_URL` and
   `DOKPLOY_LIVE_DEPLOY_URL`.
4. For a Cloudflare Access protected endpoint, also configure
   `CF_ACCESS_CLIENT_ID` and `CF_ACCESS_CLIENT_SECRET`. Keep webhook URLs and
   credentials out of the repository and logs.
5. Deploy and verify paper first, then live. Preserve the old image and record
   each application's previous configuration for rollback.
6. Set the GitHub Actions repository variable `DOKPLOY_AUTO_DEPLOY=true` after
   completing the migration. Without it the workflow only publishes images.

Once enabled, successful publication of both channels triggers each service's
webhook independently. A publication failure prevents deployment; one webhook
failure does not suppress the other notification. Notifications are not
automatically retried, avoiding duplicate deployments after a lost response.
The workflow is serialized so publication and notifications from different
runs cannot overlap. A webhook success means the deployment was accepted,
not that the Gateway has finished starting or logging in.

## Verification and rollback

- Confirm both architectures in the published GHCR manifest.
- Inspect Dokploy deployment status and the running container image ID.
  Match its image revision and registry digest to the Actions run.
- Verify the expected Gateway version, IBC startup, and internal API listener.
  Do not submit orders to test deployment.
- On deployment failure, set `DOKPLOY_AUTO_DEPLOY=false`, select the previous
  successful `stable-sha-<full Git SHA>` tag or its recorded digest in Dokploy,
  and redeploy. During initial migration, the old local image and saved
  application configuration can also restore the previous service.
