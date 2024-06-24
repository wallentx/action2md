# α²⟫🅫

`action2md` parses GitHub Actions workflow `.yml` files and generates a Markdown summary of the actions, and steps defined within them. This tool is useful for documenting workflows in a human-readable format. It's also useful if you just want to easily, and quickly see how a project is being built in CI, so that you can build it locally.

## Features

- Extracts workflow name, jobs, steps, and relevant details from GitHub Actions YAML files.
- Outputs to stdout, a specified file, or a file in the same directory as the workflow file.
- Supports formatting of `uses` links, `with` inputs, and `outputs`.

## Requirements

- `yq`: Command-line YAML, JSON, XML, CSV, TOML, and properties processor. https://github.com/mikefarah/yq

## Installation

1. **Install `yq`**:
   https://github.com/mikefarah/yq#install

2. **Download the `action2md` script**:
   ```sh
   curl -O https://github.com/wallentx/action2md/blob/main/action2md
   chmod +x action2md
   mv action2md /some/directory/with/your/PATH/
   ```

## Usage

### Basic Usage

To output the Markdown summary to stdout:
```sh
./action2md <workflow-file>
```

### Write to a Specific File

To specify an output file path:
```sh
./action2md -o /path/to/output.md <workflow-file>
```

### Write Alongside Workflow File

To write the output in the same directory as the workflow file:
```sh
./action2md -w <workflow-file>
```

### Options

- `-o <output-file>`: Specify the output file path.
- `-w`: Write output alongside the action.yml file.
- If no options are specified, the script will output to stdout.

### Example

