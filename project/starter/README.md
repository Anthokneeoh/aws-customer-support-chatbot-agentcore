
# Customer Support Chatbot with Amazon Bedrock AgentCore

## Project Overview

This project implements a customer support chatbot for a fictional online shop using the **Amazon Bedrock AgentCore managed harness**.

The application classifies incoming customer messages and routes them through distinct paths for:

* Bug reports
* Platform/FAQ questions
* Other customer support requests

The implementation uses an AgentCore managed harness with a Bedrock Flow for classification and routing. The bug-report path uses a tool exposed through the AgentCore Gateway to persist completed bug reports.

The project also includes an automated flow test suite and a Bedrock Evaluation using an LLM-as-a-judge to assess response correctness.

---

## Architecture and Flow

The Bedrock Flow follows this high-level sequence:

```text
Customer Message
       |
       v
   Flow Input
       |
       v
   Classifier
       |
       v
 Condition Node
   /     |      \
  /      |       \
bug_report support other
  |        |        |
  v        v        v
Bug      FAQ      Other
Report   Prompt   Request
Path     Path     Prompt
  |        |        |
  v        v        v
Bug      Support  Other
Output   Output   Output

```

The classifier is configured to return exactly one of:

* `bug_report`
* `support`
* `other`

The Condition node uses the classifier output to determine the destination path.

---

## Flow Evidence

### Classification and Routing

The classifier prompt instructs the model to classify each customer message into exactly one supported category.

#### Classification Rules

* **`bug_report`**
Used when the customer reports:
* A software bug
* An error
* A failure
* Broken functionality
* Unexpected system behavior


* **`support`**
Used when the customer is asking for:
* Help
* Guidance
* Information about using a product or service


* **`other`**
Used when the message does not fit either of the above categories.

The classifier is explicitly instructed to return only the category value without explanations or additional text. This makes the output suitable for deterministic routing through the Condition node.

#### Classifier Configuration

*Condition Node Expressions*

---

### Bug Report Path

The bug-report path is handled through the AgentCore managed harness.

When a customer reports a software issue, the assistant collects the required information before creating the report:

* Bug description
* Steps to reproduce
* Environment information

Once the required information has been provided, the assistant invokes the bug-report creation tool through the configured AgentCore Gateway.

After successful tool execution, the assistant confirms that the bug report was created and provides the returned ticket ID.

A completed bug report is persisted in the DynamoDB table:

> `bug-report-tool-stack-bug-reports`

#### Bug Report Evidence

The DynamoDB table contains a bug report created through the chatbot:

The conversation evidence demonstrates the collection process and bug-report tool invocation:

---

### Platform Question / FAQ Path

Questions classified as `support` are routed to the FAQ prompt.

The FAQ prompt contains the available online-shop FAQ content and uses the customer's question to produce a relevant answer when the requested information is covered.

For example:

> *How long does delivery take?*

The expected FAQ information explains that estimated delivery times are shown at checkout and in the shipping confirmation email, while processing typically takes 1–2 business days before dispatch.

#### FAQ Prompt Configuration

*Covered FAQ Question*

#### Unsupported FAQ Questions

A support question can be valid customer support but still fall outside the information available in the FAQ.

For example:

> *Can you help me book a flight?*

This is not covered by the available online-shop FAQ. The application therefore directs the customer to human support rather than inventing an answer.

The support contact number is:

`1-800-555-0199`

*Uncovered Question Evidence*

---

### Other Customer Requests

Messages classified as `other` are routed to a separate Other Request path.

For unsupported requests such as:

> *Do you offer cryptocurrency payments?*

the application does not invent a cryptocurrency payment policy. Instead, it directs the customer to human support at:

`1-800-555-0199`

*Other Request Evidence*

---

## Automated Testing

The project includes `flow-tests.json` containing test cases covering all three required paths:

1. Bug report
2. Platform/FAQ question
3. Other request

The test suite was executed using:

```bash
python generate-eval-dataset.py --tests-json flow-tests.json

```

The test run successfully generated:

`output_eval_dataset.jsonl`

The final run reported:

```text
t1_bug_report: wrote eval line
t2_platform_question: wrote eval line
t3_other_request: wrote eval line

Wrote 3 JSONL lines to output_eval_dataset.jsonl (3 harness calls succeeded).

```

The generated JSONL dataset contains the customer prompts, expected/reference responses, and the actual model responses produced by the application.

