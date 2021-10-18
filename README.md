# Docs

### Code Transpiling in the Browser

[Javascript Modules](https://www.notion.so/Javascript-Modules-980f5af4371340bb91e1e245aa48139e)

[Understanding Webpack, Bundling](https://www.notion.so/Understanding-Webpack-Bundling-56779ec95aed4563bb92f4707f829591)

[Transpiling/Bundling Remotely or Locally?](https://www.notion.so/Transpiling-Bundling-Remotely-or-Locally-911df6422ae54a79a602bb08caa64bf4)

[Webpack Replacement (on Browser), esbuild ](https://www.notion.so/Webpack-Replacement-on-Browser-esbuild-72ba05c3113d441a8ec82c8c0280233f)

### Implementing In-Browser Bundling

- Normally bundling does not work on browser

[Setup, esbuild-wasm ](https://www.notion.so/Setup-esbuild-wasm-f7ae0369f238475ba11606569ec60d6d)

[Bundling Issue in the browser](https://www.notion.so/Bundling-Issue-in-the-browser-f0d7984e5f9f4369b99291209cc9daf7)

[ESBuild Bundle](https://www.notion.so/ESBuild-Bundle-3a41186e24f04670a99a6a9b5c555114)

[bundling for plugin](https://www.notion.so/bundling-for-plugin-54a8a01124734b3a9383feddf255fb5c)

[Caching - localForage](https://www.notion.so/Caching-localForage-8fb4d66e81dd475cb9f4eec8d44fe33e)

[Fetch modules, transpilling and bundling (executable code)](https://www.notion.so/Fetch-modules-transpilling-and-bundling-executable-code-17d3549880fe403fb13ad1eaf9ce4f22)

# Why doing this?

- I'm building a service like codepen

# Issues 
- Some code might have advanced JS syntax in it (like JSX) that browser can't execute
- Some code might have import statements for other JS files or CSS. We have to deal with those import statements before executing the code 
=> It needs transpilling and bundling on the browser-side.

# How to solve the issues?
- On browser, bundling doesn't work, because it needs hard-disk.
- resolved the issue from browser with https://unpkg.com/


# Plugins

### unpkg-path-plugin.ts

- get path to fetch source code from pkg

```tsx
import * as esbuild from "esbuild-wasm";

export const unpkgPathPlugin = () => {
  return {
    name: "unpkg-path-plugin",
    setup(build: esbuild.PluginBuild) {
      // onResolve is going to be called whenever esbuild is trying to figure out a path to a particular module
      // Handle root entry file of 'index.js'
      build.onResolve({ filter: /(^index\.js$)/ }, () => {
        return { path: "index.js", namespace: "a" };
      });

      // Handle relative paths in a module
      build.onResolve({ filter: /^\.+\// }, (args: any) => {
        return {
          namespace: "a",
          path: new URL(args.path, "https://unpkg.com" + args.resolveDir + "/")
            .href,
        };
      });

      // Handle main file of a module
      build.onResolve({ filter: /.*/ }, async (args: any) => {
        return {
          namespace: "a",
          path: `https://unpkg.com/${args.path}`,
        };
      });
    },
  };
};
```

### fetch-plugin.ts

- fetch module and bundle

```tsx
import * as esbuild from "esbuild-wasm";
import axios from "axios";
import localForage from "localforage";

const fileCache = localForage.createInstance({
  name: "filecache",
});

// fetch modules => transpiling => bundling on the browser
export const fetchPlugin = (inputCode: string) => {
  return {
    name: "fetch-plugin",
    setup(build: esbuild.PluginBuild) {
      build.onLoad({ filter: /(^index\.js)/ }, () => {
        return {
          loader: "jsx",
          contents: inputCode,
        };
      });

      build.onLoad({ filter: /.*/ }, async (args: any) => {
        const cachedResult = await fileCache.getItem<esbuild.OnLoadResult>(
          args.path
        );

        if (cachedResult) {
          return cachedResult;
        }
      });

      build.onLoad({ filter: /.css$/ }, async (args: any) => {
        const { data, request } = await axios.get(args.path);

        const escaped = data
          .replace(/\n/g, "")
          .replace(/"/g, '\\"')
          .replace(/'/g, "\\'");
        const contents = `
              const style = document.createElement('style');
              style.innerText = '${escaped}';
              document.head.appendChild(style);
            `;

        const result: esbuild.OnLoadResult = {
          loader: "jsx",
          contents,
          resolveDir: new URL("./", request.responseURL).pathname,
        };
        await fileCache.setItem(args.path, result);

        return result;
      });

      build.onLoad({ filter: /.*/ }, async (args: any) => {
        const { data, request } = await axios.get(args.path);

        const result: esbuild.OnLoadResult = {
          loader: "jsx",
          contents: data,
          resolveDir: new URL("./", request.responseURL).pathname,
        };
        await fileCache.setItem(args.path, result);

        return result;
      });
    },
  };
};
```

# index.ts

```tsx
import * as esbuild from "esbuild-wasm";
import { useState, useEffect, useRef } from "react";
import ReactDOM from "react-dom";
import { unpkgPathPlugin } from "./plugins/unpkg-path-plugin";
import { fetchPlugin } from "./plugins/fetch-plugin";

const App = () => {
  const ref = useRef<any>();
  const [input, setInput] = useState("");
  const [code, setCode] = useState("");

  const startService = async () => {
    ref.current = await esbuild.startService({
      worker: true,
      wasmURL: "https://unpkg.com/esbuild-wasm@0.8.27/esbuild.wasm",
    });
  };
  useEffect(() => {
    startService();
  }, []);

  const onClick = async () => {
    if (!ref.current) {
      return;
    }

    // bundling
    const result = await ref.current.build({
      entryPoints: ["index.js"],
      bundle: true,
      write: false,
      plugins: [unpkgPathPlugin(), fetchPlugin(input)],
      define: {
        "process.env.NODE_ENV": '"production"',
        global: "window",
      },
    });

    // console.log(result);

    setCode(result.outputFiles[0].text);
  };

  return (
    <div>
      <textarea
        value={input}
        onChange={(e) => setInput(e.target.value)}
      ></textarea>
      <div>
        <button onClick={onClick}>Submit</button>
      </div>
      <pre>{code}</pre>
    </div>
  );
};

ReactDOM.render(<App />, document.querySelector("#root"));
```
