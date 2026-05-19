# Build Scripts

Shell scripts for build and dev workflow. Run webpack in Docker.

## scripts/build

```sh
#!/bin/sh
NODE_ENV=production npx webpack
```

## scripts/lint

```sh
#!/bin/sh
set -e
npx eslint js/ --ext .js,.vue "$@"
npx prettier --check 'js/**/*.{js,vue}' 'css/**/*.css' '*.config.js'
```

## scripts/lint-fix

```sh
#!/bin/sh
set -e
npx eslint js/ --ext .js,.vue --fix
npx prettier --write 'js/**/*.{js,vue}' 'css/**/*.css' '*.config.js'
```

For dev server, run `npx webpack serve` or `npm start` (inside Docker).