The following is a sample of the markdown it produced from parsing the Chia-Blockchain [build-linux-installer-deb.yml](https://github.com/Chia-Network/chia-blockchain/blob/main/.github/workflows/build-linux-installer-deb.yml):

---

# Workflow: 📦🚀 Build Installer - Linux DEB
## Job: Build ${{ matrix.os.arch }}
> ### clean-workspace
>  * Uses: [Chia-Network/actions/clean-workspace@main](https://github.com/Chia-Network/actions/clean-workspace)
> ### Add safe git directory
>  * Uses: [Chia-Network/actions/git-mark-workspace-safe@main](https://github.com/Chia-Network/actions/git-mark-workspace-safe)
> ### Checkout Code
>  * Uses: [actions/checkout@v4](https://github.com/actions/checkout)
>> | Input | Value |
>> | :------------ | -----:|
>> | fetch-depth | 0 |
>> | submodules | recursive |
> ### git-ssh-to-https
>  * Uses: [Chia-Network/actions/git-ssh-to-https@main](https://github.com/Chia-Network/actions/git-ssh-to-https)
> ### Cleanup any leftovers that exist from previous runs
> ```bash
> bash build_scripts/clean-runner.sh || true
> ```
> ### Set Env
>  * Uses: [Chia-Network/actions/setjobenv@main](https://github.com/Chia-Network/actions/setjobenv)
> ### Check tag type
> ```bash
> REG_B="^[0-9]+\.[0-9]+\.[0-9]+-b[0-9]+$"
> REG_RC="^[0-9]+\.[0-9]+\.[0-9]+-rc[0-9]+$"
> if [[ "${{ github.event.release.tag_name }}" =~ $REG_B ]] || [[ "${{ inputs.release_type }}" =~ $REG_B ]]; then
>   echo "TAG_TYPE=beta"
>   echo "TAG_TYPE=beta" >> "$GITHUB_ENV"
> elif [[ "${{ github.event.release.tag_name }}" =~ $REG_RC ]] || [[ "${{ inputs.release_type }}" =~ $REG_RC ]]; then
>   echo "TAG_TYPE=rc"
>   echo "TAG_TYPE=rc" >> "$GITHUB_ENV"
> fi
> ```
> ### Get version number
> <sup>**ID**: _version\_number_</sup>
> ```bash
> python3 -m venv ../venv
> . ../venv/bin/activate
> pip3 install setuptools_scm
> echo "CHIA_INSTALLER_VERSION=$(python3 ./build_scripts/installer-version.py)" >> "$GITHUB_OUTPUT"
> deactivate
> ```
> ### Get latest madmax plotter
> ```bash
> LATEST_MADMAX=$(gh api repos/Chia-Network/chia-plotter-madmax/releases/latest --jq 'select(.prerelease == false) | .tag_name')
> mkdir "$GITHUB_WORKSPACE"/madmax
> gh release download -R Chia-Network/chia-plotter-madmax "$LATEST_MADMAX" -p 'chia_plot-*-${{ matrix.os.madmax-suffix }}' -O "$GITHUB_WORKSPACE"/madmax/chia_plot
> gh release download -R Chia-Network/chia-plotter-madmax "$LATEST_MADMAX" -p 'chia_plot_k34-*-${{ matrix.os.madmax-suffix }}' -O "$GITHUB_WORKSPACE"/madmax/chia_plot_k34
> chmod +x "$GITHUB_WORKSPACE"/madmax/chia_plot
> chmod +x "$GITHUB_WORKSPACE"/madmax/chia_plot_k34
> ```
> ### Fetch bladebit versions
> ```bash
> # Fetch the latest version of each type
> LATEST_RELEASE=$(gh api repos/Chia-Network/bladebit/releases/latest --jq 'select(.prerelease == false) | .tag_name')
> LATEST_BETA=$(gh api repos/Chia-Network/bladebit/releases --jq 'map(select(.prerelease) | select(.tag_name | test("^v[0-9]+\\.[0-9]+\\.[0-9]+-beta[0-9]+$"))) | first | .tag_name')
> LATEST_RC=$(gh api repos/Chia-Network/bladebit/releases --jq 'map(select(.prerelease) | select(.tag_name | test("^v[0-9]+\\.[0-9]+\\.[0-9]+-rc[0-9]+$"))) | first | .tag_name')
> 
> # Compare the versions and choose the newest that matches the requirements
> if [[ "$TAG_TYPE" == "beta" || -z "$TAG_TYPE" ]]; then
>   # For beta or dev builds (indicated by the absence of a tag), use the latest version available
>   LATEST_VERSION=$(printf "%s\n%s\n%s\n" "$LATEST_RELEASE" "$LATEST_BETA" "$LATEST_RC" | sed '/-/!s/$/_/' | sort -V | sed 's/_$//' | tail -n 1)
> elif [[ "$TAG_TYPE" == "rc" ]]; then
>   # For RC builds, use the latest RC or full release if it's newer
>   LATEST_VERSION=$(printf "%s\n%s\n" "$LATEST_RELEASE" "$LATEST_RC" | sed '/-/!s/$/_/' | sort -V | sed 's/_$//' | tail -n 1)
> else
>   # For full releases, use the latest full release
>   LATEST_VERSION="$LATEST_RELEASE"
> fi
> echo "LATEST_VERSION=$LATEST_VERSION" >> "$GITHUB_ENV"
> ```
> ### Get latest bladebit plotter
> ```bash
> # Download and extract the chosen version
> mkdir "$GITHUB_WORKSPACE"/bladebit
> cd "$GITHUB_WORKSPACE"/bladebit
> gh release download -R Chia-Network/bladebit "$LATEST_VERSION" -p 'bladebit*-${{ matrix.os.bladebit-suffix }}'
> ls *.tar.gz | xargs -I{} bash -c 'tar -xzf {} && rm {}'
> ls bladebit* | xargs -I{} chmod +x {}
> cd "$OLDPWD"
> ```
> ### install
>  * Uses: [./.github/actions/install](././.github/actions/install)
>> | Input | Value |
>> | :------------ | -----:|
>> | python-version | ${{ matrix.python-version }} |
>> | development | true |
>> | constraints-file-artifact-name | constraints-file-${{ matrix.os.arch }} |
> ### activate-venv
>  * Uses: [chia-network/actions/activate-venv@main](https://github.com/chia-network/actions/activate-venv)
> ### Prepare GUI cache
> <sup>**ID**: _gui-ref_</sup>
> ```bash
> gui_ref=$(git submodule status chia-blockchain-gui | sed -e 's/^ //g' -e 's/ chia-blockchain-gui.*$//g')
> echo "${gui_ref}"
> echo "GUI_REF=${gui_ref}" >> "$GITHUB_OUTPUT"
> echo "rm -rf ./chia-blockchain-gui"
> rm -rf ./chia-blockchain-gui
> ```
> ### Cache GUI
> <sup>**ID**: _cache-gui_</sup>
>  * Uses: [actions/cache@v4](https://github.com/actions/cache)
>> | Input | Value |
>> | :------------ | -----:|
>> | path | ./chia-blockchain-gui |
>> | key | ${{ runner.os }}-${{ matrix.os.arch }}-chia-blockchain-gui-${{ steps.gui-ref.outputs.GUI_REF }} |
> ### Build GUI
> ```bash
> cd ./build_scripts
> bash build_linux_deb-1-gui.sh
> ```
> ### Build .deb package
> ```bash
> ldd --version
> cd ./build_scripts
> sh build_linux_deb-2-installer.sh ${{ matrix.os.arch }}
> ```
> ### Upload Linux artifacts
>  * Uses: [actions/upload-artifact@v4](https://github.com/actions/upload-artifact)
>> | Input | Value |
>> | :------------ | -----:|
>> | name | chia-installers-linux-deb-${{ matrix.os.arch }} |
>> | path | ${{ github.workspace }}/build_scripts/final_installer/ |
> ### Remove working files to exclude from cache
> ```bash
> rm -rf ./chia-blockchain-gui/packages/gui/daemon
> ```
>> | Output | Value |
>> | :------------ | -----:|
>> | chia-installer-version | ${{ steps.version_number.outputs.CHIA_INSTALLER_VERSION }} |
## Job: Publish ${{ matrix.os.arch }}
> ### clean-workspace
>  * Uses: [Chia-Network/actions/clean-workspace@main](https://github.com/Chia-Network/actions/clean-workspace)
> ### create-venv
> <sup>**ID**: _create-venv_</sup>
>  * Uses: [chia-network/actions/create-venv@main](https://github.com/chia-network/actions/create-venv)
> ### activate-venv
>  * Uses: [chia-network/actions/activate-venv@main](https://github.com/chia-network/actions/activate-venv)
>> | Input | Value |
>> | :------------ | -----:|
>> | directories | ${{ steps.create-venv.outputs.activate-venv-directories }} |
> ### Download constraints file
>  * Uses: [actions/download-artifact@v4](https://github.com/actions/download-artifact)
>> | Input | Value |
>> | :------------ | -----:|
>> | name | constraints-file-${{ matrix.os.arch }} |
>> | path | venv |
> ### Install utilities
> ```bash
> pip install --constraint venv/constraints.txt py3createtorrent
> ```
> ### Download packages
>  * Uses: [actions/download-artifact@v4](https://github.com/actions/download-artifact)
>> | Input | Value |
>> | :------------ | -----:|
>> | name | chia-installers-linux-deb-${{ matrix.os.arch }} |
>> | path | build_scripts/final_installer/ |
> ### Set Env
>  * Uses: [Chia-Network/actions/setjobenv@main](https://github.com/Chia-Network/actions/setjobenv)
> ### Test for secrets access
> <sup>**ID**: _check\_secrets_</sup>
> ```bash
> unset HAS_AWS_SECRET
> unset HAS_GLUE_SECRET
> 
> if [ -n "$AWS_SECRET" ]; then HAS_AWS_SECRET='true' ; fi
> echo HAS_AWS_SECRET=${HAS_AWS_SECRET} >> "$GITHUB_OUTPUT"
> 
> if [ -n "$GLUE_API_URL" ]; then HAS_GLUE_SECRET='true' ; fi
> echo HAS_GLUE_SECRET=${HAS_GLUE_SECRET} >> "$GITHUB_OUTPUT"
> ```
> ### Configure AWS credentials
>  * Uses: [aws-actions/configure-aws-credentials@v4](https://github.com/aws-actions/configure-aws-credentials)
>> | Input | Value |
>> | :------------ | -----:|
>> | role-to-assume | arn:aws:iam::${{ secrets.CHIA_AWS_ACCOUNT_ID }}:role/installer-upload |
>> | aws-region | us-west-2 |
> ### Upload to s3
> ```bash
> GIT_SHORT_HASH=$(echo "${GITHUB_SHA}" | cut -c1-8)
> CHIA_DEV_BUILD=${CHIA_INSTALLER_VERSION}-$GIT_SHORT_HASH
> echo "CHIA_DEV_BUILD=$CHIA_DEV_BUILD" >> "$GITHUB_ENV"
> aws s3 cp "$GITHUB_WORKSPACE/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb" "s3://download.chia.net/dev/chia-blockchain_${CHIA_DEV_BUILD}_${{ matrix.os.arch }}.deb"
> aws s3 cp "$GITHUB_WORKSPACE/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb" "s3://download.chia.net/dev/chia-blockchain-cli_${CHIA_DEV_BUILD}-1_${{ matrix.os.arch }}.deb"
> ```
> ### Create Checksums
> ```bash
> ls "$GITHUB_WORKSPACE"/build_scripts/final_installer/
> sha256sum "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb > "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.sha256
> sha256sum "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb > "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb.sha256
> ls "$GITHUB_WORKSPACE"/build_scripts/final_installer/
> ```
> ### Create .deb torrent
> ```bash
> py3createtorrent -f -t udp://tracker.opentrackr.org:1337/announce "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb -o "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.torrent --webseed https://download.chia.net/install/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb
> py3createtorrent -f -t udp://tracker.opentrackr.org:1337/announce "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb -o "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb.torrent --webseed https://download.chia.net/install/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb
> gh release upload --repo ${{ github.repository }} $RELEASE_TAG "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.torrent
> ```
> ### Upload Dev Installer
> ```bash
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb s3://download.chia.net/latest-dev/chia-blockchain_${{ matrix.os.arch }}_latest_dev.deb
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.sha256 s3://download.chia.net/latest-dev/chia-blockchain_${{ matrix.os.arch }}_latest_dev.deb.sha256
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb s3://download.chia.net/latest-dev/chia-blockchain-cli_${{ matrix.os.arch }}_latest_dev.deb
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb.sha256 s3://download.chia.net/latest-dev/chia-blockchain-cli_${{ matrix.os.arch }}_latest_dev.deb.sha256
> ```
> ### Upload Release Files
> ```bash
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb s3://download.chia.net/install/
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.sha256 s3://download.chia.net/install/
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb.torrent s3://download.chia.net/torrents/
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb s3://download.chia.net/install/
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb.sha256 s3://download.chia.net/install/
> aws s3 cp "$GITHUB_WORKSPACE"/build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb.torrent s3://download.chia.net/torrents/
> ```
> ### Upload release artifacts
> ```bash
> gh release upload \
>   --repo ${{ github.repository }} \
>   $RELEASE_TAG \
>   build_scripts/final_installer/chia-blockchain_${CHIA_INSTALLER_VERSION}_${{ matrix.os.arch }}.deb \
>   build_scripts/final_installer/chia-blockchain-cli_${CHIA_INSTALLER_VERSION}-1_${{ matrix.os.arch }}.deb
> ```
> ### jwt
>  * Uses: [Chia-Network/actions/github/jwt@main](https://github.com/Chia-Network/actions/github/jwt)
> ### Mark pre-release installer complete
> ```bash
> curl -s -XPOST -H "Authorization: Bearer ${{ env.JWT_TOKEN }}" --data '{"chia_ref": "${{ env.RELEASE_TAG }}"}' ${{ secrets.GLUE_API_URL }}/api/v1/${{ env.RFC_REPO }}-prerelease/${{ env.RELEASE_TAG }}/success/${{ matrix.os.glue-name }}
> ```
> ### Mark release installer complete
> ```bash
> curl -s -XPOST -H "Authorization: Bearer ${{ env.JWT_TOKEN }}" --data '{"chia_ref": "${{ env.RELEASE_TAG }}"}' ${{ secrets.GLUE_API_URL }}/api/v1/${{ env.RFC_REPO }}/${{ env.RELEASE_TAG }}/success/${{ matrix.os.glue-name }}
> ```
## Job: Test ${{ matrix.distribution.name }} ${{ matrix.mode.name }} ${{ matrix.arch.name }}
> ### clean-workspace
>  * Uses: [Chia-Network/actions/clean-workspace@main](https://github.com/Chia-Network/actions/clean-workspace)
> ### Download packages
> <sup>**ID**: _download_</sup>
>  * Uses: [actions/download-artifact@v4](https://github.com/actions/download-artifact)
>> | Input | Value |
>> | :------------ | -----:|
>> | name | chia-installers-linux-deb-${{ matrix.arch.artifact-name }} |
>> | path | packages |
> ### Update apt repos
> ```bash
> apt-get update --yes
> ```
> ### Install package
> ```bash
> ls -l "${{ steps.download.outputs.download-path }}"
> apt-get install --yes libnuma1 "${{ steps.download.outputs.download-path }}"/${{ matrix.mode.file }}
> ```
> ### List /opt/chia contents
> ```bash
> find /opt/chia
> ```
> ### Run chia dev installers test
> ```bash
> chia dev installers test --expected-chia-version "${{ needs.build.outputs.chia-installer-version }}"
> ```
> ### Verify /opt/chia present
> ```bash
> if [ ! -e /opt/chia ]
> then
>   ls -l /opt
>   false
> fi
> ```
> ### Remove package
> ```bash
> apt-get remove --yes ${{ matrix.mode.package }}
> ```
> ### Verify /opt/chia not present
> ```bash
> if [ -e /opt/chia ]
> then
>   ls -lR /opt/chia
>   false
> fi
> ```

---

## Contributing

Feel free to open issues or submit pull requests for any improvements or bug fixes.
