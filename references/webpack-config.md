# Webpack Configuration

Full webpack config for a Vue 3 SPA project. Single config file, environment-based.

## webpack.config.js

```js
'use strict';

const path = require('path');
const webpack = require('webpack');
const { VueLoaderPlugin } = require('vue-loader');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { DefinePlugin } = require('webpack');
const TerserPlugin = require('terser-webpack-plugin');
const dotenv = require('dotenv');

dotenv.config();
const isProduction = process.env.NODE_ENV === 'production';
const includeDev = !isProduction;
const siteAddress = process.env.SITE_ADDRESS;

var assetModuleFilename = '[name].[ext]';
var filename = '[name].js';
if (isProduction) {
  assetModuleFilename = '[name].[contenthash].[ext]';
  filename = '[name].[contenthash].js';
}

module.exports = {
  mode: isProduction ? 'production' : 'development',
  entry: {
    vendor: [
      'vue',
      'vue-router',
      // Add other vendor packages here
    ],
    app: {
      import: './js/app.js',
      dependOn: ['vendor'],
    },
    signin: {
      import: './js/signin/app.js',
      dependOn: 'vendor',
    },
    ...(includeDev ? { dev: { import: './js/dev/app.js' } } : {}),
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    publicPath: '/',
    filename: filename,
    assetModuleFilename: assetModuleFilename,
    clean: true,
  },
  devtool: isProduction ? 'source-map' : 'inline-source-map',
  devServer: {
    static: {
      directory: path.resolve(__dirname, 'dist'),
    },
    port: 3000,
    host: '0.0.0.0',
    allowedHosts: 'all',
    client: {
      webSocketURL: {
        protocol: 'wss',
        hostname: siteAddress,
        port: 8443,
      },
    },
    hot: false,
    historyApiFallback: true,
  },
  plugins: [
    new VueLoaderPlugin(),
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, 'html/index.html'),
      filename: 'index.html',
      inject: 'body',
      chunks: ['vendor', 'app'],
    }),
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, 'html/signin/index.html'),
      filename: 'signin/index.html',
      inject: 'body',
      chunks: ['vendor', 'signin'],
    }),
    ...(includeDev
      ? [
          new HtmlWebpackPlugin({
            template: path.resolve(__dirname, 'html/dev/index.html'),
            filename: 'dev/index.html',
            inject: 'body',
            chunks: ['dev'],
          }),
        ]
      : []),
    new MiniCssExtractPlugin({
      filename: isProduction ? '[name].[contenthash].css' : '[name].css',
    }),
    new DefinePlugin({
      __VUE_OPTIONS_API__: JSON.stringify(true),
      __VUE_PROD_DEVTOOLS__: JSON.stringify(!isProduction),
      __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: JSON.stringify(!isProduction),
    }),
  ].filter(Boolean),
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
      },
      {
        test: /\.css$/,
        use: [MiniCssExtractPlugin.loader, 'css-loader', 'postcss-loader'],
      },
    ],
  },
  resolve: {
    alias: {
      vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.esm-bundler.js'),
    },
  },
  optimization: {
    minimize: isProduction,
    minimizer: [
      new CssMinimizerPlugin(),
      new TerserPlugin({
        terserOptions: {
          format: {
            comments: false,
          },
          keep_fnames: true,
          keep_classnames: true,
        },
        extractComments: false,
      }),
    ],
  },
};
```

Key notes:
- `dependOn: 'vendor'` splits vendor libs into a separate bundle for caching
- `contenthash` in production filenames for cache busting
- `dotenv` loads `.env` for site address and other config
- `HtmlWebpackPlugin` per entry with `chunks` filtering keeps each app isolated
- Dev entry is conditionally included via `includeDev`
- `DefinePlugin` sets Vue 3 feature flags (required for proper tree-shaking)
- Vue alias points to `vue.esm-bundler.js` (full build with template compiler)
- CSS pipeline: `postcss-loader` handles Tailwind and autoprefixing, `css-loader` resolves imports, `MiniCssExtractPlugin.loader` extracts to file
- `devServer` config with `historyApiFallback` for SPA routing

