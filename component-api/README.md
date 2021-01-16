# Use of Dapr as a component API Server

This demo users Dapr instance with API token authentication to show the use of Dapr as a API server for any of its 70+ components. To illustrate, this demo will show two use-cases:

* Simple note management using Redis state store
* Sending email using Sendgrid output binding
* Querying tweets using Twitter bi-directional binding

## Setup 

### State Component 

Create a `mongo-secret`

```shell
kubectl create secret generic redis-secret --from-literal=password=""
```

Deploy component and [restart gateway](#ingress-gateway)

```shell
kubectl apply -f config/state.yaml
```

### Email Component 

Create a `email-secret`

```shell
kubectl create secret generic email-secret --from-literal=apiKey=""
```

Deploy component and [restart gateway](#ingress-gateway)

```shell
kubectl apply -f config/email.yaml
```

### Twitter Component

Create a `twitter-secret`

```shell
kubectl create secret generic twitter-secret \
  --from-literal=consumerKey="" \
  --from-literal=consumerSecret="" \
  --from-literal=accessToken="" \
  --from-literal=accessSecret=""
```

Deploy component and [restart gateway](#ingress-gateway)

```shell
kubectl apply -f config/twitter.yaml
```

### Ingress Gateway

Ensure all the gateway instances are aware of new components

```shell
kubectl rollout restart deployment/nginx-ingress-nginx-controller
kubectl rollout status deployment/nginx-ingress-nginx-controller
```

## Usage

To use any of the components you will need the Dapr API token: 

```shell
export API_TOKEN=$(kubectl get secret dapr-api-token -o jsonpath="{.data.token}" | base64 --decode)
```

### State 

And POST it to the Dapr API to save your note:

```shell
curl -X POST \
     -d '[{ "key": "1", "value": "This is my first note" }]' \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     https://api.cloudylabs.dev/v1.0/state/note-store
```

Retrieve the saved note:

```shell
curl -X GET \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     https://api.cloudylabs.dev/v1.0/state/note-store/1
```

And now delete the note:

```shell
curl -X DELETE \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     https://api.cloudylabs.dev/v1.0/state/note-store/1
```

> For brevity of the example this demo shows only the save, get, delete commands but the Dapr API also includes transactional operations for save and bulk operations for get as well. 


### Email 

To send email, first edit the [sample email](./sample/email.json) file: 

```json
{
    "operation": "create",
    "metadata": {
        "emailTo": "daprdemo@chmarny.com",
        "subject": "Dapr Demo"
    },
    "data": "<h1>Greetings</h1><p>Hi</p>"
}
```

And POST it to the Dapr API:

```shell
curl -d @./sample/email.json \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     "https://api.cloudylabs.dev/v1.0/bindings/send-email"
```

### Twitter 

To query the last 100 tweets for particular query, first edit the [sample query](./sample/twitter.json) file:

```json
{
    "operation": "get",
    "metadata": {
        "query": "dapr AND serverless",
        "lang": "en",
        "result": "recent"        
    }
}
```

Metadata parameters:

* `query` - can be any valid Twitter query (supports `AND`, `OR` `BUT NOT`, `FROM`, `TO`, `#`, `@`...)
* `lang` - (optional) is the [ISO 639-1](https://meta.wikimedia.org/wiki/Template:List_of_language_names_ordered_by_code) language code
* `result` - (optional) is one of:
  * `mixed` - include both popular and real time results in the response
  * `recent` - return only the most recent results in the response
  * `popular` - return only the most popular results in the response
* `since_id` - (optional) the not inclusive tweet ID query should start from 

And POST it to the Dapr API:

```shell
curl -d @./sample/twitter.json \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     "https://api.cloudylabs.dev/v1.0/bindings/query-twitter"
```

And if you have the command-line JSON processor [jq](https://shapeshed.com/jq-json/),  you can format the API results. For example, this will display only the ID, Author, and Text of each tweet as a new JSON object:

```shell
curl -d @./sample/twitter.json \
     -H "Content-Type: application/json" \
     -H "dapr-api-token: ${API_TOKEN}" \
     "https://api.cloudylabs.dev/v1.0/bindings/query-twitter" \
     | jq ".[] | { id: .id_str, user: .user.screen_name, text: .text}"
```

The result

```shell
{
  "id": "1298546227211055109",
  "user": "markgossa",
  "text": "What a blast! @AzureFunctions Live of August was fully packed with news (new extension bundle, Dapr extension)"
}
{
  "id": "1298181483547357184",
  "user": "ysakashita3",
  "text": "I submitted a blog post to https://t.co/DXGTgtC4Xc. 'Serverless plugin': #KEDA for scaling down your containers"
}
```

## Disclaimer

This is my personal project and it does not represent my employer. While I do my best to ensure that everything works, I take no responsibility for issues caused by this code.

## License

This software is released under the [MIT](../LICENSE)
