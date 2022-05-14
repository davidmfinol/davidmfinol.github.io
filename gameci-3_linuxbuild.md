# GameCI 3: Build with Linux

Continuing from [GameCI 2](gameci-2_testing.html), let's examine the `Build with Linux` job matrix.
We would ideally run all builds on Linux, but there are some constraints that mean we are only able to run a subset of our target builds on Linux.
Those target platforms are `Android`, `iOS`, `StandaloneLinux64`, and `WebGL`.
[The GameCI documentation website](https://game.ci/docs/github) is the best resource for learning how to build with GameCI, so I would recommend starting there before we examine my workflow's code below.

## The Code

{% highlight yml %}
{% raw %}
  buildWithLinux:
    name: Build for ${{ matrix.targetPlatform }}
    runs-on: ubuntu-latest
    needs: tests
    strategy:
      fail-fast: false
      matrix:
        targetPlatform:
          - Android
          - iOS
          - StandaloneLinux64
          - WebGL
    outputs:
      buildVersion: ${{ steps.build.outputs.buildVersion }}
    steps:
      - name: Free Disk Space for Android
        if: matrix.targetPlatform == 'Android'
        run: |
          df -h
          sudo swapoff -a
          sudo rm -f /swapfile
          sudo rm -rf /usr/share/dotnet
          sudo rm -rf /opt/ghc
          sudo rm -rf "/usr/local/share/boost"
          sudo rm -rf "$AGENT_TOOLSDIRECTORY"
          df -h
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          lfs: true
      - uses: actions/cache@v3
        with:
          path: Library
          key: Library-build-${{ matrix.targetPlatform }}-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: |
            Library-build-${{ matrix.targetPlatform }}-
            Library-build-
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
          androidAppBundle: true
          androidKeystoreName: finoldigital.keystore
          androidKeystoreBase64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          androidKeystorePass: ${{ secrets.ANDROID_KEYSTORE_PASS }}
          androidKeyaliasName: cgs
          androidKeyaliasPass: ${{ secrets.ANDROID_KEYALIAS_PASS }}
          androidTargetSdkVersion: AndroidApiLevel31
      - name: Upload Build
        uses: actions/upload-artifact@v3
        if: github.event.action == 'published' || contains(github.event.inputs.workflow_mode, matrix.targetPlatform) || (contains(github.event.inputs.workflow_mode, 'Steam') && matrix.targetPlatform == 'StandaloneLinux64')
        with:
          name: cgs-${{ matrix.targetPlatform }}
          path: build/${{ matrix.targetPlatform }}
      - name: Zip Build
        uses: montudor/action-zip@v1
        if: github.event.action == 'published' && matrix.targetPlatform == 'StandaloneLinux64'
        with:
          args: zip -qq -r build/cgs-${{ matrix.targetPlatform }}.zip build/${{ matrix.targetPlatform }}
      - name: Upload Zip to GitHub Release
        uses: svenstaro/upload-release-action@v2
        if: github.event.action == 'published' && matrix.targetPlatform == 'StandaloneLinux64'
        with:
          repo_token: ${{ secrets.CGS_PAT }}
          asset_name: cgs-${{ matrix.targetPlatform }}.zip
          file: build/cgs-${{ matrix.targetPlatform }}.zip
          tag: ${{ github.ref }}
          overwrite: true
          body: ${{ github.event.release.body }}
{% endraw %}
{% endhighlight %}

Most of this job matrix should be self-explanatory after reading [the GameCI Builder docs](https://game.ci/docs/github/builder), but I'll provide some additional details for each target platform below.

## Android

A common issue with Android builds on the GitHub runners is running out of disk space.
The `Free Disk Space for Android` step attempts to free up some space to hopefully prevent this issue, but if you have a larger project, you may need to do further work to avoid this issue:
```bash
df -h
sudo swapoff -a
sudo rm -f /swapfile
sudo rm -rf /usr/share/dotnet
sudo rm -rf /opt/ghc
sudo rm -rf "/usr/local/share/boost"
sudo rm -rf "$AGENT_TOOLSDIRECTORY"
df -h
```

For additional details about Android builds and deployment, refer to [the GameCI Android docs](https://game.ci/docs/github/deployment/android).

## iOS

Building for iOS is a 2-stage process: This `Build with Linux` job is the first stage, but the second stage requires a macOS runner.
This job generates an Xcode project, which is uploaded as an artifact to be used in [GameCI 5: Build and Deploy with MacOS](gameci-5_mac.html).

For additional details about iOS builds and deployment, refer to [the GameCI iOS docs](https://game.ci/docs/github/deployment/ios).

## StandaloneLinux64

The Linux executable can be deployed via Steam (see [GameCI 7: Conclusion](gameci-7_conclusion.html), but the `Zip Build` and `Upload Zip to GitHub Release` steps also enable players to get the build from the GitHub Releases page.
Note that the `Upload Zip to GitHub Release` step requires a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

## WebGL

For details on deploying the WebGL build, see [GameCI 4: Deploy with Linux](gameci-4_linuxdeploy.html).

## Continue
If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 4: Deploy with Linux](gameci-4_linuxdeploy.html).
