---
title: "Create React App 無理やり移行ガイド"
tags: ["react", "JavaScript", "TypeScript"]
date: 2024-12-12T15:31:55+09:00
---

Create React Appがどうやらもうアップデートされなくなってしまったため、既存のプロジェクトをうまいこと無理やり新しいwebpackなどに移行するためのメモ。。。

なお、以降のコマンドの実行の際は、npmではなくyarnを前提としている。npmを使用する場合はyarn.lockを適宜package.lockに読み替えること。

### 既存のnode_modulesからbabel-preset-react-app eslint-config-react-app react-app-polyfillを引き揚げてくる

最初に、既存のnode_modulesからもうアップデートされなくなってしまったコンポーネントを、上の階層に移動してくる。その後、上の階層のnode_modulesなども、一度削除する。

```bash
cd node_modules
cp -r babel-preset-react-app eslint-config-react-app react-app-polyfill ..
cd ..
rm -rf babel-preset-react-app/node_modules babel-preset-react-app/yarn.lock eslint-config-react-app/node_modules eslint-config-react-app/yarn.lock react-app-polyfill/node_modules node_modules/yarn.lock
rm -rf ./node_modules ./yarn.lock
```

### babel-preset-react-app eslint-config-react-app react-app-polyfill内のパッケージをすべてアップデートする

babel-preset-react-app eslint-config-react-app react-app-polyfillのそれぞれのフォルダに入り、package.jsonに列挙されている依存パッケージをすべて消去し、同じものを再インストールする

```bash
cd babel-preset-react-app

# delete old packages
cat package.json | jq 'del(.dependencies."babel-preset-react-app", .dependencies."eslint-config-react-app", .dependencies."react-app-polyfill")' > package.json.new
mv package.json.new package.json

# update all packages
cat package.json | jq -r ".dependencies | keys | .[]" > tmp_dep
cat package.json | jq -r ".devDependencies | keys | .[]" > tmp_dev
cat package.json | jq -r ".peerDependencies | keys | .[]" > tmp_peer
cat package.json | jq -r ".optionalDependencies | keys | .[]" > tmp_opt
cat package.json | jq 'del(.dependencies, .devDependencies, .peerDependencies, .optionalDependencies)' > package.json.new
cp package.json package.json.old
mv package.json.new package.json
yarn add $(cat tmp_dep)
yarn add --dev $(cat tmp_dev)
yarn add --peer $(cat tmp_peer)
yarn add --optional $(cat tmp_opt)
rm tmp_*

# 以降、eslint-config-react-appとreact-app-polyfill、上の階層のプロジェクトにも同様のことを行う。
```

この際、以下のようにproposalが付いたパッケージを変換するように言われる場合がある。指示に従い、プラグイン名を最新のものに変化させる。(proposalをtransformに変化させればok。decoratorsは、proposalのままでいい。)

```bash
warning babel-preset-react-app > @babel/plugin-proposal-numeric-separator@7.18.6: This proposal has been merged to the ECMAScript standard and thus this plugin is no longer maintained. Please use @babel/plugin-transform-numeric-separator instead.
warning babel-preset-react-app > @babel/plugin-proposal-optional-chaining@7.21.0: This proposal has been merged to the ECMAScript standard and thus this plugin is no longer maintained. Please use @babel/plugin-transform-optional-chaining instead.
warning babel-preset-react-app > @babel/plugin-proposal-private-methods@7.18.6: This proposal has been merged to the ECMAScript standard and thus this plugin is no longer maintained. Please use @babel/plugin-transform-private-methods instead.
warning babel-preset-react-app > @babel/plugin-proposal-nullish-coalescing-operator@7.18.6: This proposal has been merged to the ECMAScript standard and thus this plugin is no longer maintained. Please use @babel/plugin-transform-nullish-coalescing-operator instead.
```

### ソースコードの対応: proposalの付いたパッケージへの参照を修正

proposalが含まれているファイルを探して、proposalを対応するものに変化させる。主に、`babel-preset-react-app/create.js`が該当する。(decoratorsだけは、proposalのままでいい。)

```bash
rg -g "**/*.js" proposal
```

