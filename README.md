# Websocket Sample Package

This package aims to provide a simple to use, barebones package that starts a websocket server and waits for incoming connections. If a connection is established, the package will forward any message sent by the Editor to the websocket client, and will execute any command received from the client.

## Installation

1.  Clone the repository
2.  Run `npm i` in the root folder
3.  Run `npm run build` to build the neccessary files
4.  In the Editor at the Package Manager panel, either Approve the package at the top of the list if possible or use the `+ Add external package` button to add the **root** path of the package (for example, `C:\Users\...\package-websocket`)

## Usage

- Use the provided `Parameter Set` action block to set a parameter to a value. When the action is called (such as at a Button or Encoder event), the parameter id and parameter value is sent to a websocket client (default port is `9834`).
- Send a JSON object stringified over the Websocket connection to execute a LUA function on the module. The object must have a field `type` with `execute-code` set, and a `script` field with the LUA code. This is practically either a call to a LUA function defined on the module or a LUA code that is compatible with the modules (as can be seen in the `Export configuration` window, which is reachable by clicking the small export button in the bottom right corner of the Editor).
- An active window listening is also available in the package. With the current configuration, when enabled, the websocket action will only execute when Firefox is in focus. Change this in `index.js` at the `activeWindowTitle` field (or remove it completly from the Preference component).
- See `websocket-client-example.js` for a simple client that logs every message to the console log and executes a `print` command on the modules with the press of any keyboard key.

## Customization

- Package ID
  Every package is identified by a unique ID. If you plan to share your package or develop multiple websocket based package, it is neccessary to change it to something unique. Any valid alphanumeric string should be valid. If you have chosen a good package name, it should be updated in `package.json` file at the field `name`. The following places should also be updated to reflect the change in ID:
  - `components/Preferences.svelte`: This component is shown on the left sidebar when the package is loaded. Update the `tag` at `customElement`. Also update the ID at the `createPackageMessagePort()` function call, which returns a `MessagePort` to the main package code. Overwrite `package.json` : `preferenceComponent` with the value given at the `tag`.
  - `components/ParameterSetAction.svelte`: This component is shown when the corresponding action is expanded in the configurator panel. Change the `tag` at `customElement`. Change the package ID at the `actionRegex` and `actualCode` fields.
  - `index.js`: Inside `createAction()`, the category field should reflect the target of the package (this is used when grouping the actions inside the action picker). Under this, at the call of the `createAction()` function, update values:
    - `short`: This is a unique ID of the action. For a package, it always starts with `x`, followed by at least 2 characters representing the package (such as `ws` for WebSocket) and a unique identification string for the action.
    - `displayName`: The name of the action shown to the user
    - `defaultLua`: The default LUA code that is represented by the action. The format should match to the string set at `components/ParameterSetAction.svelte`.
    - `actionComponent`: The `tag` set for `components/ParameterSetAction.svelte`.
- Package info
  - `package.json` also contains additional info values related to the package.
    - `version`: The current version of the package. If the package is released over Github and Github releases are provided, the Editor can detect newer version and update the package if the user wants to.
    - `description`: Shown in the package manager panel as the name of the package.
    - `grid_editor`: Contains Editor specific values
      - `componentsPath`: The path to the generated web components that are loaded into the Editor. The default should be fine unless the some part of the `components` project is modified.
      - `preferenceComponent`: The tag of the preference web component. This should be unique among the packages.
      - `shortDescription`: Shown under the name of the package in the package manager panel.
      - `mainIcon`: The main logo of the package, shown in the package manager panel. Ususally a colored logo. Must be an image that can be loaded by an `<img>` tag, ususally SVG.
      - `menuIcon`: The logo shown in the left navigation bar. Ususally a white outlined logo. Must be an image that can be loaded by an `<img>` tag, ususally SVG.

## Package Development Overview

Packages for the Grid Editor generally consists of two main parts similarly to how Electron apps are sturctured:

