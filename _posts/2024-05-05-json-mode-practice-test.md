---
title: "Creating a practice test builder with OctoAI Json mode"
date: 2024-05-05
permalink: /posts/2024/05/json-mode-practice-test/
excerpt: Transform your study sessions with a custom AI-powered tool. Learn how to use OctoAI's JSON mode to create a personalized practice test builder that adapts to your strengths and weaknesses.
tags:
  - ai
  - octoai
  - python
---

I am currently taking calc 3 in university and I wanted to create a study tool for myself to practice. My goal was to create a tool that would give me practice questions based on my mastery of that certain topic, so I practice the things I am good at less than the topics I am struggling with. I know I could just do this manually but it seemed fun.

## The plan

For my class there are a lot of practice midterms that we can use to study and they are all stored as pdfs. So the first step would be to parse the pdf so I can plug it into an LLM. Then I would need the LLM to separate the test into different topics, so it could determine which parts I am good at and which parts need work.

Once it separated the test into topics, it would need to choose a topic to ask me a question on. I will talk about how to do that more later, but the general idea is to preference topics you have previously scored low on.

Once the LLM asks you the question you solve it and input your answer. Since the LLM might output the answer in a different form than you input, instead of auto grading you it will show you the work and it's answer and you can tell it whether you got it correct or incorrect.

Once it sees how you performed on that question it will update your proficiency in that topic accordingly, and the process will restart. Now time for implementation!

## Setup

Create a new folder and open it up in vscode. We need to create a virtual environment for dependency management. Run `python3 -m venv .venv` if you are on mac/linux or `python -m venv .venv` if you are on windows. Create a main.py file and restart your terminal with the new virtual environment.

## Loading the pdf

We are going to use PyPDF2 to read the Pdf so run `pip install PyPDF2`. I have my pdf saved in the root of my project, so now I can add the following code:

```
reader = PyPDF2.PdfReader('midterm1.pdf')
midterm = ""
for page in reader.pages:
    text = page.extract_text()
    midterm += text
```

Now the practice test is saved as a string which I can plug into the LLM.

## Connecting to Octo

