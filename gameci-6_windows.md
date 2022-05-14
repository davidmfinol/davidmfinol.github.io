# GameCI 6: Build and Deploy with Windows

Continuing from [GameCI 5](gameci-5_mac.html), let's examine the `Build with Windows` and `Deploy to the Microsoft Store` jobs.

## Build with Windows

{% highlight yml %}
{% raw %}
  buildWithWindows:
    name: Build for ${{ matrix.targetPlatform }}
    runs-on: windows-2019
    needs: [buildWithLinux, buildWithMac]
    outputs:
      buildVersion: ${{ steps.build.outputs.buildVersion }}
    strategy:
      fail-fast: false
      matrix:
        targetPlatform:
          - StandaloneWindows
          - StandaloneWindows64
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          lfs: true
      - uses: actions/cache@v3
        with:
          path: Library
          key: Library-buildWindows-${{ matrix.targetPlatform }}-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: |
            Library-buildWindows-${{ matrix.targetPlatform }}-
            Library-buildWindows-
      - name: Build Unity Project
        id: build
        uses: game-ci/unity-builder@v2
        env:
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          UNITY_SERIAL: ${{ secrets.UNITY_SERIAL }}
        with:
          targetPlatform: ${{ matrix.targetPlatform }}
          buildMethod: Cgs.Editor.BuildCgs.BuildOptions
      - name: Upload Build
        uses: actions/upload-artifact@v3
        if: github.event.action == 'published' || contains(github.event.inputs.workflow_mode, matrix.targetPlatform) || (contains(github.event.inputs.workflow_mode, 'Steam') && contains(matrix.targetPlatform, 'StandaloneWindows'))
        with:
          name: cgs-${{ matrix.targetPlatform }}
          path: build/${{ matrix.targetPlatform }}
      - name: Zip Build
        uses: vimtor/action-zip@v1
        if: github.event.action == 'published' && matrix.targetPlatform != 'WSAPlayer'
        with:
          files: build/${{ matrix.targetPlatform }}/
          dest: build/cgs-${{ matrix.targetPlatform }}.zip
      - name: Upload Zip to GitHub Release
        uses: svenstaro/upload-release-action@v2
        if: github.event.action == 'published' && matrix.targetPlatform != 'WSAPlayer'
        with:
          repo_token: ${{ secrets.CGS_PAT }}
          asset_name: cgs-${{ matrix.targetPlatform }}.zip
          file: build/cgs-${{ matrix.targetPlatform }}.zip
          tag: ${{ github.ref }}
          overwrite: true
          body: ${{ github.event.release.body }}
{% endraw %}
{% endhighlight %}

TODO

## Deploy to the Microsoft Store

