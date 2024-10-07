# copilot-extension-workshop

1. Create a repo

| Field           | Value                          |
| --------------- | ------------------------------ |
| Repository name | <e.g. ingredients-demo-js> |
| Readme          | Add                            |
| .gitignore      | Node                           |

2. Add /.devcontainer/devcontainer.json

- Content of [devcontainer.json](https://gist.github.com/colinbeales/9c9a96cd00571e6fcaaac705258024d3#file-devcontainer-json)
- Commit

3. Start Codespace

4. Add ./packages.json

- Content of [package.json](https://gist.github.com/colinbeales/1e1364d93c10fe318e4b4ae04940b89a#file-package-json)

- Call `npm install`

5. Implement simple express API

Should be Async

Should use JSON Middleware i.e. `app.use(express.json());`

Needs Endpoints:

- GET / - To prove this API works

- GET /callback - Output from a callback on registration

- POST /agent - Where our Copilot Extension will be called from

Example content for [index.js](https://gist.github.com/colinbeales/28141460eb151d867a3d4718f1a067ff#file-index-js)

6. Run API in the Codespace

- Run `node index.js`

- Make Codespace public

- Show ports - Copy the public URL

- Show `/` and `/callback` endpoints functioning.

7. Register Copilot Extension as a GitHub App

- Head to [Settings->Developer Settings->GitHub Apps](https://github.com/settings/apps) and select 'New GitHub App' button

- Fill in:

| Field                             | Value                                       |
| --------------------------------- | ------------------------------------------- |
| GitHub App name                   | <e.g. ingredients-demo-js>              |
| Description                       | <description of the ingrediebnts demo>      |
| Homepage URL                      | <pasted from public codespace URL>          |
| Callback URL                      | <pasted from public codespace URL>/callback |
| WebHook                           | Change to <inactive>                        |
| Account Permissions->Copilot Chat | Read-only                                   |

Click 'Create GitHub App' button.

- Fill in Copilot tab:

| Field                 | Value                                          |
| --------------------- | ---------------------------------------------- |
| App Type              | Agent                                          |
| URL                   | <pasted from public codespace URL>/agent       |
| Inference Description | <e.g. list ingredients for recipe suggestions> |

Click 'Save' to save these changes.

- Head to 'Install App' tab and click Install the App.

8. Add code to get ingredients from body (and debug using CLI):

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

9. Get code from GitHub Models and integrate

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

10. Debug the code from the Web Client (Copilot in GitHub.com)

- Run the webserver: `node index.js` (ensure public)

- Head to your repo in GitHub.com and bring up Copilot using the icon in the menu

- In the Ask Copilot edit field, execute your extension with `@<extension name> <some ingredients>`

- Click 'Connect' for Authorization flow.

- Choose your user in picker (if you have multiple accounts)

- Authorize the extension (click continue if needed) - NOTE: you have hit your `/callback` endpoint.

- Close this callback tab in the browser and head back to the Copilot Chat in GitHub.com (ignore the Connect - this is done now)

- Click up in the Ask Copilot edit field for your previous `@<extension name> <some ingredients>` command and hit enter

- Note that the console.log should have the response from the model output.

11. Output the model repsonse to Copilot

For this we will use the [Copilot Extension - Preview SDK](https://github.com/copilot-extensions/preview-sdk.js) this simplifies a number of operations in javascript extensions, such as formatting the response, but also more advanced features like confirmation message and very importantly security scenarios such as checking a request came from GitHub and signed by GitHub based on public key.

- Add import to the Copilot-Extension preview-sdk to the imports section

```JS
import { createTextEvent, createDoneEvent } from "@copilot-extensions/preview-sdk";
```

- replace `res.end();` with `createTextEvent` and `createDoneEvent` i.e.

```JS
    res.send(createTextEvent(response.body.choices[0].message.content) + createDoneEvent());
```

- Re-run the webserver: `node index.js` (ensure public)

- Head back to the Copilot Chat in GitHub.com (ignore the Connect - this is done now), click up in the Ask Copilot edit field for your previous `@<extension name> <some ingredients>` command and hit enter

- Check the response now comes to the Copilot in GitHub.com web client.

12. Wire up ingredients prompt into the model

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
