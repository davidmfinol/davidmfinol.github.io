# GameCI 7: Conclusion

Continuing from [GameCI 6](gameci-6_windows.html), let's examine the `Deploy to the Steam Marketplace` and `Announce Release to Social Media` jobs.

## Deploy to the Steam Marketplace

{% highlight yml %}
{% raw %}
  deployToSteam:
    name: Deploy to the Steam Marketplace
    runs-on: ubuntu-latest
    needs:  buildWithWindows
    if: github.event.action == 'published' || (contains(github.event.inputs.workflow_mode, 'release') && contains(github.event.inputs.workflow_mode, 'Steam'))
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Download StandaloneWindows Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-StandaloneWindows
          path: build/StandaloneWindows
      - name: Download StandaloneLinux64 Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-StandaloneLinux64
          path: build/StandaloneLinux64
      - name: Download StandaloneWindows64 Artifact
        uses: actions/download-artifact@v3
        with:
          name: cgs-StandaloneWindows64
          path: build/StandaloneWindows64
      - name: Download App
        uses: actions/download-artifact@v3
        with:
          name: Card Game Simulator.app
          path: build/StandaloneOSX/Card Game Simulator.app
      - name: Deploy to Steam
        uses: game-ci/steam-deploy@v1
        with:
          username: ${{ secrets.STEAM_USERNAME }}
          password:  ${{ secrets.STEAM_PASSWORD }}
          configVdf: ${{ secrets.STEAM_CONFIG_VDF }}
          ssfnFileName: ${{ secrets.STEAM_SSFN_FILE_NAME }}
          ssfnFileContents: ${{ secrets.STEAM_SSFN_FILE_CONTENTS }}
          appId: 1742850
          buildDescription: v${{ needs.buildWithWindows.outputs.buildVersion }}
          rootPath: build
          depot1Path: StandaloneWindows
          depot2Path: StandaloneLinux64
          depot3Path: StandaloneWindows64
          depot4Path: StandaloneOSX
          releaseBranch: prerelease
{% endraw %}
{% endhighlight %}

***TODO***

## Announce Release to Social Media

{% highlight yml %}
{% raw %}
  announceReleaseToSocialMedia:
    name: Announce Release to Social Media
    runs-on: ubuntu-latest
    needs: [buildWithWindows, deployToAppStore, deployToGooglePlay, deployToGitHubPages, deployToMacAppStore, deployToMicrosoftStore, deployToSteam]
    if: github.event.action == 'published'
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Prepare Announcement
        id: prepare
        shell: bash
        env:
          VERSION: v${{ needs.buildWithWindows.outputs.buildVersion }}
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: |
          export ANNOUNCEMENT="Released CGS $VERSION! $RELEASE_NOTES"
          ANNOUNCEMENT="${ANNOUNCEMENT//'%'/'%25'}"
          ANNOUNCEMENT="${ANNOUNCEMENT//$'\n'/'%0A'}"
          ANNOUNCEMENT="${ANNOUNCEMENT//$'\r'/'%0D'}"
          echo "$ANNOUNCEMENT"
          echo "::set-output name=ANNOUNCEMENT::$ANNOUNCEMENT"
      - name: Discord Announcement
        env:
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
        uses: Ilshidur/action-discord@0.3.2
        with:
          args: ${{ steps.prepare.outputs.ANNOUNCEMENT }}
      - name: Twitter Announcement
        uses: ethomson/send-tweet-action@v1
        with:
          status: ${{ steps.prepare.outputs.ANNOUNCEMENT }}
          consumer-key: ${{ secrets.TWITTER_CONSUMER_API_KEY }}
          consumer-secret: ${{ secrets.TWITTER_CONSUMER_API_SECRET }}
          access-token: ${{ secrets.TWITTER_ACCESS_TOKEN }}
          access-token-secret: ${{ secrets.TWITTER_ACCESS_TOKEN_SECRET }}
{% endraw %}
{% endhighlight %}

***TODO***

## Final Thoughts

***TODO***