{% highlight yml %}
{% raw %}
  deployToMicrosoftStore:
    name: Deploy to the Microsoft Store
    runs-on: windows-2019
    needs: buildWithWindows
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'WSAPlayer'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          lfs: true
      - name: Build Unity Project
        id: build
        uses: game-ci/unity-builder@v2
        env:
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          UNITY_SERIAL: ${{ secrets.UNITY_SERIAL }}
        with:
          targetPlatform: WSAPlayer
          buildMethod: Cgs.Editor.BuildCgs.BuildOptions
      - name: Upload Build
        uses: actions/upload-artifact@v3
        with:
          name: cgs-WSAPlayer
          path: build/WSAPlayer
      - name: Checkout Card-Game-Simulator
        run: |
          mkdir C:/Card-Game-Simulator.git
          git clone https://github.com/finol-digital/Card-Game-Simulator.git C:/Card-Game-Simulator.git --depth=1
          mkdir C:/Card-Game-Simulator.git/build
          cp -R build/WSAPlayer C:/Card-Game-Simulator.git/build
          ls C:/Card-Game-Simulator.git/build/WSAPlayer
      - name: Update Release Notes
        working-directory: C:/Card-Game-Simulator.git
        if: github.event.action == 'published'
        shell: bash
        env:
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: echo "$RELEASE_NOTES" > fastlane/metadata/en-US/release_notes.txt
      - name: Get Release Notes
        working-directory: C:/Card-Game-Simulator.git
        id: changelog
        shell: bash
        run: |
          export RELEASE_NOTES="$(cat fastlane/metadata/en-US/release_notes.txt)"
          RELEASE_NOTES="${RELEASE_NOTES//'%'/'%25'}"
          RELEASE_NOTES="${RELEASE_NOTES//$'\n'/'%0A'}"
          RELEASE_NOTES="${RELEASE_NOTES//$'\r'/'%0D'}"
          echo "$RELEASE_NOTES"
          echo "::set-output name=RELEASE_NOTES::$RELEASE_NOTES"
      - name: Apply Release Notes
        uses: davidmfinol/replace-action@master
        with:
          files: "C:/Card-Game-Simulator.git/storebroker/en-us/PDP.xml"
          replacements: "OUTPUT_RELEASE_NOTES=${{ steps.changelog.outputs.RELEASE_NOTES }}"
      - name: Setup Unity UWP
        uses: kuler90/setup-unity@v1
        with:
          unity-modules: universal-windows-platform
          project-path: C:/Card-Game-Simulator.git
      - name: Setup Developer Command Prompt for Microsoft Visual C++
        uses: ilammy/msvc-dev-cmd@v1
      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v1
      - name: Remove spaces from project name
        uses: davidmfinol/replace-action@master
        with:
          files: "C:/Card-Game-Simulator.git/build/WSAPlayer/Card Game Simulator.sln"
          replacements: "\"Card Game Simulator\"=\"CardGameSimulator\""
      - name: Remove spaces from project name 2
        uses: davidmfinol/replace-action@master
        with:
          files: "C:/Card-Game-Simulator.git/build/WSAPlayer/Card Game Simulator/Card Game Simulator.vcxproj"
          replacements: "</PropertyGroup>=<ProjectName>CardGameSimulator</ProjectName></PropertyGroup>"
      - name: Update manifest name
        working-directory: C:/Card-Game-Simulator.git
        shell: pwsh
        env:
          UwpProjectDirectory: build\WSAPlayer\Card Game Simulator
        run: |
          [xml]$manifest = get-content ".\$env:UwpProjectDirectory\Package.appxmanifest"
          $manifest.Package.Identity.Name = "FinolDigitalLLC.CardGameSimulator"
          $manifest.Package.Identity.Publisher = "CN=BBF9912B-079E-4CCE-A441-0D6EA6798115"
          $manifest.save(".\$env:UwpProjectDirectory\Package.appxmanifest")
      - name: Decode the Pfx
        working-directory: C:/Card-Game-Simulator.git
        shell: pwsh
        env:
          UwpProjectDirectory: build\WSAPlayer\Card Game Simulator
          SigningCertificate: Card Game Simulator_StoreKey.pfx
        run: |
          $pfx_cert_byte = [System.Convert]::FromBase64String("${{ secrets.MICROSOFT_STORE_PFX_FILE }}")
          $currentDirectory = Get-Location
          $certificatePath = Join-Path -Path $currentDirectory -ChildPath $env:UwpProjectDirectory -AdditionalChildPath $env:SigningCertificate
          [IO.File]::WriteAllBytes("$certificatePath", $pfx_cert_byte)
      - name: Build the .appxupload
        working-directory: C:/Card-Game-Simulator.git
        shell: pwsh
        env:
          SolutionPath: build\WSAPlayer\Card Game Simulator.sln
          SigningCertificate: Card Game Simulator_StoreKey.pfx
        run: msbuild $env:SolutionPath /p:Configuration="Master" /p:Platform="x64" /p:UapAppxPackageBuildMode="StoreUpload" /p:AppxBundle="Always" /p:AppxBundlePlatforms="x86|x64|arm" /p:PackageCertificateKeyFile=$env:SigningCertificate
      - name: Remove the .pfx
        working-directory: C:/Card-Game-Simulator.git
        shell: pwsh
        env:
          UwpProjectDirectory: build\WSAPlayer\Card Game Simulator
          SigningCertificate: Card Game Simulator_StoreKey.pfx
        run: Remove-Item -path $env:UwpProjectDirectory\$env:SigningCertificate
      - name: Upload .appxupload
        uses: actions/upload-artifact@v3
        with:
          name: ${{ format('CardGameSimulator_{0}.0_x64_bundle_Master.appxupload', needs.buildWithWindows.outputs.buildVersion) }}
          path: ${{ format('{0}\build\WSAPlayer\AppPackages\CardGameSimulator\CardGameSimulator_{1}.0_x64_bundle_Master.appxupload', 'C:\Card-Game-Simulator.git', needs.buildWithWindows.outputs.buildVersion) }}
      - name: Upload to the Microsoft Store
        working-directory: C:/Card-Game-Simulator.git
        shell: pwsh
        env:
          MICROSOFT_TENANT_ID: ${{ secrets.MICROSOFT_TENANT_ID }}
          MICROSOFT_CLIENT_ID: ${{ secrets.MICROSOFT_CLIENT_ID }}
          MICROSOFT_KEY: ${{ secrets.MICROSOFT_KEY }}
          MICROSOFT_APP_ID: 9N96N5S4W3J0
          STOREBROKER_CONFIG_PATH: ${{ format('{0}\storebroker\SBConfig.json', 'C:\Card-Game-Simulator.git') }}
          PDP_ROOT_PATH: ${{ format('{0}\storebroker\', 'C:\Card-Game-Simulator.git') }}
          IMAGES_ROOT_PATH: ${{ format('{0}\docs\assets\img\', 'C:\Card-Game-Simulator.git') }}
          APPX_PATH: ${{ format('{0}\build\WSAPlayer\AppPackages\CardGameSimulator\CardGameSimulator_{1}.0_x64_bundle_Master.appxupload', 'C:\Card-Game-Simulator.git', needs.buildWithWindows.outputs.buildVersion) }}
          OUT_PATH: ${{ format('{0}\build\WSAPlayer\', 'C:\Card-Game-Simulator.git') }}
          SUBMISSION_DATA_PATH: ${{ format('{0}\build\WSAPlayer\upload.json', 'C:\Card-Game-Simulator.git') }}
          PACKAGE_PATH: ${{ format('{0}\build\WSAPlayer\upload.zip', 'C:\Card-Game-Simulator.git') }}
        run: |
          Install-Module -Name StoreBroker -AcceptLicense -Force
          $pass = ConvertTo-SecureString $env:MICROSOFT_KEY -AsPlainText -Force
          $cred = New-Object System.Management.Automation.PSCredential ($env:MICROSOFT_CLIENT_ID, $pass)
          Set-StoreBrokerAuthentication -TenantId $env:MICROSOFT_TENANT_ID -Credential $cred
          New-SubmissionPackage -ConfigPath $env:STOREBROKER_CONFIG_PATH -PDPRootPath $env:PDP_ROOT_PATH -ImagesRootPath $env:IMAGES_ROOT_PATH -AppxPath $env:APPX_PATH -OutPath $env:OUT_PATH -OutName 'upload' -Verbose
          Update-ApplicationSubmission -AppId $env:MICROSOFT_APP_ID -SubmissionDataPath $env:SUBMISSION_DATA_PATH -PackagePath $env:PACKAGE_PATH -ReplacePackages -UpdateListings -AutoCommit -Force
{% endraw %}
{% endhighlight %}

TODO

## Continue
If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 7: Conclusion](gameci-7_conclusion.html).
