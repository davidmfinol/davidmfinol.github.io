# GameCI 1: Intro

As a software engineer, I believe every modern software project should have a CI/CD pipeline.
The reasons for such are many and better explained all over the internet, so I won't be making a case for why you should be using CI/CD.
Instead, I write this series of articles with the goal of helping Game Developers with their own pipelines.
In particular, I belive that Unity game projects don't have as many resources for CI/CD as they should.
Hopefully, this guide to how I built the CI/CD pipeline for my Unity project will help you with yours.

## My Workflow
A picture is worth a thousand words, so take a look at the visualization graph for my GitHub Actions workflow:
![Test, Build, and Deploy with GameCI](assets/img/cgs-workflow.png)

Nowadays, there are a lot of options 