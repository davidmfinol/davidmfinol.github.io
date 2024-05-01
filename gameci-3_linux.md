# GameCI 3: Build and Deploy with Linux

Continuing from [GameCI 2](gameci-2_testing.html), let's examine the `Build with Linux` job matrix, along with the `Deploy to the Google Play Store` and `Deploy to the Web via GitHub Pages` jobs.

## Build with Linux

We would ideally run all builds on Linux, but there are some constraints that mean we are only able to run a subset of our target builds on Linux.
The eligible target platforms are `Android`, `iOS`, `StandaloneLinux64`, and `WebGL`.
Make sure to have read the [GameCI GitHub docs](https://game.ci/docs/github/getting-started) before examining this job matrix.

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
      - name: Create LFS file list
        run: git lfs ls-files -l | cut -d' ' -f1 | sort > .lfs-assets-id
      - name: Restore LFS cache
        uses: actions/cache@v3
        id: lfs-cache
        with:
          path: .git/lfs
          key: ${{ runner.os }}-lfs-${{ hashFiles('.lfs-assets-id') }}
      - name: Git LFS Pull
        run: |
          git lfs pull
          git add .
          git reset --hard
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

Most of this job matrix should be self-explanatory after reading the [GameCI Builder docs](https://game.ci/docs/github/builder), but you can find some additional details for each target platform below.

### Android

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

For additional details about Android builds and deployment, refer to the [GameCI Android docs](https://game.ci/docs/github/deployment/android).

### iOS

Building for iOS is a 2-stage process: This `Build with Linux` job is the first stage, but the second stage requires a macOS runner.
This job generates an Xcode project, which is uploaded as an artifact to be used in [GameCI 4: Build and Deploy with MacOS](gameci-4_mac.html).

For additional details about iOS builds and deployment, refer to the [GameCI iOS docs](https://game.ci/docs/github/deployment/ios).

### StandaloneLinux64

The Linux executable can be deployed via Steam (see [GameCI 6: Conclusion](gameci-6_conclusion.html)), but the `Zip Build` and `Upload Zip to GitHub Release` steps also enable players to get the build from the GitHub Releases page.
Note that the `Upload Zip to GitHub Release` step requires a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

### WebGL

WebGL builds simply create files which need to be hosted on a web server.
See the `Deploy to the Web via GitHub Pages` job below.

## Deploy to the Google Play Store

As mentioned earlier, the [GameCI Android docs](https://game.ci/docs/github/deployment/android) is the best resource for Android builds and deployment.

{% highlight yml %}
{% raw %}
  deployToGooglePlay:
    name: Deploy to the Google Play Store
    runs-on: ubuntu-latest
    needs: buildWithLinux
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'Android'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Download Android Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-Android
          path: build/Android
      - name: Update Release Notes
        if: github.event.action == 'published'
        env:
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: echo "$RELEASE_NOTES" > fastlane/metadata/android/en-US/changelogs/default.txt
      - name: Add Authentication
        env:
          GOOGLE_PLAY_KEY_FILE: ${{ secrets.GOOGLE_PLAY_KEY_FILE }}
          GOOGLE_PLAY_KEY_FILE_PATH: ${{ format('{0}/fastlane/api-finoldigital.json', github.workspace) }}
        run: echo "$GOOGLE_PLAY_KEY_FILE" > $GOOGLE_PLAY_KEY_FILE_PATH
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 2.7.2
          bundler-cache: true
      - name: Upload to Google Play
        env:
          GOOGLE_PLAY_KEY_FILE_PATH: ${{ format('{0}/fastlane/api-finoldigital.json', github.workspace) }}
          ANDROID_BUILD_FILE_PATH: ${{ format('{0}/build/Android/Android.aab', github.workspace) }}
          ANDROID_PACKAGE_NAME: com.finoldigital.cardgamesim
        uses: maierj/fastlane-action@v2.2.1
        with:
          lane: 'android playprod'
{% endraw %}
{% endhighlight %}

An additional detail about this job is that it only runs when triggered by either a GitHub Release or a workflow dispatch with `release Android` as the input.
Also note that we use `fastlane/metadata/android/en-US/changelogs/default.txt` to publish the release notes.

## Deploy to the Web via GitHub Pages

{% highlight yml %}
{% raw %}
  deployToGitHubPages:
    name: Deploy to the Web via GitHub Pages
    runs-on: ubuntu-latest
    needs: buildWithLinux
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'WebGL'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
      - name: Download WebGL Artifact
        uses: actions/download-artifact@v4
        with:
          name: cgs-WebGL
          path: build/WebGL
      - name: Copy the WebGL build artifacts to the GitHub Pages directory
        env:
          WEBGL_BUILD_PATH: ${{ format('{0}/build/WebGL/WebGL/.', github.workspace) }}
          WEBGL_PAGES_PATH: ${{ format('{0}/docs', github.workspace) }}
        run: cp -r $WEBGL_BUILD_PATH $WEBGL_PAGES_PATH;
      - name: Deploy to GitHub Pages
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          branch: main
          file_pattern: docs/**
          commit_message: Deploy to GitHub Pages
{% endraw %}
{% endhighlight %}

This job only runs when triggered by either a GitHub Release or a workflow dispatch with `release WebGL` as the input.
All this job does is copy the WebGL artifact to the correct location in the `/docs` folder of [GitHub Pages](https://pages.github.com/).
Once copied, simply committing the files will trigger a GitHub Pages deployment.
Note that this web page relies on custom html in the form of [cgs-WebGL.html](https://github.com/finol-digital/Card-Game-Simulator/blob/develop/docs/cgs-webgl.html), though work remains to make this web page look and feel nicer.

## Continue

Continue with [GameCI 4: Build and Deploy with MacOS](gameci-4_mac.html).
