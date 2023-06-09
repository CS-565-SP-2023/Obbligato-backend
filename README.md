# Obbligato-backend

Backend for our crowdsourced optical music recognition system, Obbligato. Also includes task data, user responses, and some analysis. For more details on Obbligato itself, [check out the sister frontend repository](https://github.com/CS-565-SP-2023/Obbligato). Obbligato's backend is written in TypeScript, using Node.js, Express, and Mongoose to create a RESTful API to store and retrieve information in a MongoDB database.

Created by Runyao Fan, Mohit Gupta, Naman Jain, Chris Kull, Weili Li and Yutao Zhou from the University of Illinois Urbana-Champaign.

## Contents

- [Datasets and Analysis](#datasets-and-analysis)
- [API Documentation](#api-documentation)
- [API Usage for Frontend](#api-usage-for-frontend)
- [Development Environment Setup](#development-environment-setup)
- [Running the Server](#running-the-server)

## Datasets and Analysis

Two datasets are included in this repository, along with analysis of the results.

In `test_data`, you will find the 3 test sheets we used for the study each with 16 measures (for a total of 48 "task" measures). These were created by the project group, and they make a few assumptions. Everything is in treble clef, C major, 4/4 time signature, no accidentals, and no ledger lines. The only valid musical symbols are quarter note, half note, whole note, quarter rest, half rest, and whole rest. The original photo of each music sheet, the individually segmented out measures, and the gold answers are provided.

In `backups`, you will find the state of the MongoDB database as of 1:30pm 2023-05-07. In addition to the sheets/measures information, this also includes registered users and responses to measure tasks. Data was collected from 2023-05-04 to 2023-05-07, and is stored in JSON format. To keep file sizes reasonable, the raw binary image data is omitted. For privacy reasons, provided emails in the users dataset have been redacted. To allow for analysis on a per user level, though, emails were anonymized in such a way that map one-to-one to original emails. This means that if a user appeared twice in the original dataset (perhaps from two different sessions on the system), that is maintained.

In `analysis`, you will find a Jupyter notebook used to calculate a variety of statistics and results from the study. To run, make sure you set up the Python 3 environment (see further below).

## API Documentation

### Models

User:
| Field | Type | Required | Unique | Default | Notes |
|-------|------|----------|--------|---------|-------|
| email | string | ✘ | ✘ | - | for raffle giveaway purposes |
| experience | string | ✓ | ✘ | - | user's experience reading sheet music |

Sheet:
| Field | Type | Required | Unique | Default | Notes |
|-------|------|----------|--------|---------|-------|
| name | string | ✓ | ✓ | - | title of sheet music |
| image | Buffer | ✓ | ✓ | - | base64 encoded image of sheet, only support single image |

Measure:
| Field | Type | Required | Unique | Default | Notes |
|-------|------|----------|--------|---------|-------|
| sheetId | ObjectId | ✓ | ✘ | - | document ID for corresponding sheet |
| measureNum | number | ✓ | ✘ | - | number of measure within sheet for ordering |
| image | Buffer | ✓ | ✓ | - | base64 encoded image of measure, only support single image |
| responses | [MeasureResponse] | ✘ | ✘ | [] | stores crowdworker digitizations of measure |
| goldSymbols | [Symbols] | ✘ | ✘ | [] | correct symbols for this measure, for accuracy calculations | 

MeasureResponse:
| Field | Type | Required | Unique | Default | Notes |
|-------|------|----------|--------|---------|-------|
| userId | ObjectId | ✓ | ✘ | - | document ID for corresponding user who submitted response |
| symbols | [Symbols] | ✘ | ✘ | [] | symbols in response |

Symbols:
| Field | Type | Required | Unique | Default | Notes |
|-------|------|----------|--------|---------|-------|
| name | string | ✓ | ✘ | - | name of symbol, one of {quarter_note, half_note, whole_note, quarter_rest, half_rest, whole_rest} |
| pitch | number | ✘ | ✘ | - | location of note object, where 1 is the bottom line, 2 is the bottom space, 3 is the second to bottom line, etc. |

### Endpoints

These are the following endpoints available:

| Endpoints  | Actions | Intended Outcome                                        |
|------------|---------|---------------------------------------------------------|
| users      | GET     | Respond with a list of users                            |
|            | POST    | Create a new user. Respond with details of new user     |
| users/:id  | GET     | Respond with details of specified user or 404 error     |
|            | PUT     | Replace entire user with supplied user or 404 error     |
|            | DELETE  | Delete specified user or 404 error                      |
| sheets     | GET     | Respond with a list of sheets                           |
|            | POST    | Create a new sheet. Respond with details of new sheet   |
| sheets/:id | GET     | Respond with details of specified sheet or 404 error    |
|            | PUT     | Replace entire sheet with supplied sheet or 404 error   |
|            | DELETE  | Delete specified sheet or 404 error                     |
| measures   | GET     | Respond with a list of measures                         |
|            | POST    | Create a new measures. Respond with details of new measures |
| measures/:id | GET   | Respond with details of specified measures or 404 error |
|            | PUT     | Replace entire measures with supplied measures or 404 error |
|            | DELETE  | Delete specified measures or 404 error                  |
| lilypond/:id | GET   | WIP                                                     |
| measuretask | GET    | Get the next measure to be digitized                    |
| measureresponse | POST | Add a digitization for the given measure              |

### Query Parameters

Additionally, the API has the following JSON encoded query string parameters for GET requests on all endpoints.

| Parameter | Description                                                                                  |
|----------|----------------------------------------------------------------------------------------------|
| where    | filter results based on JSON query                                                           |
| sort     | specify the order in which to sort each specified field  (1- ascending; -1 - descending)     |
| select   | specify the set of fields to include or exclude in each document  (1 - include; 0 - exclude) |
| skip     | specify the number of results to skip in the result set; useful for pagination               |
| limit    | specify the number of results to return (default should be 100 for tasks and unlimited for users)                    |
| count    | if set to true, return the count of documents that match the query (instead of the documents themselves)                    |

## API Usage for Frontend
Below gives examples of (some of) the endpoints that will be used in the frontend.

### Adding a new user
`POST` request to `<endpoint>/api/users` with the following JSON body:
```json
{
    // note that email is an optional field, in which case you could just send an empty body
    // emails are required to be unique
    "email": "thisisatestemail1@test.com",
    "experience": "lots"
}
```

This returns something like the following:
```json
{
    "message": "User successfully created",
    "result": {
        "email": "thisisatestemail1@test.com",
        "experience": "lots",
        "_id": "644b77f8db4b011282fccd45",
        "__v": 0
    }
}
```

### Retrieving a task
`GET` request to `<endpoint>/api/measuretask`, no JSON body needed.

This returns something like the following:
```json
{
    "message": "Measure task GET successful!",
    "data": {
        "_id": "644b75053f4ea5e40af28956",
        "image": {
            "type": "Buffer",
            // base64 encoded image of the measure
            "data": [105, 86, ...]
        }
    }
}
```

### Submitting a task response
`POST` request to `<endpoint>/api/measureresponse` with the following JSON body:
```json
{
    // Object ID of the measure, from the previous endpoint
    "measureId": "644b75053f4ea5e40af28956",
    "measureResponse": {
        // Object ID of the current user
        "userId": "6441711a168095d6fe7d479f",
        // Array of symbols according to our Symbol objects.
        "symbols": [
            {
                "name": "quarter_note",
                "pitch": 9
            },
            {
                "name": "half_note",
                "pitch": 3
            },
            {
                "name": "quarter_rest"
            }
        ]
    }
}
```

This returns something like the following:
```json
{
    "message": "MeasureResponse POST successful!",
    "data": {
        "_id": "644b75053f4ea5e40af28956",
        "sheetId": "644b75043f4ea5e40af28954",
        "measureNum": 1,
        "responses": [
            {
                "userId": "6441711a168095d6fe7d479f",
                "symbols": [
                    {
                        "name": "quarter_note",
                        "pitch": 9,
                        "_id": "644b790fdb4b011282fccd48"
                    },
                    {
                        "name": "half_note",
                        "pitch": 3,
                        "_id": "644b790fdb4b011282fccd49"
                    },
                    {
                        "name": "quarter_rest",
                        "_id": "644b790fdb4b011282fccd4a"
                    }
                ],
                "_id": "644b790fdb4b011282fccd47"
            }
        ]
    }
}
```

## Development Environment Setup

Install nvm, node.js, npm (MacOS/Linux, [original reference](https://learn.microsoft.com/en-us/windows/dev-environment/javascript/nodejs-on-wsl)):
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash

# The below command may be needed so that nvm can be found by your shell
export NVM_DIR="$HOME/.nvm"
\. "$NVM_DIR/nvm.sh"
\. "$NVM_DIR/bash_completion"

nvm install --lts
```

Note that Windows will have to install a different way, [see here](https://github.com/coreybutler/nvm-windows).

Verify that the versions match these:
```bash
> node --version
v18.15.0
> npm --version
9.5.0
> nvm --version
0.39.3
```

<!-- [Following these instructions.](https://blog.logrocket.com/how-to-set-up-node-typescript-express/) -->

Install all node packages:
```bash
npm install
```

Create a `.env` file at the root project directory with the following contents (note that it is in the `.gitignore` and will be used to store secrets for development):
```properties
PORT=8000
# ask group for details
MONGO_CONNECTION=<authlink>
# Used for Python scripts, can configure to live or local server
ENDPOINT=<endpoint>
```

If using VSCode for development, installing [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) would be helpful.

Testing can be done either through the browser or using the tool [Postman](https://www.postman.com/).

To run the Python scripts, install [Python3](https://www.python.org/downloads/).

Optionally set up a virtual environment:
```bash
python3 -m venv backend-venv
```

Then install required dependencies:
```bash
python3 -m pip install -r python3_requirements.txt
```

## Running the Server

To run the server for development with hot-reload:
```bash
npm run dev
```

To run the server in "production":
```bash
npx run build && npm start
```

The repository is configured in such a way that any updates to the `main` branch will automatically be linted and then deployed to Heroku via CI.