### Test Artifacts

* **Test Definition:** `flow-tests.json`
The test definition contains at least one test for each required application path.
* **Evaluation Dataset:** `output_eval_dataset.jsonl`
This file contains the generated evaluation records produced from the automated flow tests.

---

## Bedrock Evaluation

The generated JSONL dataset was uploaded to the evaluation S3 bucket and used to create a Amazon Bedrock Evaluation job.

The evaluation uses correctness as the quality metric and evaluates the generated responses against the provided reference/ground-truth expectations.

The final evaluation contained:

```text
3 prompts
Correctness average score: 1.000
Correctness score: 1.000

```

This indicates that all three evaluated responses received a correctness score of 1.

*Evaluation Result*

---

## Observations

1. **Classification must be constrained for reliable routing**
The classifier output is used directly by the Condition node. Because routing depends on the classifier output matching the expected category values, the classifier prompt explicitly restricts the output to:
* `bug_report`
* `support`
* `other`


without explanations, punctuation, or additional text. This makes the classifier output suitable for downstream conditional routing.
2. **Routing behavior depends on classification quality**
The implementation demonstrated that a routing error can cause an otherwise correct response to be sent through the wrong path. The classifier therefore needs clear category definitions and strict output constraints. The final classifier configuration was adjusted so that customer support questions are consistently classified as `support` when they represent requests for help or information.
3. **FAQ behavior is intentionally constrained**
The FAQ path is designed to answer questions using the available FAQ content rather than inventing unsupported policies. When the requested information is not covered, the application directs the customer to human support. This reduces the risk of unsupported or fabricated customer-facing information.
4. **Unsupported requests have an explicit fallback**
Requests outside the supported FAQ scope or application capabilities are not answered with fabricated information. Instead, the application provides the human-support contact number:
`1-800-555-0199`
This provides a defined fallback behavior for unsupported customer requests.
5. **Bug reports require sufficient information before tool execution**
The bug-report path collects the required bug description, reproduction steps, and environment information before invoking the bug-report creation tool. This prevents the application from creating incomplete bug reports when the required information has not yet been supplied.
6. **Automated evaluation provided final validation**
The automated test suite successfully exercised all three required paths. The generated evaluation dataset was subsequently evaluated using Amazon Bedrock Evaluations with correctness as the evaluation metric. The final evaluation produced an average correctness score of 1.000 across 3 prompts, providing evidence that the final responses matched the expected behavior for the tested scenarios.

---

## Evidence Summary

| # | Evidence | Requirement Demonstrated |
| --- | --- | --- |
| **01** | Full flow diagram | Complete classification and routing flow |
| **02** | Classifier prompt configuration | Classifier configuration |
| **03** | Condition node expressions | Conditional routing |
| **04** | Bug-report DynamoDB | Persisted bug report |
| **05** | Bug-report chat transcript | Bug information collection and tool invocation |
| **06** | FAQ prompt node | Embedded FAQ content |
| **07** | Covered question response | Successful FAQ response |
| **08** | Uncovered question response | Unsupported FAQ question fallback |
| **09** | Other request response | Other-request fallback |
| **10** | Bedrock evaluation results | Evaluation correctness score |

---

## Implementation Notes

The project uses the Amazon Bedrock AgentCore managed harness as the application execution layer.

The Bedrock Flow is responsible for classification and routing, while the configured bug-report tool provides persistence for completed bug reports.

The project also separates:

* Application instructions in `system_prompt.txt`
* Flow configuration in Amazon Bedrock
* Test definitions in `flow-tests.json`
* Generated evaluation data in `output_eval_dataset.jsonl`
* Evidence screenshots in `evidence/`

This separation makes the implementation, testing, and evaluation artifacts independently inspectable.

## Final Result

The final implementation provides:

* **Message classification** into three defined categories
* **Conditional routing** through the Bedrock Flow
* **A dedicated bug-report workflow**
* **Bug-report persistence** through the configured tool
* **FAQ-based responses** for supported questions
* **Human-support fallback** for unsupported questions
* **A separate path** for other customer requests
* **Automated tests** covering all required paths
* **A generated Bedrock Evaluation dataset**
* **Bedrock Evaluation** using correctness scoring
* **Final correctness score of 1.000** across 3 prompts
* **Evidence screenshots** covering the required implementation and evaluation criteria