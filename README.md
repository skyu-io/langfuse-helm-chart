# Publish Langfuse to Artifact Hub 🚀

This GitHub Action is designed to publish the Langfuse Helm chart to Artifact Hub. It automates the process of packaging the Langfuse chart, updating the `README.md`, and pushing the changes to the repository. The action is triggered manually via the `workflow_dispatch` event.

## Workflow Overview 🛠️

The workflow consists of the following steps:

1. **Checkout the Repository**: The repository is checked out to the local runner.
2. **Check for Existing Langfuse Package**: The action checks if the `langfuse-${{ env.LANGFUSE_VERSION }}.tgz` file already exists. If it does, the workflow is stopped to prevent overwriting the existing package.
3. **Download Langfuse Tar File**: The action downloads the specified version of the Langfuse chart from the GitHub releases.
4. **Extract the Tar File**: The downloaded `.tgz` file is extracted, and the directory structure is checked.
5. **Replace the README File**: The `README.md` file is replaced with the latest version from the repository.
6. **Delete Tar File**: The `.tgz` file is deleted after extracting.
7. **Package the Chart**: The Helm chart is packaged into a `.tgz` file.
8. **Remove Langfuse Directory**: The extracted Langfuse directory is removed after packaging.
9. **Create Index File**: The Helm chart repository index is updated to reflect the new package.
10. **Push Changes to GitHub**: The changes, including the packaged chart, updated `README.md`, and `index.yaml`, are committed and pushed to the repository.

## How to Use 📝

1. **Update the Version**: Change the `LANGFUSE_VERSION` environment variable to the latest release version of the Langfuse repo.
   - Example: Update `LANGFUSE_VERSION: "1.1.0"` to the latest release version.

2. **Update the README File**: 
   - Copy the latest version of the README from the [Langfuse GitHub Repository](https://github.com/langfuse/langfuse-k8s/blob/main/README.md).
   - Paste it into the `./readme/README.md` file in the repository.
   - **Important**: Change the source URL in the README from `https://langfuse.github.io/langfuse-k8s` to `https://skyu-io.github.io/langfuse-helm-chart`.

3. **Run the GitHub Action**: Trigger the action by running the workflow manually using the `workflow_dispatch` event in the GitHub Actions UI.

## Notes ⚠️

- If the package for the specified version already exists, the action will fail to prevent overwriting the existing chart.
- For more information on the Langfuse Helm chart, refer to the [Langfuse GitHub Repository](https://github.com/langfuse/langfuse-k8s).

