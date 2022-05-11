## An Introduction
As a software engineer, I believe every modern software project should have a CI/CD pipeline.
The reasons for such are many and better explained all over the internet, so I won't be making a case for why you should be using CI/CD.
Instead, I write this series of articles with the goal of helping Game Developers with their own pipelines.
In particular, I belive that Unity game projects don't have as many resources for CI/CD as they should.
Hopefully, this guide to how I built the CI/CD pipeline for my Unity project will help you with yours.

## The Pipeline
So this is what the visualization graph for my workflow looks like:
![Test, Build, and Deploy with CGS](assets/img/cgs-workflow.png)

My workflow is 