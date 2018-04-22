# webex-api-emulator: an Emulator for (some of) the Webex REST APIs

The primary goal of the emulator is to help facilitate the creation of Webex Teams (formerly Cisco Spark) chat bots.
Therefore, the emulator mimics Webex API REST API behavior for /rooms, /messages, /memberships and /webhooks resources.

The following features are NOT implemented: Message Attachments, Room moderation, People LastActivity & Status and Pagination, as well as Teams, Automatic Invitations (if non Webex Teams users are added to rooms) and Administration APIs.   Membership APIs that accept both emails and personIds currently only work with personIds in the emulator.

The emulator can be used for several purposes:
- testing: on a developer laptop machine or on a CI environment to run a battery of test with no connection to the Webex APIs, and without 409 (Rate Limitations)
- QA: run your code against a stable version of Webex Teams (as the Cloud Service is incrementally upgraded, some bugs can be hard to replay. The emulator complies with a version of the API at a specific date, and helps reproduce an issue, or test for an upcoming feature (not released yet or toggled on)
- Bot Regression testing:  Create a set of regression tests to ensure that for given user inputs you will get expected bot responses.   See [Bot Testing Mode](#Bot-Testing-Mode) below.
- Training: backup plan in case of low or no connectivity location
- QA: simulate specific behaviors or errors from Webex (429, 500, 503)

## Quickstart

- Install and run the emulator on your local machine (starts on port 3210 by default)

```shell
git clone https://github.com/webex/webex-api-emulator.git
cd webex-api-emulator
npm install
DEBUG="emulator*" node server.js
```

- The emulator loads with a [static list](./tokens.json) of Webex Teams User and Bots accounts.

    [Optional] update file [tokens.json](./tokens.json) with some fake access tokens you want the emulator to accept.
    DO NOT USE FILL IN REAL WEBEX TEAMS ACCESS TOKENS.
    Generally, you'll want to put in your name, email and organization identifiers as well as your bot's information.

- Invoke Webex API Resources on default port http://localhost:3210, with a user of bot token from [tokens.json](./tokens.json)

   - GET  /people/me           shows Webex Teams account details
   - POST /rooms               create a new room
   - POST /rooms               create another room
   - GET  /rooms               shows your rooms (2)
   - DELETE /rooms:id          deletes a room
   - POST /memberships         add a bot to the room
   - POST /memberships         409 (conflict)
   - GET  /memberships         show all your memberships
   - GET  /memberships?room=   fetch memberships for you and your bot in current room
   - DELETE /memberships
   - POST /messages            create a new message.  If the payload includes a personId, will create a new one on one space if one does not already exist.  (personEmail not yet supported)
   - POST /webhooks            register a new webhook pointing to a target URL on your local machine, or on the internet
   - DELETE /webhooks/:id      deletes a webhook

## Bot Testing Mode
  The emulator can be run in a special mode for bot testing by running with the environment variable BOT_UNDER_TEST set to the email of a bot user specified in tokens.json.   When this environment variable is set additional middleware is loaded that will attempt to correlate any incoming requests from a text suite with an X-Bot-Responses header with any subsequent requests that come from the bot.

  Specifically, the middleware performs that following functions:
  - Inspect all incoming requests for an X-Bot-Responses header.  When found it will intercept the response that normally would be sent to the caller of the API
  - Inspect all incoming requests from the BOT_UNDER_TEST.  When found the system checks to see if the bot request is correlated to any prior requests with the X-Bot-Response header.   Generally correlation is done using the roomId.   
  - When correlated Bot responses are found, the emulator will build a new response body that will contain the response body to the original request in an object called testFrameworkResponse, as well as an array of botResponse objects that occurred in response to the test input.   Each botResponse object will contain the request sent by the Bot.  The number of botResponse objects collected is the value specified in the X-Bot-Responses header.   

  As an example imagine a simple bot that will send a message in response to being added to a space, and then leave the space.  To validate this behavior with a test we would write a test that adds the bot to a space using the /memberships endpoint, which also set an X-Bot-Response header to 2, ie:
```
POST http://localhost:3210/memberships
{
    "headers" {
        "x-bot-responses": "2",
        ...
    },
    "body" {
        "roomId": "Y2lzY29zcGFyazovL2VtL1JPT00vMzUzN2Q2ZTAtMmY5My00N2M0LWIwODMtZDYxNTg3MWZiMzFj"
        "personId": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS9hODYwYmFkZC0wNGZkLTQwYWEtYWFjNS05NmYyYWRhZDE3NTA"
        "isModerator": "false"
    },
    ...
}
```
  In this example the personId should match the ID of a bot user specified in the emulator's [tokens.json](./tokens.json) file.

  A sample consolidated response to such a request will have the status for the original request response and a body which might look as follows:
  ```
{
    "testFrameworkResponse": {
        "id": "Y2lzY29zcGFyazovL2VtL01FTUJFUlNISVAvMDMzMjllODctMzA4Ni00OThiLTg4ZGMtMzM5ZTVhODdlZGEy",
        "roomId": "Y2lzY29zcGFyazovL2VtL1JPT00vMzUzN2Q2ZTAtMmY5My00N2M0LWIwODMtZDYxNTg3MWZiMzFj",
        "personId": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS9hODYwYmFkZC0wNGZkLTQwYWEtYWFjNS05NmYyYWRhZDE3NTA",
        "personEmail": "bot@sparkbot.io",
        "personDisplayName": "Bot",
        "personOrgId": "Y2lzY29zcGFyazovL3VzL09SR0FOSVpBVElPTi8xZWI2NWZkZi05NjQzLTQxN2YtOTk3NC1hZDcyY2FlMGUxMGY",
        "isModerator": false,
        "isMonitor": false,
        "created": "2018-03-15T03:12:59.220Z"
    },
    "botResponses": [
        {
            "method": "POST",
            "route": "/messages",
            "body": {
                "markdown": "Hi! Sorry, I only work in one on one rooms at the moment.  Goodbye.",
                "roomId": "Y2lzY29zcGFyazovL2VtL1JPT00vMzUzN2Q2ZTAtMmY5My00N2M0LWIwODMtZDYxNTg3MWZiMzFj"
            }
        },
        {
            "method": "DELETE",
            "route": "/memberships/Y2lzY29zcGFyazovL2VtL01FTUJFUlNISVAvMDMzMjllODctMzA4Ni00OThiLTg4ZGMtMzM5ZTVhODdlZGEy",
            "body": {}
        }
    ]
}
```
The developer can then use tools like Postman to build a collection of test requests and write tests to validate the expected response.

An added benefit of the Bot Test regression framework is that it can be run on a laptop that is completely offline.   Its great for working on your bot while on an airplane!   In general your bot code itself needs to change only one thing: namely it needs to direct its Webex API calls to the emulator.   The way this is done varies depending on the framework you are using.

For more details on how this works [this sample project](https://github.com/jpjpjp/bot-test-framework-example) includes a link to this project, a link to a simple bot, a suite of postman tests and a set of detailed instructions on how to use the emulator in bot test mode.

## Testing

- The emulator comes with a Postman collection companion to quickly run requests against bot the emulator, and the real Webex API endpoint, and compare those results.  To install the postman collection:

    - Import the collection into Postman by [clicking this link](https://www.getpostman.com/collections/c76412ea237207555b57), or with the "import collection" feature

    - setup an environment with 4 variables: endpoint, spark_token, bot_token, bot_email
       - endpoint: http://127.0.0.1:3210 (in place of https://api.ciscospark.com/v1)
       - spark_token: the access token of a Webex Teams Human user
       - bot_token: the access token of a Webex Teams Bot account
       - bot_email: the email of the bot (for the user to invite it)

    Note that values for `spark_token`, `bot_token` and `bot_email` must be part of the tokens.json file

Currently, the tests primarily exercise the markdown features of the /messages endpoint.   Pull Requests for additional tests will gladly be reviewed!

## Questions and feedback

We'd love to hear how developers are using the Webex API Emulator.   If you have questions, or feedback, feel free to submit issues or join the [Ask Webex API Emulator](https://eurl.io/#HyfEkmq2f) space.
