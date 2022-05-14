# GameCI 5: Build and Deploy with MacOS

Continuing from [GameCI 4](gameci-4_linuxdeploy.html), let's examine the `Build with Mac`, `Deploy to the Mac App Store`, and `Deploy to the App Store` jobs.

## Build with Mac

{% highlight yml %}
{% raw %}
  buildWithMac:
    name: Build for StandaloneOSX
    runs-on: macos-latest
    needs: tests
    outputs:
      buildVersion: ${{ steps.build.outputs.buildVersion }}
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          lfs: true
      - uses: actions/cache@v3
        with:
          path: Library
          key: Library-buildMac-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-buildMac-
      - name: Build Unity Project
        id: build
        uses: game-ci/unity-builder@v2
        env:
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          UNITY_SERIAL: ${{ secrets.UNITY_SERIAL }}
        with:
          targetPlatform: StandaloneOSX
          buildMethod: Cgs.Editor.BuildCgs.BuildOptions
      - name: Fix File Permissions and Code-Sign
        env:
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_TEAM_NAME: ${{ secrets.APPLE_TEAM_NAME }}
          APPLE_DISTRIBUTION_CERTIFICATE: ${{ secrets.APPLE_DISTRIBUTION_CERTIFICATE }}
          APPLE_DISTRIBUTION_PASSWORD: ${{ secrets.APPLE_DISTRIBUTION_PASSWORD }}
          MAC_INSTALLER_CERTIFICATE: ${{ secrets.MAC_INSTALLER_CERTIFICATE }}
          MAC_INSTALLER_PASSWORD: ${{ secrets.MAC_INSTALLER_PASSWORD }}
          MAC_APP_BUNDLE_PATHS: Contents/PlugIns/StandaloneFileBrowser.bundle
          MAC_BUILD_PATH: ${{ format('{0}/build/StandaloneOSX', github.workspace) }}
          PROJECT_NAME: Card Game Simulator
        run: |
          bundle install
          bundle exec fastlane mac fixversion
          find $MAC_BUILD_PATH -type f -name "**.sh" -exec chmod +x {} \;
          chmod +x fastlane/sign-mac-build.sh
          ./fastlane/sign-mac-build.sh
      - name: Upload Build
        if: github.event.action == 'published' || contains(github.event.inputs.workflow_mode, 'StandaloneOSX') || contains(github.event.inputs.workflow_mode, 'Steam')
        uses: actions/upload-artifact@v3
        with:
          name: cgs-StandaloneOSX
          path: build/StandaloneOSX
      - name: Upload App
        uses: actions/upload-artifact@v3
        if: github.event.action == 'published' || contains(github.event.inputs.workflow_mode, 'StandaloneOSX') || contains(github.event.inputs.workflow_mode, 'Steam')
        with:
          name: Card Game Simulator.app
          path: build/StandaloneOSX/Card Game Simulator.app
      - name: Zip App
        uses: vimtor/action-zip@v1
        if: github.event.action == 'published'
        with:
          files: build/StandaloneOSX/
          dest: build/cgs-StandaloneOSX.zip
      - name: Upload Zip to GitHub Release
        uses: svenstaro/upload-release-action@v2
        if: github.event.action == 'published'
        with:
          repo_token: ${{ secrets.CGS_PAT }}
          asset_name: cgs-StandaloneOSX.zip
          file: build/cgs-StandaloneOSX.zip
          tag: ${{ github.ref }}
          overwrite: true
          body: ${{ github.event.release.body }}
{% endraw %}
{% endhighlight %}

## Deploy to the Mac App Store

{% highlight yml %}
{% raw %}
  deployToMacAppStore:
    name: Deploy to the Mac App Store
    runs-on: macos-latest
    needs: buildWithMac
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'StandaloneOSX'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Download iOS Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-StandaloneOSX
          path: build/StandaloneOSX
      - name: Update Release Notes
        if: github.event.action == 'published'
        env:
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: echo "$RELEASE_NOTES" > fastlane/metadata/en-US/release_notes.txt
      - name: Run Fastlane
        env:
          APPLE_CONNECT_EMAIL: ${{ secrets.APPLE_CONNECT_EMAIL }}
          APPLE_DEVELOPER_EMAIL: ${{ secrets.APPLE_DEVELOPER_EMAIL }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_TEAM_NAME: ${{ secrets.APPLE_TEAM_NAME }}
          APPSTORE_KEY_ID: ${{ secrets.APPSTORE_KEY_ID }}
          APPSTORE_ISSUER_ID: ${{ secrets.APPSTORE_ISSUER_ID }}
          APPSTORE_P8: ${{ secrets. APPSTORE_P8 }}
          MAC_BUILD_PATH: ${{ format('{0}/build/StandaloneOSX', github.workspace) }}
          MAC_BUNDLE_ID: com.finoldigital.CardGameSimulator
          PROJECT_NAME: Card Game Simulator
        run: |
          bundle install
          bundle exec fastlane mac macupload
{% endraw %}
{% endhighlight %}

## Deploy to the App Store

{% highlight yml %}
{% raw %}
  deployToAppStore:
    name: Deploy to the App Store
    runs-on: macos-latest
    needs: buildWithLinux
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'iOS'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Download iOS Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-iOS
          path: build/iOS
      - name: Update Release Notes
        if: github.event.action == 'published'
        env:
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: echo "$RELEASE_NOTES" > fastlane/metadata/en-US/release_notes.txt
      - name: Fix File Permissions and Run Fastlane
        env:
          APPLE_CONNECT_EMAIL: ${{ secrets.APPLE_CONNECT_EMAIL }}
          APPLE_DEVELOPER_EMAIL: ${{ secrets.APPLE_DEVELOPER_EMAIL }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_TEAM_NAME: ${{ secrets.APPLE_TEAM_NAME }}
          MATCH_URL: ${{ secrets.MATCH_URL }}
          MATCH_PERSONAL_ACCESS_TOKEN: ${{ secrets.CGS_PAT }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          APPSTORE_KEY_ID: ${{ secrets.APPSTORE_KEY_ID }}
          APPSTORE_ISSUER_ID: ${{ secrets.APPSTORE_ISSUER_ID }}
          APPSTORE_P8: ${{ secrets.APPSTORE_P8 }}
          USYM_UPLOAD_AUTH_TOKEN: ${{ secrets.USYM_UPLOAD_AUTH_TOKEN }}
          IOS_BUILD_PATH: ${{ format('{0}/build/iOS', github.workspace) }}
          IOS_BUNDLE_ID: com.finoldigital.CardGameSim
          PROJECT_NAME: Card Game Simulator
        run: |
          find $IOS_BUILD_PATH -type f -name "**.sh" -exec chmod +x {} \;
          find $IOS_BUILD_PATH -type f -iname "usymtool" -exec chmod +x {} \;
          bundle install
          bundle exec fastlane ios release
{% endraw %}
{% endhighlight %}

This job only runs when triggered by either a GitHub Release or a workflow dispatch with `release iOS` as the input.
This job takes the iOS Xcode project (which was built in [GameCI 3: Build with Linux](gameci-3_linuxbuild.html)) and uses fastlane to build and deploy to the App Store.

For additional details about iOS builds and deployment, refer to [the GameCI iOS docs](https://game.ci/docs/github/deployment/ios).

## Continue

If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 6: Build and Deploy with Windows](gameci-6_windows.html).
