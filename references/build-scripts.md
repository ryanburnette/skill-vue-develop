# Build Scripts

Shell scripts for build and dev workflow. Run builds in Docker.

## scripts/build

```sh
#!/bin/sh
set -eu

projectdir="$(cd "$(dirname "$0")/.." && pwd)"

docker run --rm \
  -v "$projectdir":/app \
  -v /app/node_modules \
  -e NODE_ENV=production \
  blarcos-ui \
  npx vite build
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

For dev server, run `npx vite` (inside Docker).
