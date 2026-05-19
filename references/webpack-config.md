# Webpack Configuration

Full webpack config patterns for a Vue SPA project.

## Base Config (webpack.config.js)

```js
var path = require('path');
var MiniCssExtractPlugin = require('mini-css-extract-plugin');
var VueLoaderPlugin = require('vue-loader/lib/plugin');

module.exports = {
  mode: 'development',
  entry: {
    vendor: [
      'vue',
      'vue-router',
      'axios',
      'date-fns',
      'lodash',
      'uuid'
    ],
    app: {
      import: './js/app.js',
      dependOn: 'vendor'
    }
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].js',
    assetModuleFilename: '[name][ext]'
  },
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      {
        test: /\.js$/,
        include: path.resolve(__dirname, 'js'),
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      },
      {
        test: /\.s[ac]ss$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
          {
            loader: 'sass-loader',
            options: {
              implementation: require.resolve('sass')
            }
          }
        ]
      },
      {
        test: /\.html/,
        type: 'asset/resource'
      }
    ]
  },
  resolve: {
    alias: {
      vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.js')
    }
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: 'style.css'
    }),
    new VueLoaderPlugin()
  ],
  devtool: 'source-map'
};
```

Key notes:
- `dependOn: 'vendor'` splits vendor libs into a separate bundle for caching
- The `include` path for babel-loader must point to where your code lives (here, `js/` not `src/`)
- SCSS pipeline always extracts to a file via MiniCssExtractPlugin, even in dev
- `devtool: 'source-map'` should be included (was missing in original project)
- Vue alias points to the full build (with template compiler) for dev

## Production Override (webpack.production.js)

```js
var path = require('path');
var merge = require('webpack-merge');
var CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
var TerserPlugin = require('terser-webpack-plugin');

var config = merge(require('./webpack.config.js'), {
  mode: 'production',
  resolve: {
    alias: {
      vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.min.js')
    }
  },
  optimization: {
    minimize: true,
    minimizer: [new CssMinimizerPlugin(), new TerserPlugin()]
  }
});

module.exports = config;
```

The base config's `mode: 'development'` is overridden by `mode: 'production'` here. The Vue alias switches to the minified build.

## PostCSS Config (postcss.config.js)

```js
module.exports = {
  plugins: [
    require('tailwindcss'),
    require('postcss-preset-env')({ stage: 0 })
  ]
};
```

## Tailwind Config (tailwind.config.js)

```js
module.exports = {
  mode: 'jit',
  content: [
    './html/*.html',
    './css/*.{css,scss,sass}',
    './js/**/*.{js,vue}'
  ],
  safelist: [
    // Add dynamically-constructed class names here
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ['Inter var', 'system-ui', 'sans-serif']
      }
    }
  }
};
```

## SCSS Entry (css/style.scss)

Only Tailwind directives, nothing else:

```scss
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## HTML Shell (html/index.html)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>App</title>
  <link rel="stylesheet" href="style.css">
  <link rel="stylesheet" href="https://rsms.me/inter/inter.css">
</head>
<body>
  <div class="app" v-cloak>
    <div v-if="ready">
      <router-view></router-view>
    </div>
    <loading-spinner v-else></loading-spinner>
  </div>
  <script src="vendor.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

Key patterns:
- `v-cloak` on the mount element hides un-rendered Vue templates on load
- `v-if="ready"` guards the entire app until root bootstrap completes
- Vendor script loads before app script (required by `dependOn` split)
- CSS loaded via `<link>` (extracted by MiniCssExtractPlugin)

## package.json Dependencies

Webpack-related packages:

```json
{
  "dependencies": {
    "vue": "^2.6.14",
    "vue-router": "^3.5.2",
    "vue-template-compiler": "^2.6.14",
    "axios": "^0.21.1",
    "webpack": "^5.74.0",
    "webpack-cli": "^4.10.0",
    "webpack-merge": "^5.8.0",
    "vue-loader": "^15",
    "babel-loader": "^8.2.2",
    "@babel/core": "^7.14.6",
    "@babel/preset-env": "^7.14.7",
    "sass-loader": "^12.1.0",
    "sass": "^1.54.9",
    "css-loader": "^5.2.6",
    "postcss-loader": "^6.1.0",
    "postcss": "^8.3.5",
    "postcss-preset-env": "^6.7.0",
    "tailwindcss": "^3.2.4",
    "@tailwindcss/forms": "^0.5.3",
    "autoprefixer": "^10.2.6",
    "mini-css-extract-plugin": "^1.6.1",
    "css-minimizer-webpack-plugin": "^3.0.2",
    "terser-webpack-plugin": "^5.1.4",
    "@ryanburnette/hash-assets": "^1.1.1",
    "style-loader": "^3.0.0"
  }
}
```

Note: No `devDependencies` separation. All packages are in `dependencies`. This is simple and works. If you want stricter separation for production builds, move build tools to `devDependencies`.

## Troubleshooting

- **Styles not updating in dev**: MiniCssExtractPlugin always extracts to a file. If the backend server caches, you may need a hard refresh. HMR for CSS would require `style-loader` instead, but that only works for dev mode.
- **Vue warnings about runtime-only build**: Make sure the Vue alias points to `vue.js` (full build with compiler) or `vue.min.js` in production, not `vue.runtime.js`.
- **Tailwind classes not appearing in production**: Add dynamic class names to the `safelist` array in `tailwind.config.js`. Tailwind's JIT compiler can't detect classes constructed at runtime.