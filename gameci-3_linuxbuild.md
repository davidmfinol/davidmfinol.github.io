# GameCI 3: Build with Linux

Continuing from [GameCI 2](gameci-2_testing.html), let's examine the `Build with Linux` job matrix.
We would ideally run all builds on Linux, but there are some constraints that mean we are only able to run a subset of our target builds on Linux.
Those target platforms are `Android`, `iOS`, `StandaloneLinux64`, and `WebGL`.
[The GameCI documentation website](https://game.ci/docs/github) is the best resource for learning how to build with GameCI, so I would recommend starting there before we examine my workflow's code below.

## The Code
```yml
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
```

Most of this job matrix should be self-explanatory after reading [the GameCI docs](https://game.ci/docs/github/builder), but I'll provide some additional notes for each target platform below.

## Android

***TODO***

## iOS

***TODO***

## StandaloneLinux64

***TODO***

## WebGL

***TODO***

## Continue
If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 4: Deploy with Linux](gameci-4_linuxdeploy.html).
