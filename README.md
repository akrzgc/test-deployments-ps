<!-- Copyright (c) 2022 Graphcore Ltd. All rights reserved. -->
![Deployment status](https://github.com/graphcore/api-deployment/actions/workflows/deploy.yml/badge.svg)

# One click deployment
FastAPI ASGI based model serving
with GitHub workflow for deployment in a remote AI cluster.


As an example, this server is running a text summarization inference API under the path `/summarization`,
and a short question-answering API under the path `/qa`
The summarization model (facebook/bart-large-cnn) requires 1 available IPUs.
The question-answering model (distilbert-base-cased-distilled-squad) requires 2 IPUs.
API documentation generated by Swagger under the path operation `/docs`.

Disclaimers:
-   This repository shows an example of how you might deploy a server with GitHub. We encourage you to build on this example to create a service that meets the security and resiliancy requirements of your application.
-   Using GitHub actions with secrets can bring security risks. It is important to review GitHub actions parameters and who can trigger them. Be careful of any modification brought to .github/ssh/action.yml, or any new action you introduce that uses GitHub secrets. For instance, secrets should never be logged or saved in an artifact file. Recommended read for good practises: https://engineering.salesforce.com/github-actions-security-best-practices-b8f9df5c75f5/
-   The [Graphcore PyTorch](https://hub.docker.com/r/graphcore/pytorch) base Docker image used in this example is subject to [Graphcore Container License Agreement](https://docs.graphcore.ai/projects/container-license/en/latest/).
## Configuration:
The deployment workflow is relying on an example environment named `GCore-deployment-demo`.
Make sure you configure an environment with the same name (or modify the workflow files according to your environment name).
Configure the Github encrypted secrets SSH_KEY, USER and HOST_IP at the environment level or at the repository level.

The server can be configured via the .env file at the root of this repository.

## Add a new model:
To serve a new model, the main steps are the following:

1 - Add a Python file `new_model.py` in `src/models/` containing your model. It instantiates a callable named `pipe` that takes the necessary input data (we can directly import Graphcore-Optimum `Pipeline` for instance).
In this example we consider `inputs` to be a Dict (you are free to change it).
You should also define a function `compile`: The input is you object `pipe` and the execution of this function should trigger the IPU compilation.

ex: my_model.py

```python
class Pipeline:
    def __init__(self, args):
        # Various parameters initialisation

    def __call__(self, inputs: Dict) -> Dict:
        # pre-processing,
        # model call,
        # etc ..
        prediction_dict = ...
        return prediction_dict

def compile(pipe: Pipeline):
    # compilation logic goes here, for instance:
    # pipe(dummy_inputs)
    # ...
    return
...
pipe = Pipeline(args)
```
By implementing this interface, your new model will now be available as `new_model` (your file name) as a new IPUWorker.

2 - Create the API for this new model. In `src/server.py`:

```python
@app.post("/new_model", response_model=NMResponse, include_in_schema = "new_model" in models)
def run_nm(model_input: NM):
    data_dict = model_input.dict()
    w.workers["new_model"].feed(data_dict)
    result = w.workers["new_model"].get_result()
    return {
        "results": result["prediction"]
    }
```
In this simple example, our path operation is `/new_model`. We create the function `run_nm()` and use FastAPI decorator
`@app.post()` to make it receive POST requests. Using `include_in_schema` boolean parameter will enable or disable this path given the list of model we configure.

Now, we can see we have 2 types describing our input and outputs: `NM` and `NMResponse`. These should be defined in `src/api_classes.py`. These use Pydandic `BaseModel`. It will be used to automatically to match the `json` fields from the HTTP request and response. For instance:

```python
class NM(BaseModel):
    input_1: str
    input_2: str

class NMResponse(BaseModel):
    results: str
```
In this example, `NM` contains two fields, it can automatically be converted to `Dict` when calling `model_input.dict()`.

These are the 2 most important lines:
```python
w.workers["new_model"].feed(data_dict)
result = w.workers["new_model"].get_result()
```
The first one will select our "new_model" `IPUWorker` from the `IPUWorkerGroup` and feed the data dict to its input queue.
The second one will retrieve the results from the `IPUWorker` output queue.

Finally , return the results as a Dict to match `NMResponse` format.
Here we supposed our model prediction is available under the dict key `result["prediction"]`.

3 - Edit your config.

By default the server is not configured yet to run your model.
To add it you can either: modify the default config in `src/config.py` and add it to `models` list.
Or temporary, modify the `.env` file variable `MODELS` (or just set the environment variable `MODELS`) to add your model name "new_model" to the list. (You should make sure you have enough IPUs available to run all the models in the list).

4 - Now if you run the server and go to the `IP_ADDRESS:PORT/docs` url, you should be able to see and test your new API !

## Deployment:
2 Github workflows are enabled by:

`.github/workflows/deploy.yml`: (manually triggered) Build and deploy the artifact on the remote server - Sensitive information are configured via Github Secrets.

`.github/workflows/stop.yml`: (manually triggered) Stop the container on the remote server.