- The main JavaScript module containing the logic of the package. The code runs in a NodeJS environment with no special restrictions. The package can communicate with the Editor through an object, usually referred to as `controller`.
- The web components shown in the Editor. Two types are supported: the preference component, which is responsible for handling general package related settings and is shown in the left main panel, and the action components, which can be added to the module elements in the configuration panel. These run in the Electron renderer process, see [here](https://www.electronjs.org/docs/latest/tutorial/process-model#the-renderer-process) for more details.

## Package main code

Whenever a package is enabled, Editor loads the folder as a NodeJS module (using `require`). In most cases, this means that the file specified in `package.json`:`main` gets loaded. This script should export the following functions (each one being an async function):

- `loadPackage(gridController, persistedData)`: This function is called at the moment a package is enabled (either by direct user interaction, at the start of the Editor application if it was left enabled or when a package download finishes). The `gridController` object can be used to communicate with parts of the Editor (see later), while `persistedData` is an object containing data that have been saved in a previous run (also see later). If no data has been saved, `persistedData` is null.
- `unloadPackage()`: Called when the package is disabled (either by the package manager or an uncaught exception) or when the Editor closes. Unsubscribe from everything, clear every timeout and interval, and remove every action added by the package.
  **NOTE:** Every package runs in the same global process (which is separate from the main Electron process). Forgetting to clear some resource **will** result in memory and resource leak until the Editor is fully closed.
- `addMessagePort(port, senderId)`: Called when another component wants to establish commincations by using the `createPackageMessagePort()` function (see at the Web Components section). This is either the Preference component or an action component in rarer cases. `port` is the `MessagePort` that can be used to communicate, while `senderId` is a string identifier to help differentiate between different components (assigned at the `createPackageMessagePort()` call).
- `sendMessage(args)`: Called when the Editor wants to communicate with the package. Two communication exists:
  - The module executes the `package_send(packageId, arg1, arg2, ...)` (`gps` for short) function. The args is always an array containing the parameters of the function call).
  - Another package wants to communicate with this package (see later). The args is an object in this case. The `senderPackageId` identifies the package sending the message.

The `gridController` object received in `loadPackage` is crucial to the operation of the package and should be stored in a variable. This object currently exposes a single function call, `sendMessageToEditor(message)`. (This will probably be upgraded in the future in a backwards-compatible way)
The `message` in `sendMessageToEditor` is a multi-purpose object that can represent a number of commands. The commands are differentiated by the `type` field of the `message`. The content of the command messages are the following:

- `send-message-to-package`: Send a message to another package.
  - `targetPackageId`: string, package identification
  - `message`: JSON object
- `create-window`/`close-window`: Experimental API to manage windows, used mainly to show overlays. Not ready for public usage, contact support if you need it for your usecase.
- `persist-data`: Persists data for a package. This can be used to store settings or cache data between Editor runs. Only the lates data packet is persisted. Received by the package when it loads with the call of `loadPackage()` function as the second argument.
  - `data`: The JSON stringifyable object to be stored
- `execute-lua-script`: Executes the given LUA script on either one specific module or all modules
  - `script`: string, The LUA code to be executed
  - `targetDx`,`targetDy`: int, The address of the module that should execute the code. Runs on all if undefined.
- `add-action`: Provides a new action to the editor, which can be used in the module config configurator.
  - `info`: The descriptor object of the action (kinda redundant, might be flattened in the future)
    - `actionId`: int, Package level **unique** number identifying the action. This is used when removing the action at the end of the package lifecycle.
    - `short`: string, A **global unique** ID of the action. For a package, it always starts with `x`, followed by at least 2 characters representing the package (such as `ws` for WebSocket) and a unique identification string for the action.
    - `displayName`: string, The name of the action shown to the user
    - `defaultLua`: string, The default LUA code that is represented by the action. The format should match to the string set at `components/ParameterSetAction.svelte`.
    - `actionComponent`: string, The `tag` set for the corresponding web component that can be used to configure the action. If no configuration is required, it can be empty. See `toggleable`.
    - `rendering`: string, Should be set to `standard` representing a normal action block. In rare cases, `modifier` can be used to create a nestable action, such as the If-Else or the Button Press-Release action, but this has not been tested to work from packages. Contact support if your use-case requires it.
    - `category`: string, Allows actions to be grouped under a label in the Action Picker panel. Can be any custom string.
    - `color`: string, HEX code of the color to be used as the background for the action in the picker panel.
    - `icon`, `blockIcon`: string, SVG code of the icon to be shown respectively in the action picker panel and the block component.
    - `selectable`: boolean, Whether the action block can be selected. Should be `true`.
    - `movable`: boolean, Whether the action block can be moved. Should be `true`.
    - `hideIcon`: boolean, Hides the icon of the action, should be `false`.
    - `type`: string, Allows different types of actions, should be `single` for packages.
    - `toggleable`: boolean, Whether the action can be opened to an editable state. If the action has a corresponding action block, it should be `true`. If the action doesn't have any parameters (such as an initialization block), it should be `false`.
- `remove-action`: Remove a previously added action.
  - `actionId`: int, The package level ID of the action.
- `change-page`: Changes the currently selected page of the modules.
  - `num`: int, The target page number.
- `show-message`: Displays a toast message in the Editor.
  - `message`: string, The text to show to the user.
  - `type`: string, Can be `success`, `alert`, `progress`, `fail`.
- `debug-error`: Logs the parameters into the console log of the Electron Renderer process.

## Web Components

