>This is README.md template for new project. You can remove this section.
>
>[Check az-environment documentation](environment/README.md)

Installation
------------

1. Clone repository

2. Download env files
```shell
task env:update
```

3. Create and/or copy required directories & files with command
```shell
task env:init
```

4. Modify `.env.local` file

5. Run
```shell
task docker:compose:init
```

6. Modify `.env.compose` file

7. Run
```shell
task env:build
task app:start
task app:install
```

Running Application
-------------------

1. Run
```shell
task app:start
```

Stopping Application
-------------------

1. Run
```shell
task app:stop
```