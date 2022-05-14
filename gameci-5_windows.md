# GameCI 6: Build and Deploy with Windows

Continuing from [GameCI 4](gameci-4_mac.html), let's examine the `Build with Windows` and `Deploy to the Microsoft Store` jobs.

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

Most of this job should be self-explanatory after reading [the GameCI Builder docs](https://game.ci/docs/github/builder), but here are some additional details:

Instead of using a Linux runner, we need a Windows runner because Windows builds require Windows as a runner if using [IL2CPP as the scripting backend](https://docs.unity3d.com/Manual/IL2CPP.html). 
I recommend using IL2CPP, for better performance than Mono.

The Windows executables can be deployed via Steam (see [GameCI 6: Conclusion](gameci-6_conclusion.html), but the `Zip Build` and `Upload Zip to GitHub Release` steps also enable players to get the builds from the GitHub Releases page.
Note that the `Upload Zip to GitHub Release` step requires a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

You might also ask: Why does this job run after the Linux and Mac builds instead of in parallel to those builds?
The answer is because I am using a Unity Pro license which only allows 2 seats to be used at a time.
See the [GameCI docs for details](https://game.ci/docs/docker/docker-images#concurrent-builds-on-windows-and-macos).

I don't have much else to say about building the 32-bit and 64-bit Windows executables, but I definitely do have a ton more to say about building for WSAPlayer and deploying to the Microsoft Store...

## Deploy to the Microsoft Store

So I'll dump the code here, but I highly recommend you first skip past it and come back to it as necessary:

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

### Some Context

Universal Windows Platform (UWP) apps are funny.
I don't mean "haha"-funny; I mean "huh?"-funny.
Apps using UWP are MUCH less popular than .exe's, and as such, there seems to be a dearth of documentation for how to even create and/or work with UWP.
At one point, it seemed like UWP could have been the future for Windows, as UWP used to be a requirement for getting onto the Windows Store.

That requirement is what motivated me to get my app to build with UWP, but that requirement is no longer true.
Now, I could just submit the .exe's from above to the Microsoft Store, and that would save me some pain.
But you know what?

I like UWP apps.

I don't necessarily like developing for the UWP platform (we'll get to it), but I do like using the UWP version of my app over the .exe version of my app.
I like how it's easier to install and to run.
I like how it automatically handles both full-screen and windowed views.
And most of all, I like how I can have functional deep links for the UWP version of my app. 

I consider [Deep Linking](https://en.wikipedia.org/wiki/Mobile_deep_linking) to be critical functionality for my app across all platforms, even the non-mobile, desktop platforms.
The fact that I have working deep links on UWP but not on .exe is enough for me to go through all of the below.

### Start by Building to a Solution

The start of this job is pretty similar to the other jobs we've examined thus far.
This job only runs when triggered by either a GitHub Release or a workflow dispatch with `release WSAPlayer` as the input.
This job then uses `game-ci/unity-builder` to generate and then upload the WSAPlayer artifact, which is a Visual Studio Project/Solution.

So far, so good. The next step should simply be to use that Visual Studio Project/Solution to create a .appxupload file and then submit that file to the Microsoft Store.

But it gets complicated.

### Check It Out Again

While developing this job, I ran into `out of disk space` issues.
If you read part 3, you may remember that the Android builds are likely to run into the same issue.
You could work around the issue on Android by clearing some disk space, but I couldn't find a way to do the same here.
Instead, I considered some other options:
1. I could use a self-hosted runner with more space on it. This would require spending money in addition to time/effort, so I really wanted to keep this option as a last resort.
2. I could split the job into multiple jobs that would run in parallel, and then there would be another job that would combine the results. This would introduce a lot of complexity, so I didn't like this option either.
3. I could use the C: drive of the GitHub runner, which could possibly have more space than the default drive. This is the option I chose.

Having chosen to use C: for the process of building the .appxupload file, I start by re-cloning the repo to C:, but this time, it's a shallow clone.
Going forward, I have to be careful to use `C:/Card-Game-Simulator.git` instead of the default working directory.

### Building the .appxupload

Logically, the next step is to take the WSAPlayer artifact and use it to build the .appxupload. In the `Deploy to the Microsoft Store` job, I have some steps related to Release Notes first, but I'll explain those steps later.

So how do you actually build the .appxupload? 

Good question!

I used, for example, [this guide](https://www.dotnetapp.com/github-actions-for-uwp-apps-the-good-and-the-bad-and-the-ugly/) as a reference, but ultimately, I kept running into issues that kept forcing me to come up with my own solutions.

First, the simple part: install necessary dependencies/tools with the `Setup Unity UWP`, `Setup Developer Command Prompt for Microsoft Visual C++`, and `Setup MSBuild` steps.
Next, pain: my project's name has spaces in it, but these tools do not like spaces.

So I need to go through the files in the Visual Studio Project/Solution, find instances of my project name, and replace those instances with a space-less version.
Simple enough, right? 
I tried playing around with `sed` and `awk`, but I've never been a fan. 
I searched for a GitHub Action that could work for me, but I couldn't find a perfect one. 
I ended up [forking an action](https://github.com/davidmfinol/replace-action) to get it to work for me.

Alright, space issue fixed. 
Just one more issue: Unity doesn't generate a Visual Studio Project/Solution with the expected Microsoft Store identifiers.
So I use some Powershell scripting in the `Update manifest name` step to add the expected identifiers.

The `Decode the Pfx`, `Build the .appxupload`, and `Remove the .pfx` steps are then relatively painless.
Well, I say painless, but that's because I've already discussed the `out of disk space` issue.
Specifically, this was the issue: `/p:AppxBundlePlatforms="x86|x64|arm"`.
My .appxupload is a zip file with 3 packages: 1 for each of x86, x64, and arm.
As I mentioned before, I could have built these 3 packages on separate runners and merged them afterwards, but I'm glad I didn't try going down that route.

### StoreBroker

So now that we have the .appxupload, we just need to submit it to the Microsoft Store.

The tool that Microsoft provides for this purpose is [StoreBroker](https://github.com/microsoft/StoreBroker).

StoreBroker expects that the release notes exist in a `PDP.xml` file, so let's review those Release Notes steps.
`Update Release Notes` gets the release notes from the GitHub Release and saves it to a fastlane file. 
`Get Release Notes` gets the release notes from the fastlane file.
`Apply Release Notes` is where we go through the `PDP.xml` and replace the release notes with the new release notes.
Luckily, I already have the `davidmfinol/replace-action` action to help with this.

For the `Upload to the Microsoft Store` step itself, refer to the [StoreBroker docs](https://github.com/microsoft/StoreBroker/blob/master/Documentation/SETUP.md).

It wasn't easy, but getting all of the above to succeed has been hugely satisfying for me.
I suspect that there are not many other Unity developers that are building UWP apps for the Microsoft Store.
If you're a fellow Unity + UWP developer, I hope your journey to an automated deployment pipeline is easier than mine was!


## Continue

I recommend concluding with [GameCI 6: Conclusion](gameci-6_conclusion.html).