Web components are used to allow users to interact with the package. We have only used Svelte to generate the web components, but any framework could be used that can output standard web components. Since web components run in the Electron renderer process, they are sandboxed and have limited acces to the user's computer. The two exceptions are the following:

- If a file that is bundled with the package is needed in the web component (such as an image), the special `package://<packageId>/` path will allow you to load any file that is in the package folder.
  For example, if I want to reach the image `images/test.png` inside the `cool-package` package, I can set the source URL to be `package://cool-package/images/test.png`
- When two-way communication is required, the web component can call the globally defined `createPackageMessagePort(packageId, senderId)` function. The Editor will create a `MessageChannel` and returns one of the `MessagePort` to the caller of the function, while the other port will be transported through multiple layers to finally reach the package's `addMessagePort(port, senderId)` function. These are ordinary JS `MessagePort` objects with their respective lifecycles and behaviors.

Note that the NodeJS package code can only receive a message port, it can **never** initiate a `MessagePort` connection. This is by design: while the NodeJS code is most likely to be enabled over the whole lifecycle of the Editor, the Web Components will appear and disappear based on what the user is currently doing. As such, the NodeJS code has no idea when a given component is shown or what data is required.
Due to this design pattern, it is recommended to think as if the NodeJS code is a server while the web components are clients. Whenever something is needed, the web component must initiate the connection, send the request (either the data or a request for data from the NodeJS side) and wait for a response message from the NodeJS code. After the response, the component may close the connection or keep it open for further requests. If the connection is used for multiple requests, never forget to **close** the port when the web component is destroyed. In Svelte, this is done by returning an unsubscribe function at the end of `onMount`, but other frameworks should have a similar solution.

The Editor does not care how the web components source file is created. All that is needed is a path towards the one JSON file that will register every needed component, set in `package.json`->`grid_editor`->`componentsPath`.
In our packages, we accomplish this by having a sub-project inside the package folder called `components`. This a standard Svelte project configured for web component build. Each web component is a file inside the the `src` folder, while the `src/main.js` file contains the imports for every web component that must be exported during a build. Any ususal frontend based NPM dependency can be used in this project.

### Preference component

The Preference web component is shown in the left panel if the user select the package. The main purpose is to show the status of the package (connection status) and provide ways to set package related settings. In most cases, a `MessagePort` should be requested when the component is created and closed when the component is destroyed.
In our packages, a `clientStatus` message is usually the first message received over the `MessagePort`. Note however that the web component and the package code is both in the hands of the package developer. As such, there are no requirements about the commincation protocol, it's all up to you. Don't forget to use the `senderId` parameter in the `createPackageMessagePort()` function to signal to the NodeJS code who it communicates with.

### Action component

