*This is a submission for the [Cloudflare AI Challenge](https://dev.to/devteam/join-us-for-the-cloudflare-ai-challenge-3000-in-prizes-5f99).*

## What I Built

<!-- Tell us what your app does! -->

I have developed a website that lets AI and the user **tell stories together**. The first story part is provided by the AI, then you can provide a new part and so on until the story is complete.

During each interaction, the **AI will make an image for each story part**. In case you decide not to continue, the AI will generate the final part of the story.

Additionally, the website also lets you **change the language you see it in**. Right now, English and Spanish are available.



## Demo
[Demo Here!](https://ai-storytelling.pages.dev/)
<!-- Share a link to your deployed solution on Workers or Pages, and add a screenshot or video embed of your app here. -->

{% embed https://youtu.be/jiLvIYLGpEw?si=hnC-OWQMRKyycIiU %}

## My Code
https://github.com/julianandreszb/ai-storytelling


## Journey

<!-- Tell us about your process, the task types and models you used, what you learned, anything you are particularly proud of, what you hope to do next, etc. -->

This challenge was a perfect way to explore, learn, and integrate Cloudflare Pages and AI Workers and make them work together. 

All the documentation was easy to follow and implement. Also, it helped me a lot the [Cloudflare Workers](https://www.youtube.com/@CloudflareWorkers) YouTube channel. (Thanks for suggesting “Hono framework”!)

**What I learned?**
- How easy it is to create workers and integrate them with @cloudflare/ai package. 
- How to deploy the front-end project on “Cloudflare Pages”. I used my favorite front-end framework (Vue.js) easily.
- I could explore the different AI Worker Models

**What's next?**
- Integrate “**Automatic Speech Recognition**”, so the user can enter text OR talk directly to the AI to generate story parts.
- Integrate “**Object Detection**”, so the user can upload images and create a story part based on objects that are detected. 
- Refactor the whole project. Due to time constraints, I couldn't apply the best design patterns.


**Multiple Models and/or Triple Task Types**

**Multiple Models**: Image generator. First, it [Summarizes](https://developers.cloudflare.com/workers-ai/models/bart-large-cnn/) the story part prompt, and then it [Generates](https://developers.cloudflare.com/workers-ai/models/stable-diffusion-xl-lightning/) the image. (Multiple model, same task on the worker side)

**Triple Task Types**: 
1. Translate between English<->Spanish
1. Generate text for story parts
1. Generate images for each story part

| Task    | Type | Model |
| -------- | ------- |------- |
| Translate English<->Spanish | Translation| [m2m100-1.2b](https://developers.cloudflare.com/workers-ai/models/m2m100-1.2b/) |
| Generate story parts| Text Generation    | [llama-2-7b-chat-int8](https://developers.cloudflare.com/workers-ai/models/llama-2-7b-chat-int8/)
| Summarize prompts for image generation| Summarization | [bart-large-cnn](https://developers.cloudflare.com/workers-ai/models/bart-large-cnn/)|
| Generate Images | Text-to-Image | [stable-diffusion-xl-lightning](https://developers.cloudflare.com/workers-ai/models/stable-diffusion-xl-lightning/)|



<!-- Let us know if your project utilized multiple models per task and/or if your project used three or more task types. If so, you may qualify for our additional prize categories! If not, please remove this section. -->

<!-- Team Submissions: Please pick one member to publish the submission and credit teammates by listing their DEV usernames directly in the body of the post. -->

<!-- Don't forget to add a cover image or an additional tag to this post (if you want). -->

<!-- Thanks for participating! -->
