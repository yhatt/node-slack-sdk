---
title: Interactive Messages
permalink: /interactive-messages
order: 3
---

# Slack Interactive Messages for Node

The `@slack/interactive-messages` helps your app respond to interactions from Slack's
[interactive messages](https://api.slack.com/messaging/interactivity), [actions](https://api.slack.com/actions), and [dialogs](https://api.slack.com/dialogs). This package will help you start with convenient and secure defaults.

## Installation

```shell
$ npm install @slack/interactive-messages
```

Before building an app, you'll need to [create a Slack app](https://api.slack.com/apps/new) and install it to your
development workspace. You'll also **need a public URL** where the app can begin receiving actions. Finally, you'll need
to find the **request signing secret** given to you by Slack under the "Basic Information" of your app configuration.

It may be helpful to read the tutorial on [developing Slack apps locally](https://slack.dev/node-slack-sdk/local_development).

---

### Initialize the message adapter

The package exports a `createMessageAdapter()` function, which returns an instance of the `SlackMessageAdapter` class.
The function requires one parameter, the **request signing secret**, which it uses to enforce that all events are coming
from Slack to keep your app secure.

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');

// Read the signing secret from the environment variables
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;

// Initialize
const slackInteractions = createMessageAdapter(slackSigningSecret);
```

---

### Start a server

The message adapter transforms incoming HTTP requests into verified and parsed actions, and dispatches actions to the
appropriate handler. That means, in order for it dispatch actions for your app, it needs an HTTP server. The adapter can
receive requests from an existing server, or as a convenience, it can create and start the server for you.

In the following example, the message adapter starts an HTTP server using the `.start()` method. Starting the server
requires a `port` for it to listen on. This method returns a `Promise` which resolves for an instance of an [HTTP
server](https://nodejs.org/api/http.html#http_class_http_server) once it's ready to emit events. By default, the
built-in server will be listening for events on the path `/slack/actions`, so make sure your Request URL ends with this
path.

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const slackInteractions = createMessageAdapter(slackSigningSecret);

// Read the port from the environment variables, fallback to 3000 default.
const port = process.env.PORT || 3000;

(async () => {
  // Start the built-in server
  const server = await slackInteractions.start(port);

  // Log a message when the server is ready
  console.log(`Listening for events on ${server.address().port}`);
})();
```

**Note**: To gracefully stop the server, there's also the `.stop()` method, which returns a `Promise` that resolves
when the server is no longer listening.

<details>
<summary markdown="span">
<strong><i>Using an existing HTTP server</i></strong>
</summary>

The message adapter can receive requests from an existing Node HTTP server. You still need to specify a port, but this
time its only given to the server. Starting a server in this manner means it is listening to requests on all paths; as
long as the Request URL is routed to this port, the adapter will receive the requests.

```javascript
const { createServer } = require('http');
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const slackInteractions = createMessageAdapter(slackSigningSecret);

// Read the port from the environment variables, fallback to 3000 default.
const port = process.env.PORT || 3000;

// Initialize a server using the message adapter's request listener
const server = createServer(slackInteractions.requestListener());

server.listen(port, () => {
  // Log a message when the server is ready
  console.log(`Listening for events on ${server.address().port}`);
});
```

</details>

<details>
<summary markdown="span">
<strong><i>Using an Express app</i></strong>
</summary>

The message adapter can receive requests from an [Express](http://expressjs.com/) application. Instead of plugging the
adapter's request listener into a server, it's plugged into the Express `app`. With Express, `app.use()` can be used to
set which path you'd like the adapter to receive requests from. **You should be careful about one detail: if your
Express app is using the `body-parser` middleware, then the adapter can only work if it comes _before_ the body parser
in the middleware stack.** If you accidentally allow the body to be parsed before the adapter receives it, the adapter
will emit an error, and respond to requests with a status code of `500`.

```javascript
const { createServer } = require('http');
const express = require('express');
const bodyParser = require('body-parser');
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const port = process.env.PORT || 3000;
const slackInteractions = createMessageAdapter(slackSigningSecret);

// Create an express application
const app = express();

// Plug the adapter in as a middleware
app.use('/my/path', slackInteractions.requestListener());

// Example: If you're using a body parser, always put it after the message adapter in the middleware stack
app.use(bodyParser());

// Initialize a server for the express app - you can skip this and the rest if you prefer to use app.listen()
const server = createServer(app);
server.listen(port, () => {
  // Log a message when the server is ready
  console.log(`Listening for events on ${server.address().port}`);
});
```
</details>

---

### Handle an action

Actions are interactions in Slack that generate an HTTP request to your app. These are:

-  **Block actions**: A user interacted with one of the [interactive
   components](https://api.slack.com/reference/messaging/interactive-components) in a message built with [block
   elements](https://api.slack.com/reference/messaging/block-elements).
-  **Message actions**: A user selected an [action in the overflow menu of a message](https://api.slack.com/actions).
-  **Dialog submission**: A user submitted a form in a [modal dialog](https://api.slack.com/dialogs)
-  **Attachment actions**: A user clicked a button or selected an item in a menu in a message built with [legacy message
   attachments](https://api.slack.com/interactive-messages).

You app will only handle actions that occur in messages or dialogs your app produced. [Block Kit
Builder](https://api.slack.com/tools/block-kit-builder) is a playground where you can prototype your interactive
components with block elements.

Apps register functions, called **handlers**, to be triggered when an action is received by the adapter using the
`.action(constraints, handler)` method. When registering a handler, you describe which action(s) you'd like the handler
to match using **constraints**. Constraints are [described in detail](#constraints) below. The adapter will call the
handler whose constraints match the action best.

These handlers receive up to two arguments:

1. `payload`: An object whose contents describe the interaction that occurred. Use the links above as a guide for the
   shape of this object (depending on which kind of action you expect to be handling).
2. `respond(...)`: A function used to follow up on the action after the 3 second limit. This is used to send an
   additional message (`in_channel` or `ephemeral`, `replace_original` or not) after some deferred work. This can be
   used up to 5 times within 30 minutes.

Handlers can return an object, or a `Promise` for a object which must resolve within the `syncResponseTimeout` (default:
2500ms). The contents of the object depend on the kind of action that's being handled.

- **Attachment actions**: The object describes a message to replace the message where the interaction occurred. **It's
  recommended to remove interactive elements when you only expect the action once, so that no other users might trigger
  a duplicate.** If no value is returned, then the message remains the same.
- **Dialog submission**: The object describes [validation errors](https://api.slack.com/dialogs#input_validation) to
  show the user and prevent the dialog from closing. If no value is returned, the submission is treated as successful.
- **Block actions** and **Message actions**: Avoid returning any value.

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const slackInteractions = createMessageAdapter(slackSigningSecret);
const port = process.env.PORT || 3000;

// Example of handling static select (a type of block action)
slackInteractions.action({ type: 'static_select' }, (payload, respond) => {
  // Logs the contents of the action to the console
  console.log('payload', payload);

  // Send an additional message to the whole channel
  doWork()
    .then(() => {
      respond({ text: 'Thanks for your submission.' });
    })
    .catch((error) => {
      respond({ text: 'Sorry, there\'s been an error. Try again later.' });
    });

  // If you'd like to replace the original message, use `chat.update`.
  // Not returning any value.
});

// Example of handling all message actions
slackInteractions.action({ type: 'message_action' }, (payload, respond) => {
  // Logs the contents of the action to the console
  console.log('payload', payload);

  // Send an additional message only to the user who made interacted, as an ephemeral message
  doWork()
    .then(() => {
      respond({ text: 'Thanks for your submission.', response_type: 'ephemeral' });
    })
    .catch((error) => {
      respond({ text: 'Sorry, there\'s been an error. Try again later.', response_type: 'ephemeral' });
    });

  // If you'd like to replace the original message, use `chat.update`.
  // Not returning any value.
});

// Example of handling all dialog submissions
slackInteractions.action({ type: 'dialog_submission' }, (payload, respond) => {
  // Validate the submission (errors is of the shape in https://api.slack.com/dialogs#input_validation)
  const errors = validate(payload.submission);

  // Only return a value if there were errors
  if (errors) {
    return errors;
  }

  // Send an additional message only to the use who made the submission, as an ephemeral message
  doWork()
    .then(() => {
      respond({ text: 'Thanks for your submission.', response_type: 'ephemeral' });
    })
    .catch((error) => {
      respond({ text: 'Sorry, there\'s been an error. Try again later.', response_type: 'ephemeral' });
    });
});

// Example of handling attachment actions. This is for button click, but menu selection would use `type: 'select'`.
slackInteractions.action({ type: 'button' }, (payload, respond) => {
  // Logs the contents of the action to the console
  console.log('payload', payload);

  // Replace the original message again after the deferred work completes.
  doWork()
    .then(() => {
      respond({ text: 'Processing complete.', replace_original: true });
    })
    .catch((error) => {
      respond({ text: 'Sorry, there\'s been an error. Try again later.',  replace_original: true });
    });

  // Return a replacement message
  return { text: 'Processing...' };
});

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```

---

### Handle an options request

Options requests are generated when a user interacts with a menu that uses a dynamic data source. These menus can be
inside a block element, an attachment, or a dialog. In order for an app to use a dynamic data source, you must save an
"Options Load URL" in the app configuration.

Apps register functions, called **handlers**, to be triggered when an options request is received by the adapter using
the `.options(constraints, handler)` method. When registering a handler, you describe which options request(s) you'd
like the handler to match using **constraints**. Constraints are [described in detail](#constraints) below. The adapter
will call the handler whose constraints match the action best.

These handlers receive a single `payload` argument. The `payload` describes the interaction with the menu that occurred.
The exact shape depends on whether the interaction occurred within a [block
element](https://api.slack.com/reference/messaging/block-elements#external-select),
[attachment](https://api.slack.com/docs/message-menus#options_load_url), or a
[dialog](https://api.slack.com/dialogs#dynamic_select_elements_external).

Handlers can return an object, or a `Promise` for a object which must resolve within the `syncResponseTimeout` (default:
2500ms). The contents of the object depend on where the options request was generated (you can find the expected shapes
in the preceding links).

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const slackInteractions = createMessageAdapter(slackSigningSecret);
const port = process.env.PORT || 3000;

// Example of handling options request within block elements
slackInteractions.options({ within: 'block_actions' }, (payload) => {
  // Return a list of options to be shown to the user
  return {
    options: [
      {
        text: {
          type: 'plain_text',
          text: 'A good choice',
        },
        value: 'good_choice',
      },
    ],
  };
});

// Example of handling options request within attachments
slackInteractions.options({ within: 'interactive_message' }, (payload) => {
  // Return a list of options to be shown to the user
  return {
    options: [
      {
        text: 'A decent choice',
        value: 'decent_choice',
      },
    ],
  };
});

// Example of handling options request within dialogs
slackInteractions.options({ within: 'dialog' }, (payload) => {
  // Return a list of options to be shown to the user
  return {
    options: [
      {
        label: 'A choice',
        value: 'choice',
      },
    ],
  };
});

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```

---

### Constraints

Constraints allow you to describe when a handler should be called. In simpler apps, you can use very simple constraints
to divide up the structure of your app. In more complex apps, you can use specific constraints to target very specific
conditions, and express a more nuanced structure of your app.

Constraints can be a simple string, a `RegExp`, or an object with a number of properties.

| Property name | Type | Description | Used with `.actions()` | Used with `.options()` |
|---------------|------|-------------|------------------------|------------------------|
| `callbackId` | `string` or `RegExp` | Match the `callback_id` for attachment or dialog | ✅ | ✅ |
| `blockId` | `string` or `RegExp` | Match the `block_id` for a block action | ✅ | ✅ |
| `actionId` | `string` or `RegExp` | Match the `action_id` for a block action | ✅ | ✅ |
| `type` | any block action element type or `message_actions` or `dialog_submission` or `button` or `select` | Match the kind of interaction | ✅ | 🚫 |
| `within` | `block_actions` or `interactive_message` or `dialog` | Match the source of options request | 🚫 | ✅ |
| `unfurl` | `boolean` | Whether or not the `button`, `select`, or `block_action` occurred in an App Unfurl |  ✅ | 🚫 |

All of the properties are optional, its just a matter of how specific you want to the handler's behavior to be. A
`string` or `RegExp` is a shorthand for only specifying the `callbackId` constraint. Here are some examples:

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const slackInteractions = createMessageAdapter(slackSigningSecret);
const port = process.env.PORT || 3000;

// Example of constraints for an attachment action with a callback_id
slackInteractions.action('new_order', (payload, respond) => { /* ... */ });

// Example of constraints for a block action with an action_id
slackInteractions.action({ actionId: 'new_order' }, (payload, respond) => { /* ... */ });

// Example of constraints for an attachment action with a callback_id pattern
slackInteractions.action(/order_.*/, (payload, respond) => { /* ... */ });

// Example of constraints for a block action with a callback_id pattern
slackInteractions.action({ actionId: /order_.*/ }, (payload, respond) => { /* ... */ });

// Example of constraints for an options request with a callback_id and within a dialog
slackInteractions.options({ within: 'dialog', callbackId: 'model_name' }, (payload) => { /* ... */ });

// Example of constraints for all actions.
slackInteractions.action({}, (payload, respond) => { /* ... */ });

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```

---

### Debugging

If you're having difficulty understanding why a certain request received a certain response, you can try debugging your
program. A common cause is a request signature verification failing, sometimes because the wrong secret was used. The
following example shows how you might figure this out using debugging.

Start your program with the `DEBUG` environment variable set to `@slack/interactive-messages:*`. This should only be
used for development/debugging purposes, and should not be turned on in production. This tells the adapter to write
messages about what its doing to the console. The easiest way to set this environment variable is to prepend it to the
`node` command when you start the program.

```shell
$ DEBUG=@slack/interactive-messages:* node app.js
```

`app.js`:

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const port = process.env.PORT || 3000;

// Oops! Wrong signing secret
const slackInteractions = createMessageAdapter('not a real signing secret');

slackInteractions.action({ action_id: 'welcome_agree_button' }, (payload) => {
  /* Not shown: Record user agreement to database... */
  return {
    text: 'Thanks!',
  };
});

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```

When the adapter receives a request, it will now output something like the following to the console:

```
@slack/interactive-messages:adapter instantiated
@slack/interactive-messages:adapter server created - path: /slack/actions
@slack/interactive-messages:adapter server started - port: 3000
@slack/interactive-messages:http-handler request received - method: POST, path: /slack/actions
@slack/interactive-messages:http-handler request signature is not valid
@slack/interactive-messages:http-handler handling error - message: Slack request signing verification failed, code: SLACKHTTPHANDLER_REQUEST_SIGNATURE_VERIFICATION_FAILURE
@slack/interactive-messages:http-handler sending response - error: Slack request signing verification failed, responseOptions: {}
```

This output tells the whole story of why the adapter responded to the request the way it did. Towards the end you can
see that the signature verification failed.

If you believe the adapter is behaving incorrectly, before filing a bug please gather the output from debugging and
include it in your bug report.

---

### Late response fallback

Handlers for actions and options requests can return `Promise`s. If those `Promise`s don't resolve within a certain time
(2.5 seconds by default - see [`syncResponseTimeout`](#synchronous-response-timeout)), then the adapter will attempt
to send the resolved value to the `response_url`, this is called **late response fallback**. This feature works well for
attachment actions since there's no difference between what the `response_url` expects and what the
synchronous response contains. However with dialog submission and options requests, the `response_url` expects a message
while the return values should contain validation errors or options (respectively). In these cases, late response
fallback can cause problems.

You can choose to turn late response fallback off for the entire adapter by setting the `lateResponseFallbackEnabled`
option to `false`.

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const port = process.env.PORT || 3000;

// Turn late response fallback off
const slackInteractions = createMessageAdapter(slackSigningSecret, {
  lateResponseFallbackEnabled: false,
});

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```

When choosing to turn late response fallback off, its important to stop relying on returning `Promise`s from handler
functions, and instead use the `respond()` argument in your handlers for all deferred (asynchronous) responses.

---

### Synchronous response timeout

The amount of time that the adapter is willing to wait for a `Promise` returned from a handler to resolve, before
attempting [late response fallback](#late-response-fallback) is known as the **synchronous response timeout**. This
value is set to 2.5 seconds be default. That value was chosen because Slack will wait a maximum of 3 seconds for
interactions to receive a response, and there should be some additional padding for Node's processing time and
connection time. Leaving a half second for that overhead is relatively conservative.

You can choose to change the synchronous response timeout for the entire adapter by setting the `syncResponseTimeout`
to the number of milliseconds desired.

```javascript
const { createMessageAdapter } = require('@slack/interactive-messages');
const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;
const port = process.env.PORT || 3000;

// Turn late response fallback off
const slackInteractions = createMessageAdapter(slackSigningSecret, {
  lateResponseFallbackEnabled: false,
});

(async () => {
  const server = await slackInteractions.start(port);
  console.log(`Listening for events on ${server.address().port}`);
})();
```
