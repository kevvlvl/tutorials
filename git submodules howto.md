# Application repository here with two git submodules

This examples hows how we integrate git repos as submodules using a local example (no remote origin)

## Config git to allow file protocol (for this POC only)

_This is only required to run this proof of concept locally_
```
git config --global protocol.file.allow always
```

## Setup folders

```
mkdir AppRepo
mkdir EnvUat
mkdir EnvProd
mkdir RemoteAppRepo
mkdir RemoteEnvUat
mkdir RemoteEnvProd
```

## Define Origin as git repos

```
cd RemoteAppRepo
git init
cd ..

cd RemoteEnvUat
git init
cd ..

cd RemoteEnvProd
git init
cd ..
```

## Define git repos

```
cd AppRepo
git init
git branch move master main
git remote add origin /home/kev/GitProjects/RemoteAppRepo
cd ..

cd EnvUat
git init
git branch move master main
git remote add origin /home/kev/GitProjects/RemoteEnvUat
cd ..

cd EnvProd
git init
git branch move master main
git remote add origin /home/kev/GitProjects/RemoteEnvProd
cd ..
```

## Create some mock files in submodule repos

```
cd EnvUat

vi values.yaml
* edit your values.yaml here and save :wq! *

git add values.yaml
git commit -m "feat: first values"
git push --set-upstream origin main
cd ..

cd EnvProd

vi values.yaml
* edit your values.yaml here and save :wq! *

git add values.yaml
git commit -m "feat: first values"
git push --set-upstream origin main
cd ..
```

## Mock some code in the App repo and add git submodules
```
cd AppRepo

vi README.md
* edit your README.md with some dummy info and save :wq! *
git add README.md
git commit -m "added readme"
git push --set-upstream origin main
```

## Add git submodules

_Now we can add the two repos as submodules_

```
git submodule add ../EnvProd EnvProd
git submodule add ../EnvUat EnvUat
```

At this point, the developer can open AppRepo and get a complete view of the files to work on

## Updating files

In the scenario where you want to update the file located in a git submodule (therefore a different git repo):

1. You must use the same git practices in that same submodule just like any git repo: Create feature branch

    1.1. Create a feature branch
    
    1.2. Do your file changes in your IDE/editor (which you already opened: AppRepo)

    1.3. Create your PR to merge your code into your main branch

2. In your AppRepo, your git repo is referencing an old commit ID of the submodule which you now updated (example: git commit ID abcd123). We now want AppRepo to reference the latest commit of the submodule, and we can do so with one command:

    2.1. Update the reference of that git submodule (example, EnvUat)
    ```
    git submodule update --init --remote ./EnvUat
    ```
    2.2. Now that we updated the submodule reference, this, like any other change, must be added to the git history using a git add, commit then push. Then go through PR process to merge on main
