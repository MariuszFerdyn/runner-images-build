# runner-images-build

Build and publish ready-to-use Linux and Windows runner images (Agents).

The build process uses Azure infrastructure, then stores the resulting VHD in a Storage Account where you can download it and use it anywhere:

- **Azure**: Create VMs/VMSS from the gallery or import the VHD
- **On-Premise**: Download the VHD and deploy to your local virtualization infrastructure
- **AWS**: Import the VHD to create EC2 AMIs
- **GCP**: Import the VHD to create Compute Engine images
- **Workstations**: Use the VHD to build powerful development workstations with all tools preinstalled

These images contain the rich toolset from the official GitHub Actions runner images project, making it easy to host self‑hosted runners for:

- GitHub Actions
- Azure DevOps Pipelines
- Bitbucket Pipelines (self‑hosted)

You get images that already include a broad set of preinstalled software (compilers, SDKs, CLIs, build tools, etc.) so you only add the runner/agent bootstrap you need.

> Source of preinstalled software: the upstream GitHub Actions image build scripts (formerly [`actions/virtual-environments`](https://github.com/actions/virtual-environments), now [`actions/runner-images`](https://github.com/actions/runner-images)). This repo leverages those scripts to create Ubuntu 22.04 and Windows 2025 images.

## What this repository does

The workflows in `.github/workflows/`:

**ubuntu2204.yml** - Ubuntu 22.04 Image:
1. Builds an Ubuntu 22.04 image using the upstream runner image scripts
2. Publishes that image to an Azure Compute Gallery (SIG)
3. Creates an Azure Storage account and a `vhd` container
4. Exports the image as a VHD into that container and prints a temporary SAS link
5. Cleans up the source resource group

**windows2025.yml** - Windows 2025 Image:
1. Builds a Windows 2025 image using the upstream runner image scripts
2. Publishes that image to the same Azure Compute Gallery (SIG)
3. Creates an Azure Storage account and a `vhd` container
4. Exports the image as a VHD into that container and prints a temporary SAS link
5. Cleans up the source resource group

Both workflows share the same gallery but create separate image definitions and use separate storage accounts.

## Outputs you can use

- **Azure Compute Gallery image**: ideal for creating VMs/VMSS at scale with versioning within Azure
- **VHD in Blob Storage**: downloadable and portable - use it to create custom images in Azure, import to AWS/GCP, deploy on-premise, or build development workstations

## Preinstalled software (high level)

Because the build is based on the GitHub Actions runner image, you get a comprehensive toolchain, typically including:

- Common compilers and package managers
- Multiple language runtimes (Node.js, Python, .NET, Java, etc.)
- Docker, Azure CLI, and build tools
- Many CI/CD utilities and SDKs

For the authoritative and always‑up‑to‑date list, see the upstream project: https://github.com/actions/runner-images

## Configure

Set these GitHub repository Secrets and Variables before running the workflows.

Secrets:
- `SUBSCRIPTIONID` – Azure Subscription ID
- `APPLICATIONID` – Service Principal App (Client) ID
- `SECRET` – Service Principal Client Secret
- `TENANTID` – Azure AD Tenant ID

Variables (Repository variables):
- `UbuntuImageResourceGroup` – Resource group for Ubuntu managed image (source)
- `WINDOWSIMAGERESOURCEGROUP` – Resource group for Windows managed image (source)
- `GALERYNAME` – Azure Compute Gallery name (shared target for both images)
- `Region` – Azure region (e.g., `westeurope`)

Notes:
- The workflows will ensure the gallery resource group exists (same name as `GALERYNAME`).
- Each workflow creates its own storage account derived from its respective resource group name (lower‑cased, alphanumeric, <= 24 chars), with a `vhd` container.
- Both workflows share the same gallery but create separate image definitions:
  - Ubuntu: `Runner-Image-Ubuntu2204`
  - Windows: `Runner-Image-Windows2025`

## Run the workflow

1. Go to Actions → Select either "Build Ubuntu 22.04 Image" or "Build Windows 2025 Image" → Run workflow.
2. By default each workflow runs two jobs:
   - `build` (Windows runner): compiles the image using upstream scripts
   - `export` (Linux runner): publishes to SIG and exports a VHD to Blob Storage
3. To run only the export job, comment out the `needs: build` line in the `export` job and trigger the workflow.

Both workflows can run independently and share the same gallery without conflicts.

## Use the image for runners/agents

Choose one of these consumption options:

- From Azure Compute Gallery
  - Create a VM or VM Scale Set from the published gallery image version (Ubuntu or Windows)
  - Add your bootstrap to install and configure the runner/agent on first boot (cloud-init for Linux, Custom Script Extension for Windows)

- From VHD in Blob Storage
  - Use the generated SAS URL (temporary) to copy/import the VHD into your target subscription and create a managed disk and VM
  - Attach a provisioning script to install the GitHub Actions runner, Azure Pipelines agent, or Bitbucket runner at startup

Typical bootstrap (high level):

- GitHub Actions: download runner binary, configure with your repo/org registration token, and set it up as a service
- Azure DevOps: download Azure Pipelines agent, configure with your org/project/pool, and install as a service
- Bitbucket: set up the Bitbucket runner on the VM and register it to your workspace/repo

The images already have the toolchain; you only need the small agent install script appropriate for your platform and OS.## Security considerations

- The workflow deliberately prints sensitive values (SAS URLs, storage account key) for debugging and portability. Secret masking is temporarily disabled around those lines using GitHub Actions `::stop-commands::`. Anyone with access to the workflow logs can see them while valid.
- SAS tokens expire (10 hours for disk SAS, 7 days for VHD SAS in this workflow), but treat logs as sensitive.
- Tighten repository access, shorten SAS validity windows if needed, and consider removing the stop‑commands wrappers once you are done debugging.

## Cleanup

Each export job can optionally delete the source managed image and the entire resource group where it lived. This helps minimize costs once the image has been published to the gallery and exported to the storage account.

- Ubuntu workflow cleans up `UbuntuImageResourceGroup`
- Windows workflow cleans up `WINDOWSIMAGERESOURCEGROUP`

The shared gallery and its images remain intact.

## Troubleshooting

- If the storage account name derived from `UbuntuImageResourceGroup` or `WINDOWSIMAGERESOURCEGROUP` is shorter than 3 characters, the workflow appends `img`.
- If the `export` job is all you need (e.g., re‑publishing), run it alone by removing the `needs: build` dependency.
- Ensure your Service Principal has permissions on both the source and target resource groups.
- Both workflows use idempotent operations for gallery and storage account creation, so they won't fail if resources already exist.

## License

MIT — see `LICENSE`. This repo orchestrates upstream image tooling; please also review the licenses and terms of software included by the upstream runner images project.