変化前:
```javascript
babel-preset-react-app/create.js

109:      // @babel/plugin-proposal-decorators when using TypeScript.
143:        require('@babel/plugin-proposal-decorators').default,
151:      // * @babel/plugin-proposal-class-properties
152:      // * @babel/plugin-proposal-private-methods
153:      // * @babel/plugin-proposal-private-property-in-object
156:        require('@babel/plugin-proposal-class-properties').default,
162:        require('@babel/plugin-proposal-private-methods').default,
168:        require('@babel/plugin-proposal-private-property-in-object').default,
174:      require('@babel/plugin-proposal-numeric-separator').default,
208:      require('@babel/plugin-proposal-optional-chaining').default,
209:      require('@babel/plugin-proposal-nullish-coalescing-operator').default,
220:            require('@babel/plugin-proposal-decorators').default,
```

変化後:
```javascript
babel-preset-react-app/create.js

109:      // @babel/plugin-proposal-decorators when using TypeScript.
143:        require('@babel/plugin-proposal-decorators').default,
151:      // * @babel/plugin-transform-class-properties
152:      // * @babel/plugin-transform-private-methods
153:      // * @babel/plugin-transform-private-property-in-object
156:        require('@babel/plugin-transform-class-properties').default,
162:        require('@babel/plugin-transform-private-methods').default,
168:        require('@babel/plugin-transform-private-property-in-object').default,
174:      require('@babel/plugin-transform-numeric-separator').default,
208:      require('@babel/plugin-transform-optional-chaining').default,
209:      require('@babel/plugin-transform-nullish-coalescing-operator').default,
220:            require('@babel/plugin-proposal-decorators').default,
```

### ソースコードの対応: package.jsonの修正

package.json内の、設定ファイルに対する参照を修正する。

```json
{
  "name": "frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    ＝＝＝省略＝＝＝
  },
  "scripts": {
    ＝＝＝省略＝＝＝
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    ＝＝＝省略＝＝＝
  },
  "devDependencies": {
    ＝＝＝省略＝＝＝
  },
  "jest": {
    "roots": [
      "<rootDir>/src"
    ],
    "collectCoverageFrom": [
      "src/**/*.{js,jsx,ts,tsx}",
      "!src/**/*.d.ts"
    ],
    "setupFiles": [
      "react-app-polyfill/jsdom"
    ],
    "setupFilesAfterEnv": [
      "<rootDir>/src/setupTests.ts"
    ],
    "testMatch": [
      "<rootDir>/src/**/__tests__/**/*.{js,jsx,ts,tsx}",
      "<rootDir>/src/**/*.{spec,test}.{js,jsx,ts,tsx}"
    ],
    "testEnvironment": "jsdom",
    "transform": {
      "^.+\\.(js|jsx|mjs|cjs|ts|tsx)$": "<rootDir>/config/jest/babelTransform.js",
      "^.+\\.css$": "<rootDir>/config/jest/cssTransform.js",
      "^(?!.*\\.(js|jsx|mjs|cjs|ts|tsx|css|json)$)": "<rootDir>/config/jest/fileTransform.js"
    },
    "transformIgnorePatterns": [
      "[/\\\\]node_modules[/\\\\].+\\.(js|jsx|mjs|cjs|ts|tsx)$",
      "^.+\\.module\\.(css|sass|scss)$"
    ],
    "modulePaths": [],
    "moduleNameMapper": {
      "^react-native$": "react-native-web",
      "^.+\\.module\\.(css|sass|scss)$": "identity-obj-proxy"
    },
    "moduleFileExtensions": [
      "web.js",
      "js",
      "web.ts",
      "ts",
      "web.tsx",
      "tsx",
      "json",
      "web.jsx",
      "jsx",
      "node"
    ],
    "watchPlugins": [
      "jest-watch-typeahead/filename",
      "jest-watch-typeahead/testname"
    ],
    "resetMocks": true
  },
  "babel": {
    "presets": [
      "react-app"
    ]
  }
}
```

このうち、`react-app`の名前がついている`eslintConfig` `jest.setupFiles` `babel`は修正が必要。

