# GameCI 2: Testing

Continuing from [GameCI 1](gameci-1_intro.html), let's examine the `Test Code Quality` job.

## The Code
This is the full `tests` job:
```yml
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
```

The steps for this job fall into 3 categories:
1. Unity Test Runner
2. SonarQube
3. Release Management

## Unity Test Runner
This first part is the core of any CI pipeline: the unit tests.
The code coverage for my project isn't actually that high, but I do have coverage for the most critical code paths.
And I'd like to ensure that those critical code paths are always well-tested.



## SonarQube

## Release Management

## Continue
If you have decided that you would like to read about all the jobs in order, I'd recommend continuing with [GameCI 3: Build with Linux](gameci-3_linuxbuild.html)
