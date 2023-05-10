---
title:  "Streamline Your Django / Python Code Reviews with a Custom Git Workflow and Chat GPT"
date:   2023-05-05 12:34:48 +0300
categories:
    - python
    - ai
    - chatgpt
    - git
author: "Can"
excerpt: "Learn about how to review incoming PR's with GPT"
header:
    overlay_image: /assets/images/chatgpt.jpeg
    overlay_filter: linear-gradient(rgba(2, 0, 36, 0.5), rgba(0, 146, 202, 0.5))
slug: pr-review-with-chatgpt
sidebar:
    nav: docs
layout: custom
---

In today's fast-paced development environment, code reviews are essential for maintaining code quality and ensuring 
that your team's work is consistent and efficient. 
Integrating Chat GPT and the OpenAI API into your Git workflow can revolutionize the way 
you conduct code reviews, making them more interactive and efficient. 
In this blog post, we will explore how to create a custom Git workflow that leverages 
the power of Chat GPT to provide real-time feedback and suggestions on your code, 
ultimately improving your team's productivity and code quality.

## What is Chat GPT?

Chat GPT is an AI chatbot that uses the OpenAI API to generate responses to user input.
It can be used for a variety of purposes, including customer service, code generation, and more.
The bot is trained on a large dataset of conversations, which allows it to generate responses that are
both relevant and natural-sounding.

Chat GPT is fueled by 
- `prompts` which are tasks, questions, information, etc. that you pass into the model
- `parameters` which are the settings and knobs that you can tweak to change the behavior of the model
- and generates `outputs` based on the prompts and parameters you pass in.

We will dive into the details of how to use Chat GPT in the following sections.

## Why Use Chat GPT for Code Reviews?

Code reviews are an essential part of the development process, but they can be time-consuming and tedious.
Chat GPT can help streamline this process by providing real-time feedback and suggestions on your code.
This allows you to focus on the more important aspects of your work, such as writing new features or fixing bugs.

## How to Create a Custom Git Workflow