修正例:
```json
{
  "name": "frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    ＝＝＝省略＝＝＝
  },
  "scripts": {
    ＝＝＝省略＝＝＝
  },
  "eslintConfig": {
    "extends": [
      "./eslint-config-react-app",
      "./eslint-config-react-app/jest"
    ]
  },
  "browserslist": {
    ＝＝＝省略＝＝＝
  },
  "devDependencies": {
    ＝＝＝省略＝＝＝
  },
  "jest": {
    "roots": [
      "<rootDir>/src"
    ],
    "collectCoverageFrom": [
      "src/**/*.{js,jsx,ts,tsx}",
      "!src/**/*.d.ts"
    ],
    "setupFiles": [
      "./react-app-polyfill/jsdom"
    ],
    "setupFilesAfterEnv": [
      "<rootDir>/src/setupTests.ts"
    ],
    "testMatch": [
      "<rootDir>/src/**/__tests__/**/*.{js,jsx,ts,tsx}",
      "<rootDir>/src/**/*.{spec,test}.{js,jsx,ts,tsx}"
    ],
    "testEnvironment": "jsdom",
    "transform": {
      "^.+\\.(js|jsx|mjs|cjs|ts|tsx)$": "<rootDir>/config/jest/babelTransform.js",
      "^.+\\.css$": "<rootDir>/config/jest/cssTransform.js",
      "^(?!.*\\.(js|jsx|mjs|cjs|ts|tsx|css|json)$)": "<rootDir>/config/jest/fileTransform.js"
    },
    "transformIgnorePatterns": [
      "[/\\\\]node_modules[/\\\\].+\\.(js|jsx|mjs|cjs|ts|tsx)$",
      "^.+\\.module\\.(css|sass|scss)$"
    ],
    "modulePaths": [],
    "moduleNameMapper": {
      "^react-native$": "react-native-web",
      "^.+\\.module\\.(css|sass|scss)$": "identity-obj-proxy"
    },
    "moduleFileExtensions": [
      "web.js",
      "js",
      "web.ts",
      "ts",
      "web.tsx",
      "tsx",
      "json",
      "web.jsx",
      "jsx",
      "node"
    ],
    "watchPlugins": [
      "jest-watch-typeahead/filename",
      "jest-watch-typeahead/testname"
    ],
    "resetMocks": true
  },
  "babel": {
    "presets": [
      "./babel-preset-react-app"
    ]
  }
}

```

### ソースコードの対応: 各所での`react-app`の名前がついている参照の修正
上の階層にある設定ファイル(`./config`)や、先ほど`node_modules`から引き揚げてきたパッケージ内での、`react-app`の名前がついている部分の参照も修正する必要がある。

```
rg -g "**/*.js" "react-app"
```

修正前:
```javascript
eslint-config-react-app/base.js
36:      presets: [require.resolve('babel-preset-react-app/prod')],

config/webpack.config.js
36:const babelRuntimeEntry = require.resolve('babel-preset-react-app');
412:                  'babel-preset-react-app/webpack-overrides'
416:                    require.resolve('babel-preset-react-app'),
449:                    require.resolve('babel-preset-react-app/dependencies'),
742:            extends: [require.resolve('eslint-config-react-app/base')],

config/jest/babelTransform.js
21:      require.resolve('babel-preset-react-app'),
```

修正後:
```javascript
eslint-config-react-app/base.js
36:      presets: [require.resolve('../babel-preset-react-app/prod')],

config/webpack.config.js
36:const babelRuntimeEntry = require.resolve('../babel-preset-react-app');
412:                  '../babel-preset-react-app/webpack-overrides'
416:                    require.resolve('../babel-preset-react-app'),
449:                    require.resolve('../babel-preset-react-app/dependencies'),
742:            extends: [require.resolve('../eslint-config-react-app/base')],

config/jest/babelTransform.js
21:      require.resolve('../babel-preset-react-app'),
```

### web-vitalsの型の修正

web-vitalsを最新版にした場合、型や変数の名前・定義の変更があるので、以下のように変更する。(別にこれに関しては前のバージョンのままでも特に問題はなさそう)

```javascript
import { CLSMetric, FCPMetric, LCPMetric, TTFBMetric } from "web-vitals";

const reportWebVitals = (onPerfEntry?: (metric: CLSMetric | FCPMetric | LCPMetric | TTFBMetric) => void) => {
  if (onPerfEntry && onPerfEntry instanceof Function) {
    import("web-vitals").then(({ onCLS, onFCP, onLCP, onTTFB }) => {
      onCLS(onPerfEntry);
      onFCP(onPerfEntry);
      onLCP(onPerfEntry);
      onTTFB(onPerfEntry);
    });
  }
};

export default reportWebVitals;
```

### 終わり

以上で設定は終了となる。
今回のように、`babel-preset-react-app`といったモジュールのインポートをjsファイルに相対パスとして書くのではなく、package.jsonに相対パスとして書く場合は、`../`や`./`に参照を修正する部分は必要ないはず。

