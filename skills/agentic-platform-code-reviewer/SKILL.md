---
name: agentic-platform-code-reviewer
description: "When reviewing code, check for code smells, documentation drift, and configuration misalignment. Provide a summary of findings and detailed explanations for each issue in a structured format."
disable-model-invocation: false
user-invocable: true
---

# Agentic Platform Code Reviewer

## Inputs: what to read
- Files included in the pull request,
- Only imported modules that the changed lines actually
  call.
- Modules not imported by the files affected by the pull request, but mentioned in code comments related to the code directly affected by the pull request.
- Other environment files that are not in the pull request but are related to the files being reviewed.
Those markdown documentation pages:
    - AGENT.md
    - CODING-PRACTICE.md

## Inputs: what NOT to read
Files that do not fit in the categories above.

## What to check
### Afraid to fail
The code is overly cautious, with excessive error handling or defensive programming that makes it hard to read or maintain. If it is expected that the code will fail, it should fail fast and clearly and the error thrown should be specific and actionnable (as much as possible). The code should not either create too many Error custom classes, or use generic Error classes that are not specific enough. The code should also not be too defensive, for example by checking for null values in places where it is not necessary.
Exemple:
```
def create_foo() -> dict:
    response = requests.get('https://api.github.com/events')
    if response.status_code != requests.codes.ok:
        return {'status_code': response.status_code, 'foo': None}
    return {
        'status_code': response.status_code,
        'foo': Foo(**response.json())
    }

foo_response: dict = create_foo()
if foo_response['status_code'] == requests.codes.ok:
    foo: Foo = foo_response['foo']
```
or 
```
try: 
    res = failing_func()
except:
    print("oops")
    res = None
return res
```
### Code duplication
The code contains repeated logic or patterns that could be abstracted into reusable functions or modules. The code should follow DRY (Don't Repeat Yourself) principles to reduce duplication and improve maintainability.  (**limit investigation to the file being reviewed, not the whole codebase**). This also aplies to duplicated code in tests, where use of test helpers abd fixtures should be used to reduce duplication. This also applies to alternative classes with different interfaces that implement the same logic, propose the use of a single interface and multiple implementations instead of duplicating logic across different classes.
Flag repeated logic when it represents the same behavior and could drift. Extracting a function is one common way to address this
Exemple:
```
def create_chat_request(model, messages, temperature):
    if not model.strip():
        raise ValueError("model is required")
    if not messages:
        raise ValueError("messages cannot be empty")
    if not 0 <= temperature <= 2:
        raise ValueError("temperature must be between 0 and 2")
    # Build chat request...


def create_streaming_request(model, messages, temperature):
    if not model.strip():
        raise ValueError("model is required")
    if not messages:
        raise ValueError("messages cannot be empty")
    if not 0 <= temperature <= 2:
        raise ValueError("temperature must be between 0 and 2")
    # Build streaming request...

```
### Silent fallbacks
The code should have clear and explicit happy paths for the deployed application. If some configurations must be handled differently because we are in a different environment (ex.: local vs production), we should make sure that the code is explicit about the differences and that it does not silently fallback to a default behavior that may not be appropriate for the current environment. Hardcoded values or behaviors that are not be appropriate for all environments are flagged as well.
Exemple:
```
# No target_url is passed when deploying locally 
target_url = os.environ.get("target_url", "127.0.0.1")
```
### Shotgun surgery
Introducing a change in one part of the codebase requires making changes in many other parts of the codebase. This can be a sign of poor separation of concerns or tight coupling between different modules. Heavy coupling needs to be flag during review as possible technical debt. Provide an overview (two lines max) of what the design should aim to.
Exemple:
```
class UserDataProcessor:
    def process_profiles(self, users):
        # 1. Start timer setup
        start_time = time.time()
        try:
            for user in users:
                # Actual business logic
                print(f"Processing user: {user}")
        except Exception as e:
            # 2. Duplicated error handling format
            logging.error(f"[SYSTEM FAILURE] Step failed: {str(e)}")
            raise
        finally:
            # 3. Duplicated metrics logging
            duration = time.time() - start_time
            print(f"[METRIC] process_profiles took {duration:.4f}s")

class FinancialReportProcessor:
    def generate_ledger(self, transactions):
        # Duplicated timer setup
        start_time = time.time()
        try:
            for tx in transactions:
                # Actual business logic
                print(f"Auditing transaction: {tx}")
        except Exception as e:
            # Duplicated error handling format
            logging.error(f"[SYSTEM FAILURE] Step failed: {str(e)}")
            raise
        finally:
            # Duplicated metrics logging
            duration = time.time() - start_time
            print(f"[METRIC] generate_ledger took {duration:.4f}s")
```


### Documentation drift
In addition to these code smells, validate that the documentation is up to date and accurately reflects the current state of the codebase. This includes inline comments, README files, and any other relevant documentation. 
Some of the documentation is markdown files that are related to coding practices. Flag it when the code reviewed does not follow the practices described in these markdown files. 
Docstring style should be consistent and follow the aggreed upon convention stated in the documentation.

### Configuration misalignment
When the pull request updates environment files, validate that the variables value between the environments are consistent (URLs can be different, and env specific taxonomy is expected to be different per env, like [d, i, p]) but there can be typos that can slide in.
Flag possible typos and duplicated values between environments.

## What to ignore
Linter issues (ruff), unit tests execution.

## Output format
    ## Summary
        Display a summary of the findings (or lack of), with the number issues found, and in which categories.
        For each category with findings, have a one-liner definition for each issue, prefixed with a severity level, on a separate line.
        Order the findings by severity level, from critical to minor. If no issues were found, state that explicitly.
        Severity levels are: 
        Critical: This is a critical issue that could lead to major problems in the codebase, such as security vulnerabilities, data loss, or system crashes. Immediate attention is required.
        Major: This is a significant issue that could lead to problems in the codebase, such as performance degradation, maintainability issues, or potential bugs. Attention is required, but it may not be as urgent as a critical issue.
        Minor: This is a minor issue that does not significantly impact the codebase, but is worth addressing for consistency or best practices.

    ## Code smell
        This sections is for the code smell issue flagged during review. Skip it if none was found.
        Each issue description starts with the one liner definition, followed by a direct relative link to the file path and line numbers in the repository, then continues with a real description of the finding and why it is an issue.

        ### [File Path & Line Numbers Placeholder]
        Format as: `[path/to/file.py:line_start-line_end](repo_link)`

        Close each findings with a persuasive argument on why it is an issue; describe a concrete future change that this code makes harder or riskier; if you can't name one, don't report the finding. The intensity of the persuasion should be proportional to the severity of the issue (too many bells makes us deaf to all bells).
        
    ## Documentaion Maintenance
        This section is for the discrepancy between the code state and the documentation. Skip if none were found.
        Separate each findings per feature (a feature can have documentation in multiple markdown files, and if it was not updated in one, chances are that it was not updated in others also)
        Start with a one-liner definition, then explain where the documentation is drifting. offer documentation updates to be validated by the reviewer using git diff style blocks.


## Example Generated Output

## Summary
* **3 issues found in Code smell:**
    * [Critical] Hardcoded fallback logic masks proper configuration across deployment environments.
    * [Major] Duplicated validation logic across requests introduces maintenance risk and potential drift.
    * [Minor] Infrastructure code is duplicated inside unrelated business logic functions.
* **1 issue found in documentation_maintenance:**
    * [Minor] Inconsistent docstring styles and outdated configuration keys in documentation markdown files.

## Code smell

### Hardcoded fallback logic masks proper configuration across deployment environments.
**Location:** [`src/config/environments.py:12-15`](#)

The codebase falls back to a hardcoded string `127.0.0.1` when retrieving the `target_url` from environment variables. This approach hides configuration errors in staging or production environments, allowing the app to silently default to an incorrect local routing path instead of crashing immediately to signal a misconfiguration.

Why: If the `target_url` environment variable is not set in a production deployment, the application will attempt to route requests to `127.0.0.1` instead of failing fast to indicate a missing configuration.


### Duplicated validation logic across requests introduces maintenance risk and potential drift.
**Location:** [`src/api/handlers.py:45-62`](#), [`src/api/handlers.py:88-105`](#)

The functions `create_chat_request` and `create_streaming_request` contain identical parameter validation patterns checking model constraints, empty array lengths, and temperature bounds. Keeping these checks duplicated within the file violates DRY principles and means future updates to validation criteria could be applied to one endpoint while accidentally omitting the other.

Why: If the validation rules for `model`, `messages`, or `temperature` change in the future, developers will need to remember to update both functions. This increases the risk of inconsistent behavior and potential bugs if one function is updated while the other is not.


### Infrastructure code is duplicated inside unrelated business logic functions.
**Location:** [`src/api/adapters.py:45-88`](#)

The `UserDataProcessor.process_profiles` and `FinancialReportProcessor.generate_ledger` methods both implement identical timing, error handling, and logging logic. This duplication of infrastructure code within business logic functions increases the risk of inconsistent behavior and makes it harder to maintain or update the logging and error handling strategy across different parts of the application.

Why: If your team decides to switch from printing metrics to sending them to a monitoring platform (like Prometheus or Datadog), or if you want to change the logging prefix from [SYSTEM FAILURE] to [CRITICAL], you must make identical code modifications across every single processor class in your codebase.



## Documentation Maintenance

### Inconsistent docstring styles and outdated configuration keys in documentation markdown files.

The application's API onboarding documentation is drifting away from the actual code changes introduced in this pull request. The codebase has transitioned to standard Google-style docstrings, but inline documentation still leverages the old Sphinx standard. Additionally, docs/architecture/deployment.md still refers to a deprecated configuration block key that has been deleted from the source logic.

Proposed Documentation Updates

# File: docs/architecture/deployment.md
## Configuration Variables
```
| Key | Description | Default |
| :--- | :--- | :--- |
- | target_url_legacy | Deprecated routing variable | None |
+ | target_url | Primary upstream target routing endpoint | None (Throws error if unset) |
```
