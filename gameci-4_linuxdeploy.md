# GameCI 4: Deploy with Linux

Continuing from [GameCI 3](gameci-3_linuxbuild.html), let's examine the `Deploy to the Google Play Store` and `Deploy to the Web via GitHub Pages` jobs.

## Deploy to the Google Play Store
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

This job only runs when triggered by either a GitHub Release or a workflow dispatch with `release Android` as the input.
Note that we use `fastlane/metadata/android/en-US/changelogs/default.txt` to publish the release notes.
For more details, refer to [the GameCI Android deployment docs](https://game.ci/docs/github/deployment/android).

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
        uses: actions/checkout@v3
      - name: Download WebGL Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-WebGL
          path: build/WebGL
      - name: Copy the WebGL build artifacts to the GitHub Pages directory
        env:
          WEBGL_BUILD_PATH: ${{ format('{0}/build/WebGL', github.workspace) }}
          WEBGL_PAGES_PATH: ${{ format('{0}/docs/WebGL', github.workspace) }}
        run: find $WEBGL_BUILD_PATH -type f -name "**WebGL.*" -exec cp {} $WEBGL_PAGES_PATH \;
      - name: Deploy to GitHub Pages
        uses: stefanzweifel/git-auto-commit-action@v4
        with:
          branch: main
          file_pattern: docs/**
          commit_message: Deploy to GitHub Pages
{% endraw %}
{% endhighlight %}

This job only runs when triggered by either a GitHub Release or a workflow dispatch with `release WebGL` as the input.
All this jobs does is copy the WebGL artifact to the correct location in the `/docs` folder of [GitHub Pages](https://pages.github.com/).
Once copied, simply committing the files will trigger a GitHub Pages deployment.
As setup, I did create and edit [cgs-WebGL.html](https://github.com/finol-digital/Card-Game-Simulator/blob/develop/docs/cgs-webgl.html), though work remains to make this web page look and feel nicer.

## Continue
If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 5: Build and Deploy with Mac](gameci-5_mac.html).
