# GitHub Actions CI/CD for ASP .NET and Azure

# Continuous  Integration (CI)
Create a new GitHub repo.

Clone the repo to your local computer.

Create a brand new .net webapi in that folder (you will start adding code to the repo). Add some dummy unit tests (just to check if everything is working fine in GitHub Actions).
> E.g. Assert.True(1 == 1);

Open the folder with VS Code.

Add a `.gitignore`file.  Go to [gitignore.io](https://www.toptal.com/developers/gitignore) website to add the contents of your interest. As an alternative, you can use the VS Code extension (once installed, press F1 and write gitignore and select Visual Studio in case your webapi was created using Visual Studio).
```
...
# User-specific files
*.rsuser
*.suo
*.user
*.userosscache
*.sln.docstates

# User-specific files (MonoDevelop/Xamarin Studio)
*.userprefs
...
```

Add all the code to the repo (in VS Code itself, Changes (+), write a message, commit, push).

Add GitHub Actions VS Code extension. This will help in writting workflow files for GitHub Actions.

Create a new folder `.github\workflows` and add there a .yml file with any name you want.
Fill the YAML file with the below code:
```yaml
name: dotnet package

on: 
  push:
    branches:
	  - main

jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      matrix:
        dotnet-version: ['6.0.x' ]

    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET Core SDK ${{ matrix.dotnet-version }}
        uses: actions/setup-dotnet@v2
        with:
          dotnet-version: ${{ matrix.dotnet-version }}
      - name: Install dependencies
        run: dotnet restore
      - name: Build
        run: dotnet build --configuration Release --no-restore
      - name: Test
        run: dotnet test --no-restore --verbosity normal
```
> Note: for more info of workflows in .NET [here](https://docs.github.com/es/actions/automating-builds-and-tests/building-and-testing-net)
> Note: there are thousands of GitHub Actions in the marketplace. For instance, you can add SonarCloud Scan to check the code quality for free (if public repo).

Add the new files to git and push to GitHub and you will see now actions are running you workflows. 

Add a status badge to you main README.md file.

## Simulate an issue and fix
Change your dummy test to fail.
> Assert.True(1 == 0);

Commit and push and you will see badge is failing.

Open a new issue in GitHub with title _Build broken_ and add some comments. You can assign the issue to someone (or yourself), add labels (bug, documentation, question, etc.). Create the new issue.

The new issue will appear (Build broken #1).

Now, fix the test to pass. When commiting the changes, use the title `Fix unit test, resolves #1`. The **#1 is important** because GitHub will automatically match the push to that issue number (and it will close the issue). Of course, your "fix" could fail again and you would need to add a new issue #2 and again try to fix it. This is the best practice.

Continuous  Integration (CI) ends here.

# Continuous  Deployment (CD)
You can add more steps to you CI workflow to pick the generated CI assets/artefacts and use them to deploy your code to Azure or another cloud.
