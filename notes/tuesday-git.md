# Working Directory vs Staging vs History
### Working Directory
- The working directory contains the files currently stored on your computer. (git status)
- The staging area is where developers prepare the exact set of changes they want to include in the next commit. (git add filename)
- Instead of committing every change automatically, Git allows developers to carefully select which modifications belong together.

### Staging
- The staging area is where developers prepare the exact set of changes they want to include in the next commit.
- git add filename

### History
- The repository history contains all committed snapshots of the project.
- When changes move from the staging area into the repository history, Git permanently records them as a commit.
- Developers create commits using the command: git commit -m "commit message"

### Branching Rules
-  Require a pull request before merging
-  keep pull requests small and focused
-  write clear titles that describe the purpose of the change
-  include a description explaining why the change was made
-  provide instructions on how reviewers can verify the change
-  avoid mixing unrelated changes in a single pull request

### Pull Request Expectations
- PRs provide early feedback before code reaches production.
- Automated CI checks can run automatically when a PR is opened.
- Review discussions create a permanent record explaining the reasoning behind changes.
- Merge policies ensure that only reviewed code enters the shared branch.
