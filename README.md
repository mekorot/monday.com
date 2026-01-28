# How to Use the Monday API to Update Monday Boards from SAP FI Module

A concise step-by-step guide to integrate SAP FI with Monday.com using the Monday GraphQL API.

## Prerequisites
- Monday.com account with API permissions  
- Monday API token (use a secure secret store; do not hardcode) — e.g. `<MONDAY_API_TOKEN>`  
- Basic REST/GraphQL knowledge  
- Access to your SAP system

## 1. Set Up the Environment
Install HTTP/GraphQL client libraries in your environment.

Python:
```bash
pip install requests
```

Node.js:
```bash
npm install axios
```

## 2. Example: Query a Board (Node.js)
Replace the token placeholder with a secure value from your environment.
```js
const axios = require('axios');

const query = `
  query {
    boards(ids: 123) {
      name
    }
  }
`;

axios.post('https://api.monday.com/v2', { query }, {
  headers: {
    'Authorization': '<MONDAY_API_TOKEN>',
    'Content-Type': 'application/json'
  }
})
.then(res => console.log(res.data))
.catch(err => console.error(err));
```

## 3. Update Monday Board from SAP FI Module — Workflow

### 3.1 Fetch Monday Board ID by FI Project ID
Maintain a mapping board (example board ID: `8449913235`) that maps SAP FI Project IDs to Monday board IDs.

GraphQL request:
```graphql
query {
  boards(ids: 8449913235) {
    items_page(query_params: { rules: [
      { column_id: "text_mkphw6sd", compare_value: ["FI Project ID"] }
    ] }) {
      cursor
      items {
        id
        name
        column_values {
          column { id title }
          value
        }
      }
    }
  }
}
```

Sample relevant response excerpt:
```json
{
  "column_values": [
    { "column": { "id": "text_mkphe2nw", "title": "monday board ID" }, "value": "\"8813971050\"" },
    { "column": { "id": "text_mkphw6sd", "title": "Sap Project ID" }, "value": "\"123\"" }
  ]
}
```

Use the returned `monday board ID` to target subsequent updates.

### 3.2 Find WBS Item on Target Board
Query the target board for an item matching a WBS number:

```graphql
query {
  boards(ids: <fetchedBoardId>) {
    items_page(query_params: { rules: [
      { column_id: "wbs__1", compare_value: ["2000000036.2"] }
    ] }) {
      items {
        id
        name
        column_values { column { id title } value }
      }
    }
  }
}
```

- If an item is returned, update it.
- If no items returned, create a new item.

### 3.3 Update Existing WBS Item
Mutation to change multiple columns (use numeric values without quotes for numbers):

```graphql
mutation {
  change_multiple_column_values(
    board_id: 7388692471,
    item_id: 7388693067,
    column_values: "{\"name\":\"wbs name\",\"numbers9__1\":111,\"numeric8__1\":50,\"numeric5__1\":5}"
  ) {
    id
  }
}
```

Column mapping (example):
- name — WBS name (text)  
- numbers9__1 — approved budget (number)  
- numeric8__1 — actual payment (number)  
- numeric5__1 — left orders (number)

### 3.4 Create New WBS Item
Create one or several items in a single mutation:

```graphql
mutation {
  create_item1: create_item(board_id: 7388692471, item_name: "שם סעיף תקציבי 1", column_values: "{\"wbs__1\":\"200000011\",\"numbers9__1\":111,\"numeric8__1\":50,\"numeric5__1\":5}") {
    id
  }
  create_item2: create_item(board_id: 7388692471, item_name: "שם סעיף תקציבי 2", column_values: "{\"wbs__1\":\"200000022\",\"numbers9__1\":222,\"numeric8__1\":44,\"numeric5__1\":2}") {
    id
  }
}
```

## Conclusion
- Keep API tokens secure.  
- Fetch mapping (SAP FI Project ID → Monday board ID) first, then find/create/update WBS items by WBS number.  
- Refer to official docs for full API details:
  - Monday API: <https://api.developer.monday.com/>
  - SAP Integration: <https://help.sap.com/>

// ...existing code...
## 4. Power Automate (Power Platform) — Example Flow

A minimal Power Automate flow pattern to sync SAP FI → Monday.com.

Flow outline:
1. Trigger: "When an HTTP request is received" (or SAP connector trigger).
2. Action: Compose GraphQL query (or build in Body).
3. Action: HTTP (POST) to https://api.monday.com/v2 to run queries/mutations.
4. Action: Parse JSON on the response.
5. Condition: If items found → Update (mutation). Else → Create (mutation).

Example HTTP action settings (use secure storage for token, e.g., Power Automate connection or Azure Key Vault):

- Method: POST  
- URI: https://api.monday.com/v2  
- Headers:
  - Authorization: Bearer <MONDAY_API_TOKEN>
  - Content-Type: application/json
- Body (fetch board ID by FI Project ID):
```json
{ "query": "query { boards(ids: 8449913235) { items_page(query_params: {rules: [{column_id: \"text_mkphw6sd\", compare_value: [\"123\"]}]}) { items { id name column_values { column { id title } value } } } } }" }
```

Parse JSON schema: use a sample Monday response (items array) to generate schema.

If condition finds an item (items length > 0):
- HTTP action to update item (change_multiple_column_values):
```json
{
  "query": "mutation { change_multiple_column_values(board_id: 7388692471, item_id: 7388693067, column_values: \"{\\\"name\\\":\\\"wbs name\\\",\\\"numbers9__1\\\":111,\\\"numeric8__1\\\":50,\\\"numeric5__1\\\":5}\") { id } }"
}
```

Else (no item found):
- HTTP action to create item:
```json
{
  "query": "mutation { create_item(board_id: 7388692471, item_name: \"שם סעיף תקציבי\", column_values: \"{\\\"wbs__1\\\":\\\"200000011\\\",\\\"numbers9__1\\\":111,\\\"numeric8__1\\\":50,\\\"numeric5__1\\\":5}\") { id } }"
}
```

Notes:
- Store the Monday API token securely (Power Automate connection or Azure Key Vault).  
- Use dynamic content from the trigger (SAP payload) to fill FI Project ID, WBS number and numeric values.  
- Add error handling and retry policies for robustness.
// ...existing code...
