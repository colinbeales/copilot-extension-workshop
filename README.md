# copilot-extension-workshop

## 1. Create a repo

| Field           | Value                          |
| --------------- | ------------------------------ |
| Repository name | <e.g. ingredients-demo-js> |
| Readme          | Add                            |
| .gitignore      | Node                           |

## 2. Create devcontainer.json for Codespace details

- Using the Web UI for the new repo add a new file /.devcontainer/devcontainer.json

```JSON
{
  "name": "Node.js & TypeScript",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:1-22-bookworm",
  "customizations": {
    "vscode": {
        "extensions": [
            "GitHub.copilot",
            "GitHub.copilot-chat"
        ]
    }
  },
  "postCreateCommand": "npm install"
}
```
- Commit this file to the repo so it can be used for your Codespace

## 3. Start Codespace

- Start a the codespace now it defines your dev environment and VS Code Copilot Extension

## 4. Add ./packages.json

- In VS Code add a new file ./packages.json and add the following content

```JSON
{
    "name": "ingredients-extension-js",
    "version": "1.0.0",
    "description": "A JavaScript project with Express and Azure AI Inference",
    "main": "index.js",
    "type": "module",
    "scripts": {
      "start": "node index.js"
    },
    "dependencies": {
      "@copilot-extensions/preview-sdk": "^4.0.2",
      "@azure-rest/ai-inference": "^1.0.0-beta.2",
      "@azure/core-auth": "^1.3.0",
      "express": "^4.21.0"
    },
    "author": "",
    "license": "ISC"
}
```

- In the terminal call `npm install`

## 5. Implement a simple Express based API

- In VS Code add a new file `index.js`

- Try using Copilot Chat to generate your API. Use a prompt like, "Create an Express based API with a GET endpoint for / and /callback and a POST endpoint for /agent. Each endpoint should log a call to the console.

