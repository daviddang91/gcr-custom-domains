# Vanity (Custom) Domains for Google Container Registry

This project offers a very simple reverse proxy that lets you expose your
(public or private) Google Container Registries on `gcr.io` as a public registry
on your own domain name.

For example, if you have a public registry, and offering images like:

    docker pull gcr.io/ahmetb-public/busybox

You can use this proxy, and instead offer your images ✨way fancier🎩, like:

    docker pull r.ahmet.dev/busybox

## Building

Download the source code, and build as a container image:

    docker build --tag gcr.io/[YOUR_PROJECT]/gcr-proxy .

Then, push to a registry like:

    docker push gcr.io/[YOUR_PROJECT]/gcr-proxy

## Deploying (to Google Cloud Run)

You can easily deploy this as a serverless container to [Google Cloud Run][run].
This handles many of the heavy-lifting for you.

1. Build and push docker images (previous step)
1. Deploy to [Cloud Run][run].
1. Configure custom domain.
   1. Create domain mapping
   1. Verify domain ownership
   1. Update your DNS records
1. Have fun!

To deploy this to [Cloud Run][run], replace `[PROJECT_ID]` with the project ID
of the GCR registry you want to expose publicly:

```sh
gcloud beta run deploy \
    --allow-unauthenticated \
    --image "[IMAGE]" \
    --set-env-vars "GCR_PROJECT_ID=[PROJECT_ID]"
```

> This will deploy a proxy for your `gcr.io/[PROJECT_ID]` public registry. If
> your GCR registry is private, see the section below on "Exposing private
> registries".

Then create a domain mapping by running (replace the `--domain` value):

```sh
gcloud beta run domain-mappings create \
    --service gcr-proxy \
    --domain reg.ahmet.dev
```

This command will require verifying ownership of your domain name, and have you
set DNS records for your domain to point to [Cloud Run][run]. Then, it will take
some 15-20 minutes to actually provision TLS certificates for your domain name.

### Deploying (elsewhere)

...is much harder. You need to deploy the application to an environment like
Kubernetes, obtain a valid TLS certificate for your domain name, and make it
publicly accessible.

### Exposing private registries publicly

> ⚠️ This will make images in your private GCR registries publicly accessible on
> the internet.

1. Create an [IAM Service
   Account](https://cloud.google.com/iam/docs/creating-managing-service-accounts#creating_a_service_account).

1. [Give it
   permissions](https://cloud.google.com/container-registry/docs/access-control)
   to access the GCR registry GCS Bucket. (Or simply, you can give it the
   project-wide `Storage Object Viewer` role.)

1. Copy your service account JSON key into the root of the repository as
   `key.json`.

1. (Not ideal, but whatever) Rebuild the docker image with your service account
   key JSON in it. This will require editing `Dockerfile` to add `COPY` and
   `ENV` directives like:

       COPY key.json /app/key.json
       ENV GOOGLE_APPLICATION_CREDENTIALS /app/key.json
       ENTRYPOINT [...]

   You need to rebuild and deploy the updated image.

### Advanced Customization

While deploying, you can set additional environment variables for customization:

- **`GCR_HOST`**: defaults to `gcr.io`.
- **`DISABLE_BROWSER_REDIRECTS`**: if you set this variable to any value,
  visiting `example.com/image` on this browser will not redirect to
  `gcr.io/[PROJECT_ID/image` to allow your users to browse the image on GCR. If
  you're exposing private registries, you might want to set this variable.
- **`GOOGLE_APPLICATION_CREDENTIALS`**: path to the IAM service account JSON key
  file to expose the private GCR registries publicly.

-----

This is not an official Google project. See [LICENSE](./LICENSE).

[run]: https://cloud.google.com/run
