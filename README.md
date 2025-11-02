# runner-images-build

Build and publish a ready-to-use Linux runner image (DevOps Agent base) to Azure. The workflow produces:

- A versioned image in an Azure Compute Gallery (SIG)
- A VHD exported to Azure Storage for direct VM creation or lift-and-shift

This image contains the rich toolset from the official GitHub Actions runner images project, making it easy to host self‑hosted runners for:

- GitHub Actions
- Azure DevOps Pipelines
- Bitbucket Pipelines (self‑hosted)

You get an image that already includes a broad set of preinstalled software (compilers, SDKs, CLIs, build tools, etc.) so you only add the runner/agent bootstrap you need.

> Source of preinstalled software: the upstream GitHub Actions image build scripts (formerly `actions/virtual-environments`, now `actions/runner-images`). This repo leverages those scripts to create an Ubuntu 22.04 image.

## What this repository does

The workflow in `.github/workflows/ubuntu2204.yml`:

1. Builds an Ubuntu 22.04 image using the upstream runner image scripts
2. Publishes that image to an Azure Compute Gallery (SIG)
3. Creates an Azure Storage account and a `vhd` container
4. Exports the image as a VHD into that container and prints a temporary SAS link
5. Optionally cleans up the source resource group

## Outputs you can use

- Azure Compute Gallery image: ideal for creating VMs/VMSS at scale with versioning
- VHD in Blob Storage: usable for custom image creation, on‑prem scenarios, or migration paths

## Preinstalled software (high level)

Because the build is based on the GitHub Actions runner image, you get a comprehensive toolchain, typically including:

- Common compilers and package managers
- Multiple language runtimes (Node.js, Python, .NET, Java, etc.)
- Docker, Azure CLI, and build tools
- Many CI/CD utilities and SDKs

For the authoritative and always‑up‑to‑date list, see the upstream project: https://github.com/actions/runner-images

## Configure

Set these GitHub repository Secrets and Variables before running the workflow.

Secrets:
- `SUBSCRIPTIONID` – Azure Subscription ID
- `APPLICATIONID` – Service Principal App (Client) ID
- `SECRET` – Service Principal Client Secret
- `TENANTID` – Azure AD Tenant ID

Variables (Repository variables):
- `UbuntuImageResourceGroup` – Resource group that holds the managed image (source)
- `GALERYNAME` – Azure Compute Gallery name (target)
- `Region` – Azure region (e.g., `westeurope`)

Notes:
- The workflow will ensure the gallery resource group exists (same name as `GALERYNAME`).
- The storage account name is derived from `UbuntuImageResourceGroup` (lower‑cased, alphanumeric, <= 24 chars), and a `vhd` container is created.

## Run the workflow

1. Go to Actions → “Build Ubuntu 22.04 Image” → Run workflow.
2. By default it runs two jobs:
	 - `build` (Windows): compiles the Ubuntu image using upstream scripts
	 - `export` (Linux): publishes to SIG and exports a VHD to Blob Storage
3. To run only the export job, comment out the `needs: build` line in the `export` job (already prepared in the workflow comments) and trigger the workflow.

## Use the image for runners/agents

Choose one of these consumption options:

- From Azure Compute Gallery
	- Create a VM or VM Scale Set from the published gallery image version
	- Add your bootstrap to install and configure the runner/agent on first boot (cloud-init, Custom Script Extension)

- From VHD in Blob Storage
	- Use the generated SAS URL (temporary) to copy/import the VHD into your target subscription and create a managed disk and VM
	- Attach a provisioning script to install the GitHub Actions runner, Azure Pipelines agent, or Bitbucket runner at startup

Typical bootstrap (high level):

- GitHub Actions: download runner binary, configure with your repo/org registration token, and set it up as a service
- Azure DevOps: download Azure Pipelines agent, configure with your org/project/pool, and install as a service
- Bitbucket: set up the Bitbucket runner on the VM and register it to your workspace/repo

The image already has the toolchain; you only need the small agent install script appropriate for your platform.

## Security considerations

- The workflow deliberately prints sensitive values (SAS URLs, storage account key) for debugging and portability. Secret masking is temporarily disabled around those lines using GitHub Actions `::stop-commands::`. Anyone with access to the workflow logs can see them while valid.
- SAS tokens expire (10 hours for disk SAS, 7 days for VHD SAS in this workflow), but treat logs as sensitive.
- Tighten repository access, shorten SAS validity windows if needed, and consider removing the stop‑commands wrappers once you are done debugging.

## Cleanup

The export job can optionally delete the source managed image and the entire resource group where it lived. This helps minimize costs once the image has been published to the gallery and exported to the storage account.

## Troubleshooting

- If the storage account name derived from `UbuntuImageResourceGroup` is shorter than 3 characters, the workflow appends `img`.
- If the `export` job is all you need (e.g., re‑publishing), run it alone by removing the `needs: build` dependency.
- Ensure your Service Principal has permissions on both the source and target resource groups.

## License

MIT — see `LICENSE`. This repo orchestrates upstream image tooling; please also review the licenses and terms of software included by the upstream runner images project.