The Action components are one of the key parts of any package: they allow users to add different actions to their modules (the Active Window package does not have any action associated with it, but that's more of an exception). As such, they are heavily integrated with the Editor.

The job of an action component is to receive the LUA code that is being represented, parse it, and show the resulting data in an easy-to-use component for the user. When one of the parameters is modified, it must also notify the Editor about the change.

Due to the specification of web components, there is no easy way to handle communication between the Editor and the web components. From multiple different solutions, we have chosen native JS events that can bubble through the web component - parent barrier to achieve the communication. However, event bubbling can only happen from child element to parent element, but we need two-way communcation. To accomplish this, we have reached the following architecture after several iterations:

1.  Define a function that will parse the incoming LUA code into separate parameters. It is highly recommended to do it with a regex (see later). In our code, this is usually named `handleConfigUpdate()`
2.  When the component is mounted, send a bubbling event with the type `updateConfigHandler`, with the detail object having a `handler` field that contains a reference to our `handleConfigUpdate()` function.
3.  The Editor will catch this event, and call the included function reference with the initial LUA code.

Afterwards whenever there is a change with the code, the Editor will call the saved reference.

An example:

```js
const event = new CustomEvent("updateConfigHandler", {
  bubbles: true,
  detail: { handler: handleConfigUpdate },
});
ref.dispatchEvent(event);
```

When the user changes a parameter in the action, an bubbling event with the type `updateCode` should be dispatched, with the details containing the new code in the `script` field, as such:

```js
const event = new CustomEvent("updateCode", {
  bubbles: true,
  detail: { script: String(code) },
});
if (ref) {
  ref.dispatchEvent(event);
}
```

The LUA code that is being parsed is typically a call to `gps` (short code for `package_send`), which will send a message to the respective package's `sendMessage()` function.

The regex that parses the code should follow the following format:

```js
const actionRegex = /^gps\("package-websocket", "*(.*?)", (.*?)\)$/;
```

The parentheses represent the parameters that must be parsed.
**Notice the difference between the first and second parameter!** While both parameters will be parsed as a string, there is a fundamental difference between how the code is executed due to the usage of the straight quotes in the regex.

A typical call to this function would look like this: `gps("package-websocket", "volume", self:get_auto_value())` (this is the default LUA code in the package). This will send the array `["volume", <current value of element>]`. When the `gps` code is executed on the module, we want the first parameter to be a string constant (since with this action, we want to set the volume of some program), but we want the second parameter to be a LUA executable code that can get the current value of the element. This is why we need double quotation marks in the first parameter and don't need it in the second parameter.

Similarly, when we construct the code using string interpolation, we want the quation marks in the right places as such:

```js
function actualCode() {
  return `gps("package-websocket", "${parameterId}", ${parameterValue})`;
}
```

With these defined, we can take a look at how to handle the code changes in the web component.
A typical implementation of the `handleConfigUpdate()` function looks like this:

```js
function handleConfigUpdate(config) {
  if (currentCodeValue != config.script) {
    currentCodeValue = config.script;
    const match = config.script.match(actionRegex);
    if (match) {
      parameterId = match[1] ?? "";
      parameterValue = match[2] ?? "";
      console.log({ parameterId, parameterValue });
      isInitialized = true;
    }
  }
}
```

The first if statemant checks whether there is any difference between the new and old code. If there is, we parse the code and update the corresponding variables. **Note** the `isInitialized` variable: unfortunately, there is a race condition between the action updating the code inside the Editor and the Editor sending the initial code that must be parsed. As such, this flag is used to only allow updating the code after we have received the initial code, as can be seen in the following function:

```js
function updateCode() {
  if (!isInitialized) return;

  var code = actualCode();
  if (currentCodeValue != code) {
    currentCodeValue = code;
    const event = new CustomEvent("updateCode", {
      bubbles: true,
      detail: { script: String(code) },
    });
    if (ref) {
      ref.dispatchEvent(event);
    }
  }
}
```

## Development flow

While there are still improvements to be made with regards to the tooling of package development, there are a few things that can help with quickly checking the modifications made in the package code.
As a general rule:

- If there is a change in the NodeJS code of the package, a `Force restart` in the package manager panel **should** be enough to reload the package and dependencies with the new code.
- If there is a change in the components code, **after rebuiling the component project** a frontend restart is needed on the Editor side. This practically means either an app restart or an Electron restart (Shift+R or Command+R)

A hot reload functionality is also provided within the Editor. To activate it, enter the Preferences, select the Developer Settings, and check the Package Developer flag at the bottom of the panel. This will open up a Websocket client on the `9000` port ready to accept commands from the package.

If a new package tries to communicate with the websocket, the package will automatically be added to the top of the package manager list, where the communication can either be approved or rejected. Packages added with the `+ Add external package` button are automatically approved for communication.

If the web components should be reloaded, the following JSON string should be sent through the websocket:

```js
JSON.stringify({
  type: "developer-package",
  event: "components-build-complete",
  id: <packageId>,
  rootPath: path.resolve(__dirname, ".."), //root path of the package
})
```

In our official packages, a hook is already provided in the `components/vite.postbuild.ts` file that is called when the `components` build completes. Running `npm run dev` from either the components or root folder (which will pipe through to the `components` `dev` command) will listen for any changes in the source code, rebuild on change and call the hook.
**Note** JS files are cached when imported in the Electron renderer process, even if the underlying source code is modified. Because of this, some voodoo magic is required to bust through the cache and redefine the web components. While this worked in our testing, it is possible there are edge-cases when the component is not updated. Please report these to the support team!

While not used by the official packages, a similar hook is provided for the NodeJS code changes. If the build system supports hooks, send a message with the same parameters as the components reload, with the `event` field having the value `package-build-complete`. This is equivalent to disabling and enabling the package. We have not tested this thoroughly, contact the support team if any problem occurs!

Currently most of our projects only need to build the `components` project. If any other build step is required, it is recommended to add them to the root folder's `package.json`. Our projects are configured such that all of the `build:<component>`, `dev:<component>` and `install:<component>` scripts are executed when the `npm run build`, `npm run dev` and `npm i` are run from the root folder. This reduces the number of terminals needed for package development.

## Logs

Currently, package logs and errors are sent to two different locations based on where the log is executed.

- For the web components, logs are always shown in the Electron renderer dev tools, which can be opened with either Ctrl+Shift+i or Command+Options+i.
- For debug messages sent from the NodeJS code with the `controller.sendMessageToEditor`, they are also displayed in the renderer dev tools.
- Any other log from the NodeJS code is shown in the Electron main process terminal. While this is reachable from the Editor's folder (`AppData\Roaming\Grid Editor\logs` for Windows), it is usually more straightforward to launch the Editor executable from a terminal. The logs will be shown in the terminal window.