We will be using OctoAI to host our model. First we need to get an API token. There are instructions on how to do that [here](https://octo.ai/docs/getting-started/how-to-create-an-octoai-access-token). Once you have your token you can set the environment variable with this command: `export OCTOAI_TOKEN=YOUR_TOKEN_HERE`.

Next we have to install Octo so run the following command: `pip install OctoAI`.

Now we can connect to the model with the following code:

```
import os
from octoai.client import OctoAI

OCTOAI_TOKEN = os.getenv('OCTOAI_TOKEN')

octoai = OctoAI(api_key=OCTOAI_TOKEN)
```

## Getting topics

We want to get a list of topics that are covered on the test. An easy way to do this is with Octo's Json mode. We can use pydantic to describe the shape we want out data to be. In our case we just want one list of topics:

```
from pydantic import BaseModel
from typing import List

class Test(BaseModel):
    topics: List[str]
```

Now we can use this schema when we query the LLM:

```
import json
from octoai.text_gen import ChatMessage, ChatCompletionResponseFormat

completion = octoai.text_gen.create_chat_completion(
    model="llama-2-70b-chat",
    messages=[
        ChatMessage(
            role="system",
            content="Below is an instruction that describes a task. Write a response that appropriately completes the request.",
        ),
        ChatMessage(role="user", content="I will give you a practice midterm. Please give me a list of topics covered on the midterm. Each topic should be a string the array" + midterm),
    ],
    response_format=ChatCompletionResponseFormat(
        type="json_object",
        schema=Test.model_json_schema()
    ),
)

topics = json.loads(completion.choices[0].message.content)["topics"]
```

Now topics holds a lists of each topic on the test. We also want to keep track of how well you are doing on each section so I am going to create a dictionary that keeps track of your scores.

```
scores = {key: 0 for key in topics}
```

## Generating questions

Now that we have the topics we have to generate questions. The first step is deciding which topic we should ask a question on. We need to come up with some way to determine which topic to choose based off of the current proficiency scores.

To do this we are basically going to calculate the difference between the score and 100 and use that as a weight for our random choice. This means that if you have a proficiency score of 100 you won't get any more questions on that topic. This is how we can implement it.

```
import random

def get_next_topic():
    inverted_weights = [100 - score for score in scores.values()]
    return random.choices(topics, weights=inverted_weights, k=1)[0]
```

Now that we know which topic we are going to ask a question for we have to actually ask the question. Since we want the question, the answer, and the work for the solution we are again going to use Json mode.

Lets define the schema:

```
class Problem(BaseModel):
    question: str
    work: str
    answer: str
```

And now we are going to start a while loop so it will keep asking questions until we decide we are done studying. Inside the while loop we are going to generate the question and solution, and some other stuff I will cover later.

```
running = True

while running:
    topic = get_next_topic()
    completion = octoai.text_gen.create_chat_completion(
        model="llama-2-70b-chat",
        messages=[
            ChatMessage(
                role="system",
                content="Below is an instruction that describes a task. Write a response that appropriately completes the request.",
            ),
            ChatMessage(role="user", content="Please give me a question about " + topic + " it should not be taken from the midterm but it should be heavily inspired by the midterm. You should also solve the question and provide the work in the work section and the final answer in exact form in the answer section. Do not round."),
        ],
        response_format=ChatCompletionResponseFormat(
            type="json_object",
            schema=Problem.model_json_schema()
        ),
    )

    problem = completion.choices[0].message.content
    question = json.loads(problem)["question"]
    work = json.loads(problem)["work"]
    correct_answer = json.loads(problem)["answer"]
```

Now that we have the question and the solution, the only thing left to do is determine whether or not the answer was correct and adjust the scores accordingly.

Since its pretty hard to determine correctness programmatically (they might input the answer in a different format etc...), we are just going to ask the user if they got the answer correct.

```
    print("Question: " + question)
    given_answer = input("What is your answer?" + "\n")

    print("Correct Answer: " + correct_answer)
    print("Work: " + work)

    while True:
      response = input("Did you get the answer correct? (y/n)")
      if response == "y":
          if scores[topic] < 100:
            scores[topic] += 10
          print("Your new score for " + topic + " is " + str(scores[topic]))
          break
      elif response == "n":
          if scores[topic] > 10:
              scores[topic] -= 10
          print("Your new score for " + topic + " is " + str(scores[topic]))
          break
      else:
          print("Invalid input. Please enter 'y' for yes or 'n' for no.")

    while True:
      response = input("Would you like to continue? (y/n)")
      if response == "y":
          get_next_topic()
          break
      elif response == "n":
          running = False
          break
      else:
          print("Invalid input. Please enter 'y' for yes or 'n' for no.")
```

If the user wants to continue it will restart the whole process with a new random topic based on the updated proficiency scores.

## Conclusion

This was just a simple tool I wanted to build and thought it could be cool to incorporate AI into it. It has a long way to go before it will help me ace my next test, but I like this direction of AI helping me learn about complex topics and I want to keep advancing it.

A couple things I would like to add would be a better way of keeping track of proficiency. Right now I just increment or decrement by a fixed amount and it might be cool to have some more complicated algorithm to determine proficiency. I also think it would be interesting if I could take a picture of my work and it would analyze the correctness. That way if you made one small mistake it wouldn't penalize you too much, similar to the way the real tests are graded.

Also this is all just in the terminal. It would be neat to have some sort of UI to interact with it. Also all of the variables are just stored on the machine so it wouldn't be able to remember your proficiency from the last session, so hooking it up to a database might be cool.

## Entire code

```
import json
import random
import PyPDF2
import os
from octoai.client import OctoAI

from pydantic import BaseModel
from typing import List

from octoai.text_gen import ChatMessage, ChatCompletionResponseFormat

reader = PyPDF2.PdfReader('midterm1.pdf')
midterm = ""
for page in reader.pages:
    text = page.extract_text()
    midterm += text

OCTOAI_TOKEN = os.getenv('OCTOAI_TOKEN')

octoai = OctoAI(api_key=OCTOAI_TOKEN)

class Test(BaseModel):
    topics: List[str]

completion = octoai.text_gen.create_chat_completion(
    model="llama-2-70b-chat",
    messages=[
        ChatMessage(
            role="system",
            content="Below is an instruction that describes a task. Write a response that appropriately completes the request.",
        ),
        ChatMessage(role="user", content="I will give you a practice midterm. Please give me a list of topics covered on the midterm. Each topic should be a string the array" + midterm),
    ],
    response_format=ChatCompletionResponseFormat(
        type="json_object",
        schema=Test.model_json_schema()
    ),
)

topics = json.loads(completion.choices[0].message.content)["topics"]

scores = {key: 0 for key in topics}

def get_next_topic():
    inverted_weights = [100 - score for score in scores.values()]
    return random.choices(topics, weights=inverted_weights, k=1)[0]

class Problem(BaseModel):
    question: str
    work: str
    answer: str

running = True

while running:
    topic = get_next_topic()
    completion = octoai.text_gen.create_chat_completion(
        model="llama-2-70b-chat",
        messages=[
            ChatMessage(
                role="system",
                content="Below is an instruction that describes a task. Write a response that appropriately completes the request.",
            ),
            ChatMessage(role="user", content="Please give me a question about " + topic + " it should not be taken from the midterm but it should be heavily inspired by the midterm. You should also solve the question and provide the work in the work section and the final answer in exact form in the answer section. Do not round."),
        ],
        response_format=ChatCompletionResponseFormat(
            type="json_object",
            schema=Problem.model_json_schema()
        ),
    )

    problem = completion.choices[0].message.content
    question = json.loads(problem)["question"]
    work = json.loads(problem)["work"]
    correct_answer = json.loads(problem)["answer"]

    print("Question: " + question)
    given_answer = input("What is your answer?" + "\n")

    print("Correct Answer: " + correct_answer)
    print("Work: " + work)

    while True:
      response = input("Did you get the answer correct? (y/n)")
      if response == "y":
          if scores[topic] < 100:
            scores[topic] += 10
          print("Your new score for " + topic + " is " + str(scores[topic]))
          break
      elif response == "n":
          if scores[topic] > 10:
              scores[topic] -= 10
          print("Your new score for " + topic + " is " + str(scores[topic]))
          break
      else:
          print("Invalid input. Please enter 'y' for yes or 'n' for no.")

    while True:
      response = input("Would you like to continue? (y/n)")
      if response == "y":
          get_next_topic()
          break
      elif response == "n":
          running = False
          break
      else:
          print("Invalid input. Please enter 'y' for yes or 'n' for no.")
```