Quoting the definition of a `git workflow` from [Github](https://docs.github.com/en/actions/using-workflows/about-workflows):

> *A workflow is a configurable automated process made up of one or more jobs.*


Basically, a `workflow` is a pipeline which gets triggered by an event in your repository
that runs a series of jobs either sequentially or in parallel.
Each job can either run a `Git Action` or a `script` to accomplish a task and will run in its
own virtual machine runner, which are containers with isolated environments that spin up to
run your workflow.

To create a `workflow`, add a `.github/workflows` directory to your repository and create a `yaml` file
with the name of your workflow.
In our case, let's create a `reviewer.yml` file in the `.github/workflows` directory with the
following code as a start:

{% highlight yaml %}
name: Review Pull Request

on:
  pull_request:
    branches:
      - master
    types: [ labeled ]

jobs:
  review_pull_request:
    if: ${{ "{{" }} github.event.label.name == 'review' {{"}}}}
    runs-on: custom-runner
    steps:
      - name: Check out repository
        uses: actions/checkout@v2

{% endhighlight %}

Let's review the anatomy of this workflow:

- `name`: The name of the workflow. This is displayed in the Actions tab of your repository.
- `on`: The event that triggers the workflow. In our case, we want to trigger the workflow when a pull request opened against the `master` branch is labeled as `review`.
- `jobs`: The jobs that run in the workflow. In our case, we only have one job called `review_pull_request`.
- `runs-on`: The type of runner that the job runs on. In our case, we want to run the job on a `custom-runner` which is a custom runner that we have set up for our repository.
- `steps`: The steps that run in the job. In our case, we only have one step called `Check out repository` which checks out the repository so that we can access the files in it.
- `uses`: The action that the step uses. In our case, we are using the `actions/checkout@v2` action which checks out the repository.

Next, let's add a Github action that will configure the Django environment and install the dependencies. 
The name of this action will be `setup.yml` and it will be located in the `.github/actions` directory.

{% highlight yaml %}
name: "Setup"
runs:
  using: "composite"
  steps:
    - name: Install pipenv
      run: pipx install pipenv
      shell: bash

    - name: Setup python
      uses: actions/setup-python@v3
      with:
        python-version: 3.9.13
        cache: "pipenv"

    - name: Sync dependencies
      run: pipenv sync --dev
      shell: bash
{% endhighlight %}

This is a composite Git action, which means that it is made up of a collection of steps that can be reused across multiple workflows.
The `runs` section specifies the steps that run in the action.
In our case, we have three steps:
- `Install pipenv`: Installs the `pipenv` package manager.
- `Setup python`: Sets up the Python environment. We are using the `actions/setup-python@v3` action to do this. This action will automatically install the specified version of Python and cache it for future use.
- `Sync dependencies`: Syncs the dependencies in the `Pipfile` with the `Pipfile.lock` file. We are using the `pipenv sync --dev` command to do this, which will install all of the dependencies in the `Pipfile` and their dependencies recursively.

Now finally we can use this custom action in our `reviewer.yml` workflow:
    
{% highlight yaml %}
...
...
jobs:
  ai_security_check_for_pull_requests:
    ...
    ...
    steps:
      - name: Check out repository
        uses: actions/checkout@v2

      - name: Setup dependencies
        uses: ./.github/actions/setup
{% endhighlight %}

We have to setup Python and install the dependencies before we can run the Chat GPT model, since we will define a custom Django management command and run it with the `manage.py` command.

## How to Define a Custom Management Command in Django

Django provides a convenient way to define custom management commands that can be run with the `manage.py` command.
To define a custom management command, create a `management/commands` directory in your Django app and create a `python` file with the name of your command.
In our case, we will create a `review.py` file in the `management/commands` directory with the following code:

{% highlight python %}
import djclick as click

from ..review.reviewer import main


@click.command()
@click.option(
    "-s",
    "--strategy",
    required=False,
    type=click.Choice(["security", "nit-pick", "syntactic-sugar"], case_sensitive=False),
    default="security",
    help="The strategy to use for the PR check.",
)
def command(strategy):
    main(strategy)

{% endhighlight %}

This command will take in a `strategy` argument which will be used to determine which strategy to use for the PR check.

We can add strategies like `security` that would only check for security issues, 

`nit-pick` that would only check for nit-picky issues, and 

`syntactic-sugar` that would only check for syntactic sugar issues.

Here, we are leveraging the djclick library to create the command, which provides a convenient way to create Django management commands with the click library.
The `main` function will be defined in the `reviewer.py` file, which we will create in the `review` directory.
Let's create this file and add the following code:

{% highlight python %}
import os

import requests

# GPT parameters
MAX_TOKENS = 300
PATCH_MAX_LENGTH = 5500
TEMPERATURE = 0.8
STOP_WORD = "Nothing found"

# Prompts
SECURITY_ANALYZER_USER_PROMPT = "Analyze the following code snippet for security and privacy issues:\n\nCode:\n"
SECURITY_ANALYZER_SYSTEM_PROMPT = f"""
You're a helpful application security expert skilled in Python and Django, looking for security and privacy issues
in a code.
You will be given a code snippet which contains a git patch and you will need to provide a list of security and privacy
issues you find in the code and sort it by severity level.
You should take OWASP Top 10 and CWE Top 25 into consideration.
For each vulnerability you find, point out exactly which statement causes the vulnerability and wrap
that statement in backticks to format it for markdown.
For each vulnerability you find, also provide a mitigation for it using the framework or programming language
that the code snippet is written in.
If there is no significant security issue found, respond with '{STOP_WORD}'
Limit the length of your response to {MAX_TOKENS} words or less.
"""


class GitClient:
    def __init__(self):
        self.repo = os.environ["GH_REPOSITORY"]
        self.pull_request = os.environ["GH_EVENT_PULL_REQUEST_NUMBER"]
        self.token = os.environ["GH_TOKEN"]
        GH_API = requests.Session()
        GH_API.headers.update({"Content-Type": "application/json", "Authorization": f"Bearer {self.token}"})
        self.api = GH_API
        self.last_commit = None

    def get_pull_request_files(self):
        pr_files_response = self.api.get(
            f"https://api.github.com/repos/{self.repo}/pulls/" f"{self.pull_request}/files"
        ).json()
        return pr_files_response

    def get_changed_file_shas(self):
        pr_commits = self.api.get(
            f"https://api.github.com/repos/{self.repo}/pulls/" f"{self.pull_request}/commits"
        ).json()
        last_commit = pr_commits[len(pr_commits) - 1].get("sha")
        self.last_commit = last_commit
        files_in_commit = self.api.get(f"https://api.github.com/repos/{self.repo}/commits/{last_commit}").json()[
            "files"
        ]
        return [file.get("sha") for file in files_in_commit if file.get("status") != "removed"]

    def add_comment_on_code_snippet(self, issues, path, start_line):
        start_line = 1 if start_line < 1 else start_line
        comments_response = self.api.post(
            f"https://api.github.com/repos/{self.repo}/pulls/{self.pull_request}/comments",
            json=dict(
                body=issues,
                path=path,
                commit_id=self.last_commit,
                start_side="RIGHT",
                line=start_line,
                side="RIGHT",
            ),
        )
        if comments_response.ok:
            print("Successfully commented on PR.")


class CodeReviewer:
    def __init__(self, system_prompt, user_prompt):
        self.system_prompt = system_prompt
        self.user_prompt = user_prompt

        self.git_client = GitClient()

        OPENAI_API = requests.Session()
        OPENAI_API.headers.update(
            {"Content-Type": "application/json", "Authorization": f"Bearer {os.environ['OPENAI_TOKEN']}"}
        )
        self.openai_api = OPENAI_API
        self.openai_model = "gpt-4"

    def execute(self):
        try:
            pr_files = self.git_client.get_pull_request_files()
            changed_file_shas = self.git_client.get_changed_file_shas()
            for file in pr_files:
                if file["status"] == "removed" or file["sha"] not in changed_file_shas:
                    continue

                patch = file.get("patch", "")
                if not patch or len(patch) > PATCH_MAX_LENGTH:
                    continue

                issues = self.analyze_code(patch)
                if not issues or STOP_WORD in issues:
                    print(STOP_WORD)
                else:
                    print(f"Issues Found: {issues}")
                    file_path = file.get("filename")
                    start_line = int(patch.split()[1].split(",")[0][1:]) if patch.startswith("@@") else 1
                    self.git_client.add_comment_on_code_snippet(issues, file_path, start_line)
        except requests.exceptions.RequestException as e:
            print("Error analyzing code:", e)
            raise e

    def analyze_code(self, patch):
        try:
            response = self.openai_api.post(
                "https://api.openai.com/v1/chat/completions",
                json={
                    "model": self.openai_model,
                    "messages": [
                        {
                            "role": "system",
                            "content": self.system_prompt,
                        },
                        {"role": "user", "content": f"{self.user_prompt} {patch}"},
                    ],
                    "n": 1,
                    "temperature": TEMPERATURE,
                    "max_tokens": MAX_TOKENS,
                },
            )
            return response.json()["choices"][0]["message"]["content"].strip()
        except requests.exceptions.RequestException as e:
            print("Error analyzing code:", e)
            raise e


def main(strategy):
    system_prompt = None
    user_prompt = None

    if strategy == "security":
        system_prompt = SECURITY_ANALYZER_SYSTEM_PROMPT
        user_prompt = SECURITY_ANALYZER_USER_PROMPT
    else:
        print("Error: Invalid strategy.")
        exit(1)

    if system_prompt and user_prompt:
        code_analyzer = CodeReviewer(system_prompt, user_prompt)
        code_analyzer.execute()

{% endhighlight %}

## The Management Command Code

- I've defined various GPT parameters to tweak the model's behavior. 
  - Namely, the `temperature` setting controls the randomness of the model's output. Higher temperature values will lead to more creative results, whereas
  lower temperature values will lead to more predictable results. Lower temperature values are especially useful when the answer is based on facts and should be deterministic.
  Here, we've set the `temperature` to `0.8` to get a balance between creativity and predictability, since we want the model to be creative in finding issues, but we also want the mitigations to be based on facts.
  - The `max_tokens` setting controls the maximum number of tokens that the model will generate. We've set this to `300` since we don't want the model to generate too much text and the API bills based on the number of tokens generated.
  - It is generally considered best practice to use a stop word to indicate whether the model generated a successful response. Here, if the model is not able to find any issues, it will return the stop word `Nothing found`.
- `GH_TOKEN`, `GH_REPOSITORY`, `GH_EVENT_PULL_REQUEST_NUMBER`, and `OPENAI_TOKEN` are environment variables that are passed into the workflow job step as secrets.
- We have defined a `GitClient` class that is responsible for interacting with the GitHub API. 
  - The `get_pull_request_files` method returns the list of files that were changed in the pull request. 
  - The `get_changed_file_shas` returns the list of file SHAs that were changed in the pull request. 
  - The `add_comment_on_code_snippet` adds a comment on the pull request with the issues found in the code snippet. It takes in a `start_line` parameter which is used to indicate the line number where the code snippet starts in the file.
- We have a `CodeReviewer` class that is responsible for analyzing the code. It has a method `execute` that iterates over the files that were changed in the pull request and analyzes the code in each file. 
Note that if the file is marked as `removed` or was not modified in the latest commit, the class does not conduct a review for that file.
It also has a method `analyze_code` that uses the OpenAI API to analyze the code snippet and return the issues found in the code.
- We have a `main` function that is responsible for executing the code analyzer. It creates an instance of the `CodeReviewer` class and calls the `execute` method.
- We have a `security` strategy that is responsible for analyzing the code for security issues. It has a `system_prompt` and a `user_prompt` that are used to analyze the code.
- The system prompt is the prompt that is used to initialize the conversation and set the model's behavior. The user prompt is the prompt that is used to analyze the code. The `user_prompt` is appended with the code snippet that is being analyzed.

## Final Touches

Let's wrap up this project by finalizing our Git workflow.
Add a final job step to the `reviewer.yml` file:

{% highlight yaml %}
jobs:
  review_pull_request:
    if: ${{ "{{" }} github.event.label.name == 'review' {{"}}}}
    runs-on: custom-runner

    steps:
      ...
      ...

      - name: Finding security and privacy code vulnerabilities
        id: ai_security_check
        run: pipenv run python manage.py review
        env:
          GH_TOKEN: ${{ "{{" }} secrets.GH_TOKEN {{"}}}}
          GH_REPOSITORY: ${{ "{{" }} github.repository {{"}}}}
          GH_EVENT_PULL_REQUEST_NUMBER: ${{ "{{" }} github.event.number {{"}}}}
          OPENAI_TOKEN: ${{ "{{" }} secrets.OPEN_AI_KEY {{"}}}}
{% endhighlight %}

Here, we're passing the tokens, the pull request number and the repository name to the management command as environment variables.
Note that the workflow has access to the repository name and the pull request number via the `github` context.
Also, make sure to run your management command via `pipenv run python manage.py review` so that the management command is run in the virtual environment.

Here is the final workflow:

{% highlight yaml %}
name: AI Security Check for Pull Requests

on:
  pull_request:
    branches:
      - master
    types: [ labeled ]

jobs:
  review_pull_request:
    if: ${{ "{{" }} github.event.label.name == 'review' {{"}}}}
    runs-on: custom-runner

    steps:
      - name: Check out repository
        uses: actions/checkout@v2

      - name: Setup dependencies
        uses: ./.github/actions/setup

      - name: Finding security and privacy code vulnerabilities
        id: ai_security_check
        run: pipenv run python manage.py review
        env:
          GH_TOKEN: ${{ "{{" }} secrets.GH_TOKEN {{"}}}}
          GH_REPOSITORY: ${{ "{{" }} github.repository {{"}}}}
          GH_EVENT_PULL_REQUEST_NUMBER: ${{ "{{" }} github.event.number {{"}}}}
          OPENAI_TOKEN: ${{ "{{" }} secrets.OPEN_AI_KEY {{"}}}}
{% endhighlight %}

## Findings on Prompts

At the outset of this project, ChatGPT produced satisfying results, though the responses were a bit too generic.
In some of the responses, the model responded with the CVE number of the vulnerability, which was a good sign, but
at subsequent responses, the model would not attach the CVE number, so it was not consistent.

To address this issue, I decided to adding more context via `Role Prompting`, which is a technique that OpenAI recommends
to improve the model's performance. The idea was to assign a `role` to the model as if it were a human. Rather than
prompting `You're a helpful application security expert`, I refined the prompt to `.. helpful application security expert skilled in Python and Django`.
This helped the model to find mitigations specific to Python / Django which are more fine-tuned and fine-grained.

The model also surprisingly recommended 3rd party packages that could be used to mitigate the vulnerability,
which was a good sign. I was assuming that this was a direct outcome of both the `Role Prompting` technique and
the high `temperature` setting, however lowering the `temperature` still yielded to similar results in subsequent prompts.

Furthermore, the prompt uses nested prompts to break down complex questions into simpler steps like 
`Sort it by severity level` or `Point out exactly which statement causes the vulnerability` or `wrap the statement in backticks`.
By breaking down the question into simpler and concise steps, the model was able to provide more accurate responses.

- Sample output with parameters `temperature=0.8` and `max_tokens=300`:

![alt](/assets/images/img.png)

The only problem left unaddressed was that the `max_tokens` setting caused the model to stop abruptly, 
which resulted in incomplete responses. To tackle this, I've decided to remove this setting and instead add
a prompt `Limit the issues found to 3 issues.`. This solved the problem of incomplete responses. In the
following prompt, I've also set the `temperature` to `0.2` to get more predictable results, however I think
there is no tone change between the previous prompt and this one.

- Sample output with parameters `temperature=0.2` and no `max_tokens` setting:

![alt](/assets/images/img_1.png)

## Conclusion

There you have it folks! We've successfully built a code analyzer that can analyze code for security issues and
provide mitigations. The model is not perfect, but it is a good start. I'm sure that with more training data and
more fine-tuning, we can get better results. I'm also planning to add more strategies to the code analyzer, so stay tuned!

## Useful Links

[Github Actions and Workflows](https://docs.github.com/en/actions/using-workflows/about-workflows)

[Prompt Engineering Best Practices](https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-openai-api)

[Github API Reference](https://docs.github.com/en/rest?apiVersion=2022-11-28)

[OpenAI Completions API Reference](https://platform.openai.com/docs/api-reference/completions/create)


So this was all for today! Enjoy the rest of your day!