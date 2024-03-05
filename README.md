Installation
------------

1. Clone repository
2. Change branch to ```project-initializer```
```shell
git checkout project-initializer
```
2. Create and/or copy required directories & files with command
```shell
task env:init
```
2. Modify `.env.local` file
3. Build docker containers and create a new Symfony project with command
```shell
task env:new-project
```