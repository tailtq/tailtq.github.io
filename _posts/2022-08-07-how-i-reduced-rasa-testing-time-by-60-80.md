---
title: How I reduced RASA testing time by 60-80%
author: Tai Le
date: 2022-08-06
tags: [Python, Back-end, AI, NLP]
---

It is quite a long time since I wrote my latest post, and now I come back stronger with a topic that contains both Back-end and AI. It is my journey to customize the Natural Language Understanding (NLU) testing process of the RASA platform to reduce the computation time. In other words, I applied **batch prediction** in the testing phase, which is currently not available in the pipeline. This post is probably long because I will list things that I modified, so please be patient and follow it until the end.

## 1. Technology

During the process, I only use **Python**, **RASA (v3.1)**, and the **Monkey Patching** technique to customize the RASA package. For people who don’t know about Monkey Patching yet, it is a technique to edit attributes at runtime, it can also be used to override functions/methods in libraries.

![Vargas, 2019](/assets/img/2022-08-06/monkey-patching.png)

RASA, on the other hand, is a famous platform to build chatbots that currently has 15k stars in its repository. It supports many NLP components to provide the pipeline having the best performance and high accuracy.

![RASA](/assets/img/2022-08-06/rasa.png)

## 2. DietClassifier

As I read from the RASA source, each component in the RASA pipeline has two main methods: `train` and `process`. The `train` method will be used in the training process, and the `process` one will be used in the testing process as well as when running RASA servers. Therefore, our ultimate goal is to find out why the `process` method cannot run in batches and make adjustments to improve it.

Because the model is often the slowest one, I will check the `DietClassifier` class to make improvements for it first. Looking at the code link below, we can see that the `process` method receives a list of messages as the parameter and loops through it. So we only need to refactor the `process` method, don’t we?

