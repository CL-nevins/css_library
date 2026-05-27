# Welcome to Xinference!

<div class="toctree" maxdepth="2" hidden="">

getting_started/index models/index user_guide/index examples/index
reference/index development/index

</div>

Xorbits Inference (Xinference) is an open-source platform to streamline
the operation and integration of a wide array of AI models. With
Xinference, you're empowered to run inference using any open-source
LLMs, embedding models, and multimodal models either in the cloud or on
your own premises, and create robust AI-driven applications.

## Developing Real-world AI Applications with Xinference

<div class="tabs">

<div class="code-tab">

python LLM

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

\# Chat to LLM model.chat( messages=\[{"role": "system", "content": "You
are a helpful assistant"}, {"role": "user", "content": "What is the
largest animal?"}\], generate_config={"max_tokens": 1024} )

</div>

<div class="code-tab">

python Embedding

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

model.create_embedding("What is the capital of China?")

</div>

<div class="code-tab">

python Image

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

model.text_to_image("An astronaut walking on the mars")

</div>

<div class="code-tab">

python Audio

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

with open("speech.mp3", "rb") as audio_file:  
model.transcriptions(audio_file.read())

</div>

<div class="code-tab">

python Rerank

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

query = "A man is eating pasta." corpus = \[ "A man is eating food.", "A
man is eating a piece of bread.", "The girl is carrying a baby.", "A man
is riding a horse.", "A woman is playing violin." \]
print(model.rerank(corpus, query))

</div>

<div class="code-tab">

python Video

from xinference.client import Client

client = Client("<http://localhost:9997>") model =
client.get_model("MODEL_UID")

model.text_to_video("")

</div>

</div>

## Getting Started

<div class="grid">

2

<div class="grid-item-card" link="installation" link-type="ref">

Install Xinference

Install Xinference on Linux, Windows, and macOS.

</div>

<div class="grid-item-card" link="using_xinference" link-type="ref">

Try it out!

Start by running Xinference on a local machine.

</div>

</div>

<div class="grid">

2

<div class="grid-item-card" link="models_builtin_index" link-type="ref">

Explore models

Explore a wide range of models supported by Xinference.

</div>

<div class="grid-item-card" link="models_custom" link-type="ref">

Register your own model

Register model weights and turn it into an API.

</div>

</div>

## Explore the API

<div class="grid">

2

<div class="grid-item-card" link="chat" link-type="ref">

Chat & Generate

Learn how to chat with LLMs in Xinference.

</div>

<div class="grid-item-card" link="tools" link-type="ref">

Tools

Learn how to connect LLM with external tools.

</div>

</div>

<div class="grid">

2

<div class="grid-item-card" link="embed" link-type="ref">

Embeddings

Learn how to create text embeddings in Xinference.

</div>

<div class="grid-item-card" link="rerank" link-type="ref">

Rerank

Learn how to use rerank models in Xinference.

</div>

</div>

<div class="grid">

2

<div class="grid-item-card" link="image" link-type="ref">

Images

Learn how to generate images with Xinference.

</div>

<div class="grid-item-card" link="multimodal" link-type="ref">

Multimodal

Learn how to process images and audio with LLMs.

</div>

</div>

<div class="grid">

2

<div class="grid-item-card" link="audio" link-type="ref">

Audio

Learn how to turn audio into text or text into audio with Xinference.

</div>

<div class="grid-item-card" link="video" link-type="ref">

Video

Learn how to generate video with Xinference.

</div>

</div>

<div class="grid">

2

<div class="grid-item-card" link="flexible" link-type="ref">

Flexible

Learn how to inference traditional ML models with Xinference.

</div>

</div>

## Getting Involved

<div class="grid" gutter="1">

<div class="grid-item">

<div class="div">

sd-font-weight-normal sd-fs-5

Get Latest News

</div>

<div class="grid" gutter="3">

1

<div class="grid-item-card" link="https://twitter.com/Xorbitsio">

`twitter` Follow us on Twitter

</div>

<div class="grid-item-card" link="https://zhihu.com/org/xorbits">

`zhihu` Read our blogs

</div>

</div>

</div>

<div class="grid-item">

<div class="div">

sd-font-weight-normal sd-fs-5

Get Support

</div>

<div class="grid" gutter="3">

1

<div class="grid-item-card"
link="https://xorbits.cn/assets/images/wechat_work_qr.png">

`weixin` Find community on WeChat

</div>

<div class="grid-item-card" link="https://discord.gg/Xw9tszSkr5">

`discord` Find community on Discord

</div>

<div class="grid-item-card"
link="https://github.com/xorbitsai/inference/issues/new/choose">

`github` Open an issue

</div>

</div>

</div>

<div class="grid-item">

<div class="div">

sd-fs-5

Contribute to Xinference

</div>

<div class="grid" gutter="3">

1

<div class="grid-item-card"
link="https://github.com/xorbitsai/inference/pulls">

`github` Create a pull request

</div>

</div>

</div>

</div>