Some notes (of what you want from Copilot (or paste in the below linked example gist code):
- Should be Async
- Should use JSON Middleware i.e. `app.use(express.json());`
- Has a GET / - To prove this API works
- Has a GET /callback - Output from a callback on registration
- Has POST /agent - Where our Copilot Extension will be called from

Example content for [index.js](https://gist.github.com/colinbeales/28141460eb151d867a3d4718f1a067ff#file-index-js)

## 6. Run API in the Codespace

- Run `node index.js`

- Make Codespace public

- Show ports - Copy the public URL

- Show `/` and `/callback` endpoints functioning.

## 7. Register Copilot Extension as a GitHub App

- Head to [Settings->Developer Settings->GitHub Apps](https://github.com/settings/apps) and select 'New GitHub App' button

- Fill in:

| Field                             | Value                                       |
| --------------------------------- | ------------------------------------------- |
| GitHub App name                   | <e.g. ingredients-demo-js>                  |
| Description                       | takes ingredients to return recipes         |
| Homepage URL                      | [pasted from public codespace URL]          |
| Callback URL                      | [pasted from public codespace URL]/callback |
| WebHook                           | Change to [inactive]                        |
| Account Permissions->Copilot Chat | Read-only                                   |

Click 'Create GitHub App' button.

- Fill in Copilot tab:

| Field                 | Value                                          |
| --------------------- | ---------------------------------------------- |
| App Type              | Agent                                          |
| URL                   | [pasted from public codespace URL]/agent       |
| Inference Description | <e.g. list ingredients for recipe suggestions> |

Click 'Save' to save these changes.

- Head to 'Install App' tab and click Install the App.

## 8. Add code to get ingredients from body (and debug using CLI):

- Install CLI Debug Tool `gh extension install github.com/copilot-extensions/gh-debug-cli` onto GitHub CLI

- Add code to get ingredients:

```js
const { messages } = req.body;
const ingredients = messages[messages.length - 1].content;

console.log(`Ingredients: ${ingredients}`);

res.end();
```

- Run the webserver: `node index.js` (ensure public)

- Run the debug client: `gh debug-cli --url <pasted from public codespace URL>/agent`

- Check ingredients taken from the last message

## 9. Get code from GitHub Models and integrate

- Head to [GitHub Marketplace for Models]()

- Pick a model, go to playground and select code.

Take meaningful sections from the console app

Add imports to top of the file:

```JS
import ModelClient from "@azure-rest/ai-inference";
import { AzureKeyCredential } from "@azure/core-auth";

const token = process.env["GITHUB_TOKEN"];
```

For the agent section copy this after the ingredients variable is output:

```JS
    const client = new ModelClient(
        "https://models.inference.ai.azure.com",
        new AzureKeyCredential(token)
    );

    const response = await client.path("/chat/completions").post({
        body: {
            messages: [
                { role: "system", content: "" },
                { role: "user", content: "Can you explain the basics of machine learning?" }
            ],
            model: "gpt-4o",
            temperature: 1,
            max_tokens: 4096,
            top_p: 1
        }
    });

    if (response.status !== "200") {
        throw response.body.error;
    }
    console.log(response.body.choices[0].message.content);
```

The line for token makes no sense in our extension as we can get this token from headers so refactor the token line to read headers and move the line to the top of the `/agent` endpoint

```JS
const token = process.env["GITHUB_TOKEN"];
```

and change to

```JS
const token = req.headers["x-github-token"];
```

## 10. Debug the code from the Web Client (Copilot in GitHub.com)

- Run the webserver: `node index.js` (ensure public)

- Head to your repo in GitHub.com and bring up Copilot using the icon in the menu

- In the Ask Copilot edit field, execute your extension with `@<extension name> <some ingredients>`

- Click 'Connect' for Authorization flow.

- Choose your user in picker (if you have multiple accounts)

- Authorize the extension (click continue if needed) - NOTE: you have hit your `/callback` endpoint.

- Close this callback tab in the browser and head back to the Copilot Chat in GitHub.com (ignore the Connect - this is done now)

- Click up in the Ask Copilot edit field for your previous `@<extension name> <some ingredients>` command and hit enter

- Note that the console.log should have the response from the model output.

## 11. Output the model repsonse to Copilot

For this we will use the [Copilot Extension - Preview SDK](https://github.com/copilot-extensions/preview-sdk.js) this simplifies a number of operations in javascript extensions, such as formatting the response, but also more advanced features like confirmation message and very importantly security scenarios such as checking a request came from GitHub and signed by GitHub based on public key.

- Add import to the Copilot-Extension preview-sdk to the imports section

```JS
import { createTextEvent, createDoneEvent, verifyAndParseRequest } from "@copilot-extensions/preview-sdk";
```

- replace `res.end();` with `createTextEvent` and `createDoneEvent` i.e.

```JS
    res.send(createTextEvent(response.body.choices[0].message.content) + createDoneEvent());
```

- Re-run the webserver: `node index.js` (ensure public)

- Head back to the Copilot Chat in GitHub.com (ignore the Connect - this is done now), click up in the Ask Copilot edit field for your previous `@<extension name> <some ingredients>` command and hit enter

- Check the response now comes to the Copilot in GitHub.com web client.

## 12. Wire up ingredients prompt into the model

- Change the system and user messages to your correct prompt

i.e.

```JS
                { role: "system", content: "" },
                { role: "user", content: "Can you explain the basics of machine learning?" }
```

now changes to something like:

```JS
                { role: "system", content: "As a chef you will collect a list of ingredients and recommend some recipes" },
                { role: "user", content: "My ingredients are " + ingredients }
```

- Head back to the Copilot Chat in GitHub.com (ignore the Connect - this is done now), click up in the Ask Copilot edit field for your previous `@<extension name> <some ingredients>` command and hit enter

- Check the recipe response now comes to the Copilot in GitHub.com web client.

## 13. [Optional] Adding Security to the endpoint to verify calls come from GitHub

- Edit the existing middleware to have a rawBody in utf8 which is not JSON modified so can be verified as a payload with GitHub's public key.
  
```JS
app.use(express.json();
```

changing it to 

```JS
app.use(express.json({
    verify: (req, res, buf) => {
      req.rawBody = buf.toString('utf8');
    }
  }))
```

- Call verifyAndParseRequest from the Preview SDK to verify the payload. Inserting the following code after the current `const token = req.headers["x-github-token"];` line.

```JS
    const signature = String(req.headers["github-public-key-signature"]);
    const keyId = String(req.headers["github-public-key-identifier"]);

    const { isValidRequest, payload, cache } = await verifyAndParseRequest(
        req.rawBody,
        signature,
        keyId,
        {
            token: token,
        },
    );
      
    if (!isValidRequest) {
        throw new Error("Request could not be verified");
    }
```
