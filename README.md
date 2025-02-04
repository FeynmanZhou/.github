<details>

<summary>Tips for collapsed sections</summary>

### You can add a header

You can add text within a collapsed section. 

You can add an image or a code block, too.

```yaml
# build and push an OCI artifact to ACR, setup notation and sign the artifact
name: notation-github-actions-sign-template

on:
  push:

env:
  ACR_REGISTRY_NAME: <registry_name_of_your_ACR>        # example: myRegistry.azurecr.io
  ACR_REPO_NAME: <repository_name_of_your_ACR>          # example: myRepo
  ACR_USERNAME: <user_name_of_your_ACR>                 # example: myRegistry
  AKV_NAME: <your_Azure_Key_Vault_Name>                 # example: myNotationAKV
  KEY_ID: <key_id_of_your_private_key_to_sign_in_AKV>   # example: https://mynotationakv.vault.azure.net/keys/notationLeafCert/c585b8ad8fc542b28e41e555d9b3a1fd
  NOTATION_EXPERIMENTAL: 1                              # [Optional] when set, can use Referrers API in the workflow

jobs:
  notation-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: prepare
        id: prepare
        # using `v1` as an example tag, user can pick their own
        run: |
          echo "target_artifact_reference=${{ env.ACR_REGISTRY_NAME }}/${{ env.ACR_REPO_NAME }}:v1" >> "$GITHUB_ENV"
      # Log into your ACR
      - name: docker login
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_REGISTRY_NAME }}
          username: ${{ env.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      # Build and Push an OCI artifact to ACR
      # Using `Dockerfile` as an example to build an OCI artifact
      - name: Build and push
        id: push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ env.target_artifact_reference }}
      # Get the manifest digest of the OCI artifact
      - name: Retrieve digest
        run: |
          echo "target_artifact_reference=${{ env.ACR_REGISTRY_NAME }}/${{ env.ACR_REPO_NAME }}@${{ steps.push.outputs.digest }}" >> "$GITHUB_ENV"
      # Log into Azure in order to access AKV
      - name: Azure login
        uses: Azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          allow-no-subscriptions: true
      
      # Install Notation CLI, the default version is "1.0.0"
      - name: setup notation
        uses: notaryproject/notation-action/setup@main
      
      # Sign your OCI artifact using private key stored in AKV
      - name: sign OCI artifact using key pair from AKV
        uses: notaryproject/notation-action/sign@main
        with:
          plugin_name: azure-kv
          plugin_url: https://github.com/Azure/notation-azure-kv/releases/download/v1.0.0/notation-azure-kv_1.0.0_linux_amd64.tar.gz
          plugin_checksum: 82d4fee34dfe5e9303e4340d8d7f651da0a89fa8ae03195558f83bb6fa8dd263
          key_id: ${{ env.KEY_ID }}
          target_artifact_reference: ${{ env.target_artifact_reference }}
          signature_format: cose
          plugin_config: |-
            ca_certs=.github/cert-bundle/cert-bundle.crt
            self_signed=false
          # if using self-signed certificate from AKV, then the plugin_config should be:
          # plugin_config: |-
          #   self_signed=true
          allow_referrers_api: 'true'
```

</details>
