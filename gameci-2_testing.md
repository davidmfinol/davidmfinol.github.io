# GameCI 2: Testing

Continuing from [GameCI 1](gameci-1_intro.html), let's examine the `Test Code Quality` job.

## The Code

This is the full `tests` job:

{% highlight yml %}
{% raw %}
  tests:
    name: Test Code Quality
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          lfs: true
      - name: Cache Library
        uses: actions/cache@v3
        with:
          path: Library
          key: Library-test-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-test-
      - name: Run Unit Tests
        uses: game-ci/unity-test-runner@v2
        env:
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          UNITY_SERIAL: ${{ secrets.UNITY_SERIAL }}
        with:
          githubToken: ${{ secrets.GITHUB_TOKEN }}
      - name: Setup dotnet 5 for SonarQube
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: '5.0.x'
      - name: Set up JDK 11 for SonarQube
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: 11
      - name: Setup Unity for SonarQube
        id: setup-unity
        uses: kuler90/setup-unity@v1
      - name: Activate Unity for SonarQube
        uses: kuler90/activate-unity@v1
        with:
          unity-username: ${{ secrets.UNITY_EMAIL }}
          unity-password: ${{ secrets.UNITY_PASSWORD }}
          unity-serial: ${{ secrets.UNITY_SERIAL }}
      - name: SonarQube Analysis
        env:
          FrameworkPathOverride: ${{ steps.setup-unity.outputs.unity-path }}/../Data/MonoBleedingEdge/
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          pwd; ls -alh
          xvfb-run --auto-servernum ${{ steps.setup-unity.outputs.unity-path }} -batchmode -nographics -quit -logFile "-" -customBuildName Card-Game-Simulator -projectPath . -executeMethod Packages.Rider.Editor.RiderScriptEditor.SyncSolution
          pwd; ls -alh
          sed -i 's/<ReferenceOutputAssembly>false<\/ReferenceOutputAssembly>/<ReferenceOutputAssembly>true<\/ReferenceOutputAssembly>/g' *.csproj
          sed -i 's/\([A-Za-z0-9.-]\+csproj\)/Card-Game-Simulator\/&/g' Card-Game-Simulator.sln
          mv Card-Game-Simulator.sln ..
          cd ..
          dotnet tool install --global dotnet-sonarscanner
          dotnet sonarscanner begin \
            /o:"finol-digital" \
            /k:"finol-digital_Card-Game-Simulator" \
            /d:sonar.login="$SONAR_TOKEN" \
            /d:sonar.host.url=https://sonarcloud.io \
            /d:sonar.exclusions=Assets/Plugins/**,Assets/Mirror/**, \
            /d:sonar.cpd.exclusions=Assets/Tests/** \
            /d:sonar.coverage.exclusions=Assets/Tests/** \
            /d:sonar.cs.nunit.reportsPaths=Card-Game-Simulator/artifacts/editmode-results.xml,Card-Game-Simulator/artifacts/playmode-results.xml \
            /d:sonar.cs.opencover.reportsPaths=Card-Game-Simulator/CodeCoverage/Card-Game-Simulator-opencov/EditMode/TestCoverageResults_0000.xml,Card-Game-Simulator/CodeCoverage/Card-Game-Simulator-opencov/PlayMode/TestCoverageResults_0000.xml
          dotnet build Card-Game-Simulator.sln
          dotnet sonarscanner end /d:sonar.login="$SONAR_TOKEN"
          cd Card-Game-Simulator
      - name: SonarQube Quality Gate Check
        uses: SonarSource/sonarqube-quality-gate-action@v1.0.0
        with:
          scanMetadataReportFile: ../.sonarqube/out/.sonar/report-task.txt
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      - name: Update Release Notes
        if: github.event.action == 'published'
        env:
          RELEASE_NOTES: ${{ github.event.release.body }}
        run: |
          echo "$RELEASE_NOTES" > fastlane/metadata/android/en-US/changelogs/default.txt
          echo "$RELEASE_NOTES" > fastlane/metadata/en-US/release_notes.txt
      - name: Auto-Commit Release Notes
        if: github.event.action == 'published'
        uses: stefanzweifel/git-auto-commit-action@v4
        with:
          branch: main
          file_pattern: fastlane/metadata
          commit_message: Update Release Notes
{% endraw %}
{% endhighlight %}

The steps for this job fall into 3 categories:
1. Unity Test Runner
2. SonarQube
3. Release Management

## Unity Test Runner

This first part is the core of any CI pipeline: the unit tests.
The code coverage for my project isn't actually that high, but I do have coverage for the most critical code paths.
And I'd like to ensure that those critical code paths are always well-tested.

The [Unity Test Framework](https://docs.unity3d.com/Packages/com.unity.test-framework@1.1/manual/index.html) is the obvious option for Unity unit tests, and as a maintainer for GameCI, I recommend running tests with the [Unity Test Runner action](https://github.com/marketplace/actions/unity-test-runner).
Please refer to [the GameCI documentation website](https://game.ci/docs/github) as the best resource for how to use the Unity test runner.
After you have an understanding of the `Checkout Repository`, `Cache Library`, and `Run Unit Tests` steps, you may want to move on to SonarQube.

## SonarQube

[SonarQube](https://www.sonarqube.org/) is a code analysis tool that you may want to use to help maintain a high level of code quality for your project.
If you are interested, I recommend starting by reading the [docs for Sonar Cloud](https://docs.sonarcloud.io/).
Since we are using GitHub Actions, we can [start with Sonar Cloud for GitHub](https://docs.sonarcloud.io/getting-started/github/).
Unfortunately, Unity C# code is not eligible for Automatic Analysis, so you would have to go with CI-based Analysis.
At GameCI, we would like to eventually provide additional support for the Sonar Cloud tooling, but for now, the following process is what I am using.

The Sonar CI-based analysis scanner tool has dependencies on 3 tools: .NET 5, Java 11, and Unity.
Hence, the `Setup dotnet 5 for SonarQube`, `Set up JDK 11 for SonarQube`, and `Setup Unity for SonarQube` steps install these tools onto the GitHub Runner.
Furthermore, the Sonar scanner requies .sln and .csproj files upon which it runs the analysis.
These .sln and .csproj files should be ignored by git, and they need to be generated by Unity.
The `Activate Unity for SonarQube` step will let us use Unity to generate these .sln and .csproj files as part of the actual `SonarQube Analysis` step.

In the `SonarQube Analysis` step, the first thing we do is check for the .sln and .csproj files:
```bash
pwd; ls -alh
xvfb-run --auto-servernum ${{ steps.setup-unity.outputs.unity-path }} -batchmode -nographics -quit -logFile "-" -customBuildName Card-Game-Simulator -projectPath . -executeMethod Packages.Rider.Editor.RiderScriptEditor.SyncSolution
pwd; ls -alh
```

Note that the `customBuildName` should match your project's name, and that `Packages.Rider.Editor.RiderScriptEditor.SyncSolution` is only available with the [JetBrains Rider editor package](https://docs.unity3d.com/Packages/com.unity.ide.rider@3.0/manual/index.html).
Before Unity v2021.3, `UnityEditor.SyncVS.SyncSolution` was another method that could generate the files, but that method seems to no longer work as of v2021.3.1f1.
Furthermore, all the .sln and .csproj files will be generated in the project root folder, but the sonar tool won't run if they are all in the same folder.
So there's a hack to move the .sln to the parent directory while still keeping all the .csproj files in the project root directory:
```bash
sed -i 's/<ReferenceOutputAssembly>false<\/ReferenceOutputAssembly>/<ReferenceOutputAssembly>true<\/ReferenceOutputAssembly>/g' *.csproj
sed -i 's/\([A-Za-z0-9.-]\+csproj\)/Card-Game-Simulator\/&/g' Card-Game-Simulator.sln
mv Card-Game-Simulator.sln ..
cd ..
```

The final setup step is to install the Sonar scanner tool with `dotnet tool install --global dotnet-sonarscanner`.
With that, we can finally run the Sonar scan, remembering to `cd` back into the project root directory after we are done:
```bash
dotnet sonarscanner begin \
  /o:"finol-digital" \
  /k:"finol-digital_Card-Game-Simulator" \
  /d:sonar.login="$SONAR_TOKEN" \
  /d:sonar.host.url=https://sonarcloud.io \
  /d:sonar.exclusions=Assets/Plugins/**,Assets/Mirror/**, \
  /d:sonar.cpd.exclusions=Assets/Tests/** \
  /d:sonar.coverage.exclusions=Assets/Tests/** \
  /d:sonar.cs.nunit.reportsPaths=Card-Game-Simulator/artifacts/editmode-results.xml,Card-Game-Simulator/artifacts/playmode-results.xml \
  /d:sonar.cs.opencover.reportsPaths=Card-Game-Simulator/CodeCoverage/Card-Game-Simulator-opencov/EditMode/TestCoverageResults_0000.xml,Card-Game-Simulator/CodeCoverage/Card-Game-Simulator-opencov/PlayMode/TestCoverageResults_0000.xml
dotnet build Card-Game-Simulator.sln
dotnet sonarscanner end /d:sonar.login="$SONAR_TOKEN"
cd Card-Game-Simulator
```

The config for the Sonar scan should be self-explanatory after reviewing the [Sonar docs](https://docs.sonarcloud.io/advanced-setup/analysis-parameters/).
After your first successful Sonar scan, you'll want to configure your rules and quality gate settings in the [Sonar Cloud console](https://sonarcloud.io/project/settings).
Once you have everything configured as you like, you can use the `SonarQube Quality Gate Check` step to ensure that your code quality requirements are always met.

## Release Management

The final part of this first job involves some book-keeping surrounding GitHub Releases.
As mentioned in Part 1, we use `if: github.event.action == 'published'` to detect when a Release has been created through the GitHub UI.
This release will include the Release Notes, and we want to capture these release notes for subsequent runs.
We can simply commit the release notes to the git repo with the `Update Release Notes` and `Auto-Commit Release Notes` steps.
GitHub Actions has safeguards in place to ensure that a workflow does not re-trigger itself, so we don't have to worry about potentially causing an infinite loop of workflow runs.

## Continue

With that, we have put in unit tests, quality checks, and release management.
We're now ready to move on to actually building and deploying our game.
Continue with [GameCI 3: Build and Deploy with Linux](gameci-3_linux.html).
