# GameCI 1: Intro to GitHub Actions for Unity

As a software engineer, I believe every modern software project should have a CI/CD pipeline.
The reasons for such are many and better explained all over the internet, so I won't be making a case for why you should be using CI/CD.
Instead, I write this series of articles with the goal of helping Game Developers with their own pipelines.
In particular, I belive that Unity game projects don't have as many resources for CI/CD as they should.
Hopefully, this guide to how I built the CI/CD pipeline for my Unity project will help you with yours.

## Visualzing My Workflow
A picture is worth a thousand words, so take a look at the visualization graph for my workflow:
![Test, Build, and Deploy with GameCI](assets/img/cgs-workflow.png)

Nowadays, developers have a lot of CI options (Unity Cloud Build, CircleCI, GitLab CI, and Jenkins, are just a few examples).
I chose GitHub Actions because it is tightly integrated to the GitHub repo where I already keep my open-source project, and GitHub provides free CI minutes for open-source projects like mine.
Plus, it makes this nice visualization graph.