## postcss.config.js

```js
'use strict';

const tailwindcss = require('tailwindcss');

module.exports = {
  plugins: ['postcss-preset-env', tailwindcss],
};
```

## tailwind.config.js

```js
'use strict';

import forms from '@tailwindcss/forms';

export default {
  content: ['./html/**/*.html', './js/**/*.vue'],
  theme: {
    extend: {
      fontFamily: {
        sans: ['"Source Sans 3"', 'Arial', 'sans-serif'],
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeOut: {
          '0%': { opacity: '1' },
          '100%': { opacity: '0' },
        },
        fadeScaleIn: {
          '0%': { opacity: '0', transform: 'scale(0.975)' },
          '100%': { opacity: '1', transform: 'scale(1)' },
        },
      },
      animation: {
        'fade-in': 'fadeIn 0.3s ease',
        'fade-out': 'fadeOut 0.3s ease',
        'fade-scale-in': 'fadeScaleIn 0.3s ease',
      },
      colors: {
        // App-specific colors
      },
    },
  },
  safelist: ['hidden', 'opacity-0', 'opacity-100'],
};
```

## CSS Entry (css/style.css)

Tailwind directives and `@layer components` for app-wide component styles:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  input.cb-form-input {
    @apply block w-full rounded-md border-0 py-1.5 px-3 bg-white text-gray-900
           shadow-sm ring-1 ring-inset ring-gray-200 placeholder:text-gray-400
           sm:text-sm/6;
  }
  .cb-button {
    @apply rounded-md px-3 py-2 bg-gray-100 text-sm font-semibold text-gray-900
           hover:text-cogburn bg-white hover:bg-gray-100 shadow-sm ring-2
           ring-inset ring-gray-300 focus-visible:ring-cogburn;
  }
}

[v-cloak] {
  display: none;
}
```

## HTML Shells

HtmlWebpackPlugin generates the script and style tags. Templates only need the mount element:

```html
<!-- html/index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>App</title>
</head>
<body>
  <div id="app" v-cloak></div>
</body>
</html>
```

```html
<!-- html/signin/index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Sign In</title>
</head>
<body>
  <div id="app" v-cloak></div>
</body>
</html>
```

```html
<!-- html/dev/index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Dev</title>
</head>
<body>
  <div id="dev-app" v-cloak></div>
</body>
</html>
```

## package.json Dependencies

```json
{
  "dependencies": {
    "vue": "^3.5",
    "vue-router": "^4.5",
    "@vue/compiler-sfc": "^3.5",
    "vue-loader": "^17",
    "webpack": "^5",
    "webpack-cli": "^5",
    "webpack-dev-server": "^5",
    "css-loader": "^7",
    "css-minimizer-webpack-plugin": "^7",
    "dotenv": "^16",
    "html-loader": "^5",
    "html-webpack-plugin": "^5",
    "mini-css-extract-plugin": "^2",
    "postcss": "^8",
    "postcss-loader": "^8",
    "postcss-preset-env": "^10",
    "prettier": "^3",
    "tailwindcss": "^3",
    "terser-webpack-plugin": "^5"
  },
  "devDependencies": {
    "eslint": "^10",
    "eslint-plugin-vue": "^10"
  }
}
```

Minimal. Every package is accounted for. No babel, no sass, no axios, no hash-assets.

## Troubleshooting

- **Styles not updating in dev**: MiniCssExtractPlugin always extracts to a file. If the backend server caches, hard refresh.
- **Vue warnings about runtime-only build**: Make sure the alias points to `vue.esm-bundler.js`, not `vue.runtime.esm-bundler.js`.
- **Tailwind classes not appearing in production**: Add dynamic class names to `safelist` in `tailwind.config.js`.
- **Dev entry in production build**: Check `includeDev` is `false` when `NODE_ENV=production`.