[https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/classifiers/diet_classifier.py#L1018-L1021](https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/classifiers/diet_classifier.py#L1018-L1021)


Unfortunately, there is only one message inside the list after adding logging to that function. Therefore, I had to look for where it is called. After reading through many methods, I found out that the data is passed through these methods below first before going through `process`:

1. `rasa.nlu.test.get_eval_data`: receive a list of messages, loop through them, and call parse_message.
2. `rasa.core.processor.MessageProcessor.parse_message`: pass each message to the above method and verify the result.
3. `rasa.core.processor.MessageProcessor._parse_message_with_graph`: pass each message to the pipeline.

The third one is the culprit which sends only one message to the pipeline at a time. Therefore, we need to customize this one first.

#### a. rasa.nlu.test.get_eval_data

In the code below, I only loop through all of the text, convert its format into messages, and forward them into the `MessageProcessor.parse_message`  function.

```python
async def get_eval_data(processor: MessageProcessor, test_data: TrainingData) -> Tuple[List, List, List]:
		"""
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/test.py#L1246-L1342
    """
		# measure time
    start_time = time.time()

    ...
    should_eval_entities = len(test_data.entity_examples) > 0

    # TODO: Pass all messages to the processor
    messages = [UserMessage(text=example.get(TEXT)) for example in test_data.nlu_examples]
    results = await processor.parse_message(messages, only_output_properties=False)
    # End change

    for i, result in enumerate(results):
				# access to the example using the index
        example = test_data.nlu_examples[i]
				...
```

#### b. rasa.core.processor.MessageProcessor.parse_message

We still have a lot of work besides editing the `get_eval_data` method. You can see that the `parse_message` method only receives one message at a time, so we must customize it as well.

In this function, I only edited the first parameter’s name and looped through the result to verify the format (`_check_for_unseen_features`) when it gets returned.

```python
async def parse_message(
    self, messages: List[UserMessage], only_output_properties: bool = True
) -> Dict[Text, Any]:
    """
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/core/processor.py#L619-L646
    """
    if self.http_interpreter:
        parse_data = await self.http_interpreter.parse(messages)
    else:
        parse_data = self._parse_message_with_graph(messages, only_output_properties)

    # TODO: parse_data before modifying returns only one item.
    #  Here I return multiple items and loop through them to modify
    for item in parse_data:
        logger.debug(
            "Received user message '{}' with intent '{}' "
            "and entities '{}'".format(
                item["text"], item["intent"], item["entities"]
            )
        )
        self._check_for_unseen_features(item)

    return parse_data
```

#### c. rasa.core.processor.MessageProcessor._parse_message_with_graph

In this method, instead of wrapping around a message with a list, I passed all messages to the pipeline (containing multiple components). Then I retrieved the data using `targets` via the key `self.model_metadata.nlu_target`, looped through them, and edited the results.

```python
def _parse_message_with_graph(
    self, messages: List[UserMessage], only_output_properties: bool = True
) -> Dict[Text, Any]:
    """
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/core/processor.py#L648-L673
    """
    results = self.graph_runner.run(
        inputs={PLACEHOLDER_MESSAGE: messages},  # TODO: receive multiple messages and pass to graph_runner
        targets=[self.model_metadata.nlu_target],
    )
    parsed_messages = results[self.model_metadata.nlu_target]
    # TODO: loop through message and update them
    for i, parsed_message in enumerate(parsed_messages):
        parsed_messages[i] = {
            TEXT: "",
            INTENT: {INTENT_NAME_KEY: None, PREDICTED_CONFIDENCE_KEY: 0.0},
            ENTITIES: [],
            **parsed_message.as_dict(only_output_properties=only_output_properties)
        }
    return parsed_messages
```

#### d. rasa.nlu.classifiers.diet_classifier.DIETClassifier.process

Adjusting these methods above only opens the entry gate for batch processing, there is still a lot of work to do. In `process`, we can see that it gets the predictions from `_predict` and does post-processing.

[https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/classifiers/diet_classifier.py#L1018-L1039](https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/classifiers/diet_classifier.py#L1018-L1039)

There are a few approaches here:

1. Modify the `_predict` method to accept all messages and the `self.model.run_inference` method will separate data into batches and do the batch prediction there.
2. Separate all messages into batches inside the `process` method and directly call `self.model.run_inference`

I tried both approaches and found out that only the second one worked. The first approach is very intuitive but it doesn’t work **because the predictions of many batches don’t share the same dimensions, so they cannot stack on top of each other**. I  could not figure out the root causes and I did not want to either because I am not the AI Engineer anymore *(just a Back-end guy)*.

The second approach is not impossible, we only need to understand the below methods:

1. `rasa.nlu.classifiers.diet_classifier.DIETClassifier._predict`: It only creates the `RasaModelData` object so we obviously can move to `process`.
2. `rasa.utils.tensorflow.models.RasaModel.run_inference`: It generates the data generator based on the `RasaModelData` object from the previous step, then does prediction concatenation *(we have to skip this one because of what I mentioned earlier)*.

Therefore, here is the new `run_inference` method:

```python
def RasaModel_run_inference(self, data: tuple) -> Dict[Text, Union[np.ndarray, Dict[Text, Any]]]:
    """
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/utils/tensorflow/models.py#L280-L322
    """
    # data_generator is a tuple of 2 elements - input and output.
    # We only need input, since output is always None and not
    # consumed by our TF graphs.
    batch_in = data[0]
    # TODO: Inference a batch of data instead of one message. I also removed the data concatenation between batches
    outputs: Dict[Text, Union[np.ndarray, Dict[Text, Any]]] = self._rasa_predict(batch_in)
    return outputs
```

And the new `process` method is here, you can look at the description in the code. Note that we need to define a global `BATCH_SIZE` variable to easily change it.

```python
def DIETClassifier_process(self, messages: List[Message]) -> TrainingData:
    """
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/classifiers/diet_classifier.py#L1018-L1039
    """
    print("Run DIETClassifier process")
    start_time = time.time()
    # TODO: Chunk messages into multiple smaller chunks with BATCH_SIZE = 256
    global BATCH_SIZE
		# create RasaModelData just like what _predict does
    model_data = self._create_model_data(messages, training=False)
    if model_data.is_empty():
        return messages
    # separate the data into batches
    (data_generator, _) = rasa.utils.train_utils.create_data_generators(
        model_data=model_data, batch_sizes=BATCH_SIZE, epochs=1, shuffle=False
    )
    data_iterator = iter(data_generator)
    message_counter = 0
    total_messages = 0
    while True:
        try:
            # TODO: pass batch of data to model.run_inference function and do post-processing here to
            #  ignore the result concatenation (if not, the concatenation will be handle in RasaModel_run_inference)
            batch = next(data_iterator)
            out = self.model.run_inference(batch)
            total_messages += len(out["diagnostic_data"]["attention_weights"])

            # TODO: post-processing
            while message_counter < total_messages:
                message = messages[message_counter]
                message_out = {
                    DIAGNOSTIC_DATA: {
                        "attention_weights": out[DIAGNOSTIC_DATA]["attention_weights"][message_counter % BATCH_SIZE],
                        "text_transformed": out[DIAGNOSTIC_DATA]["text_transformed"][message_counter % BATCH_SIZE],
                    },
                } if out else None

                if out and "i_scores" in out:
                    message_out["i_scores"] = out["i_scores"][message_counter % BATCH_SIZE]

                if out and "e_entity_ids" in out:
                    message_out["e_entity_ids"] = out["e_entity_ids"][message_counter % BATCH_SIZE][np.newaxis, ...]
                    message_out["e_entity_scores"] = out["e_entity_scores"][message_counter % BATCH_SIZE][np.newaxis, ...]

                if self.component_config[INTENT_CLASSIFICATION]:
                    label, label_ranking = self._predict_label(message_out)
                    message.set(INTENT, label, add_to_output=True)
                    message.set("intent_ranking", label_ranking, add_to_output=True)

                if self.component_config[ENTITY_RECOGNITION]:
                    entities = self._predict_entities(message_out, message)
                    message.set(ENTITIES, entities, add_to_output=True)

                if out and self._execution_context.should_add_diagnostic_data:
                    message.add_diagnostic_data(
                        self._execution_context.node_name, message_out.get(DIAGNOSTIC_DATA)
                    )
                message_counter += 1
        except StopIteration:
            break
    print(f"End DIETClassifier: {time.time() - start_time}")

    return messages
```

#### e. Do Monkey Patching

```python
from rasa.nlu import test
from rasa.core.processor import MessageProcessor
from rasa.nlu.classifiers.diet_classifier import DIETClassifier
from rasa.utils.tensorflow.models import RasaModel

test.get_eval_data = get_eval_data
MessageProcessor.parse_message = parse_message
MessageProcessor._parse_message_with_graph = _parse_message_with_graph
DIETClassifier.process = DIETClassifier_process
RasaModel.run_inference = RasaModel_run_inference
```

## 3. ConveRTFeaturizer

After customizing `DIETClassifier`, I recalled that the time was cut in half, but I had to wait for a long time to see the data passed to the `DIETClassifier`. Therefore, I kept investigating and found out that there is another bottleneck in `ConveRTFeaturizer`, this component also uses a third-party model. Therefore, we need to adjust this one too. 

#### a. rasa.nlu.featurizers.dense_featurizer.convert_featurizer. ConveRTFeaturizer.process

In NLU, I only use `TEXT` key, so I skipped `ACTION_TEXT`. I only used a function to chunk the data and looped through each chunk.

```python
def chunks(lst, n_chunks):
    """Yield successive n-sized chunks from lst."""
    chunk_size = len(lst) // n_chunks
    chunk_size = len(lst) if chunk_size == 0 else chunk_size

    for i in range(0, len(lst), chunk_size):
        yield lst[i:i + chunk_size]

def ConveRTFeaturizer_process(self, messages: List[Message]) -> List[Message]:
    """
    https://github.com/RasaHQ/rasa/blob/3.1.x/rasa/nlu/featurizers/dense_featurizer/convert_featurizer.py#L379-L395
    """
    print("Run ConveRTFeaturizer process")

    # TODO: Chunk messages into multiple smaller chunks with BATCH_SIZE = 256
    start_time = time.time()
    global BATCH_SIZE
    for message_chunk in chunks(messages, BATCH_SIZE):
        attribute = TEXT
        sequence_features, sentence_features = self._compute_features(message_chunk, attribute=attribute)
        self._set_features(message_chunk, sequence_features, sentence_features, attribute)
    print(f"End ConveRTFeaturizer: {time.time() - start_time}")

    return messages
```

#### b. Do Monkey Patching

```python
from rasa.nlu.featurizers.dense_featurizer.convert_featurizer import ConveRTFeaturizer

ConveRTFeaturizer.process = ConveRTFeaturizer_process
```

## 4. Run the new command

I mimic a part of all options that the `rasa test` command provides for some specific usage. Assume that the code is placed in the file `run_new_test.py`,  here is how we can operate it.

```python
# TODO Command:
#  python run_new_test.py \
#       --model models/model.tar.gz \
#       --nlu data \
#       --out results \
#       --batch-size 256 \
#       --no-plot

parser = ArgumentParser()
parser.add_argument("-m", "--model", type=str, default=get_latest_model(DEFAULT_MODELS_PATH))
parser.add_argument("-u", "--nlu", type=str, required=True)
parser.add_argument("-d", "--domain", type=str, default="domain.yml")
parser.add_argument("--out", type=str, default="results")
parser.add_argument("--batch-size", type=int, default=128)
parser.add_argument("--no-plot", action="store_true")
args = parser.parse_args()

BATCH_SIZE = args.batch_size
asyncio.run(
    test_nlu(args.model, args.nlu, output_directory=args.out, domain_path=args.domain, additional_arguments={
        "loglevel": None,
        "model": args.model,
        "stories": ".",
        "max_stories": None,
        "endpoints": "endpoints.yml",
        "fail_on_prediction_errors": False,
        "url": None,
        "evaluate_model_directory": False,
        "nlu": args.nlu,
        "config": None,
        "domain": args.domain,
        "cross_validation": False,
        "folds": 5,
        "runs": 3,
        "percentages": [0, 25, 50, 75],
        "disable_plotting": args.no_plot,
        "successes": False,
        "no_errors": False,
        "no_warnings": False,
        "out": args.out,
        "errors": True,
        "func": run_nlu_test
    })
)
```

## 5. Conclusion

I applied this code to my project and the time reduced significantly, around 60-80% for all models running on ~10MB datasets. I really enjoyed this task even though it seemed impossible at the beginning, the feeling is extremely like when I step out of my comfort zone.

That's it. I could modify a small part of the RASA's pipeline using my curiosity, AI Engineering, and Back-end knowledge. You can do that too by following these steps above and making the magic happens, good luck.