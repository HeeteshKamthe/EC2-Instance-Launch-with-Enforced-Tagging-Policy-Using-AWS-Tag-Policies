# EC2 Instance Launch with Enforced Tagging Policy

**Project:** EC2 Instance Launch with Enforced Tagging (intern exercise)

**Purpose:** Teach interns how to require and verify tags on EC2 instances at creation time using AWS Tag Policies, Service Control Policies (SCPs), and fallback IAM/Config approaches.

---

## Overview / Objective

This hands-on project helps an intern:

* Understand tagging best practices.
* Create and attach a Tag Policy in AWS Organizations that defines required tags and (optionally) prevents noncompliant operations.
* Demonstrate how to launch EC2 instances with tags (Console & CLI).
* Show that creating instances without required tags fails when enforcement is active.

---

## Required tags (project standard)

The EC2 instance **must** have these tags at launch:

| Tag Key | Tag Value (example)                                     |
| ------- | ------------------------------------------------------- |
| Name    | Your Name (e.g., Rahul)                                 |
| emailID | [Your.Email@company.com](mailto:Your.Email@company.com) |
| phoneNo | +91-XXXXXXXXXX                                          |
| Place   | Pune                                                    |

> The examples above are for the intern to replace with their own details.


---

## Notes on Enforcement (short)

* **Tag policies** in AWS Organizations let you standardize tag keys/values and *can* be configured to prevent noncompliant tag operations for specified resource types.
* To make tags *mandatory at creation time*, organizations commonly combine **Tag Policies** (for standardization & validation) with an **SCP** that denies `ec2:RunInstances` unless required tags are present on the request.
* If you do not have Organizations or want a per-account solution, use **AWS Config rules + Lambda** to detect missing tags and remediate (or block via automation), or use IAM condition keys where possible.

---

## Tag Policy (example)

> This Tag Policy is created in the AWS Organizations management account and attached to the OU/account. It defines the allowed keys and value constraints. Note: tag policy evaluates tags and — if enforcement is turned on for resource types — can prevent noncompliant tag operations.

```json
{
  "tags": {
    "Name": {
      "enforced_for": ["ec2:instance"],
      "value": {
        "type": "string",
        "maxLength": 128
      }
    },
    "emailID": {
      "enforced_for": ["ec2:instance"],
      "value": {
        "type": "string",
        "pattern": "^.+@.+\\..+$"
      }
    },
    "phoneNo": {
      "enforced_for": ["ec2:instance"],
      "value": {
        "type": "string",
        "pattern": "^\\+?\\d{7,15}$"
      }
    },
    "Place": {
      "enforced_for": ["ec2:instance"],
      "value": {
        "type": "string"
      }
    }
  }
}
```

> Replace the `enforced_for` resource types with the correct service resource identifiers if needed (e.g., `ec2:instance` or `ec2:instance/*` depending on how your organization expects it). Always validate the policy in the Organizations console before enforcing.

---

## Service Control Policy (SCP) Example — enforce presence of tags at launch

> SCPs are attached to the root/OU/account in Organizations. The SCP below denies `RunInstances` unless the request includes the required tags. This is a common guardrail to *prevent* untagged resource creation.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRunInstancesWithoutRequiredTags",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotEquals": {
          "aws:RequestTag/Name": "",
          "aws:RequestTag/emailID": "",
          "aws:RequestTag/phoneNo": "",
          "aws:RequestTag/Place": ""
        },
        "Null": {
          "aws:RequestTag/Name": "true",
          "aws:RequestTag/emailID": "true",
          "aws:RequestTag/phoneNo": "true",
          "aws:RequestTag/Place": "true"
        }
      }
    }
  ]
}
```

**Important:** The condition logic in SCPs can be tricky — above is a template showing intent. In production you should craft conditions carefully (e.g., `StringEquals`/`Null` combos) and test in a sandbox OU before broad deployment.

---

## Alternative: IAM policy for specific users (account-level)

If you cannot use Organizations / SCPs, you can attach an IAM policy to a role/user that denies `ec2:RunInstances` unless the request contains the required request tags.

Example IAM policy snippet for a role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRunInstancesIfMissingTags",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Name": "true",
          "aws:RequestTag/emailID": "true",
          "aws:RequestTag/phoneNo": "true",
          "aws:RequestTag/Place": "true"
        }
      }
    }
  ]
}
```

Attach this to the IAM principals that will launch instances. This is effective for IAM-controlled users/roles but does not prevent root or other principals without the policy attached.

---

## Console: Step-by-step — Create & Enforce Tag Policy (high level)

1. Sign in to the AWS Organizations **management account**.
2. In the left navigation choose **Policies** → **Tag policies**.
3. Click **Create policy** and paste the Tag Policy JSON (example above).
4. Save the policy and then **Attach** it to the OU or Account you want to govern.
5. OPTIONAL: Toggle enforcement for the resource types you care about (e.g., EC2 instance). AWS Organizations provides a UI to select enforcement; test in a small OU or single account first.
6. To make tags mandatory at creation time, create an SCP (see SCP example) and attach it to the OU/account.

---

## Console: Step-by-step — Launch EC2 with tags (and expected behavior)

### Launch with required tags (expected: SUCCESS)

1. Sign in to the target AWS account console (or switch to role with launch permissions).
2. Open EC2 → Instances → Launch instances.
3. Choose AMI, instance type, etc.
4. **Step: Configure instance details** → Under *Advanced details* or *Tags* step (depending on console flow) add the required tags: `Name`, `emailID`, `phoneNo`, `Place`.
5. Finish launch. If SCP/enforcement is configured correctly, the instance should create successfully.

**Screenshot placeholder:** `assets/success-with-tags.png`
*Caption:* "Instance launch succeeded because all required tags were present."

### Launch without required tags (expected: FAIL when enforced)

1. Repeat the launch flow but omit one or more required tags.
2. If enforcement is active (SCP + Tag Policy enforcement), the console should display an error or denial message preventing the operation. Capture the message.

**Screenshot placeholder:** `assets/failure-missing-tags.png`
*Caption:* "Instance launch blocked — required tags missing."

**Example expected error (console/CLI):** `You are not authorized to perform this operation. The request must include the tag 'Name'` — actual wording may vary. If using SCP, the denial is surfaced as an access denied error for `ec2:RunInstances`.

---

## AWS CLI: Launching an EC2 instance WITH tags (example)

Run this from a shell configured for the target account/role:

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value="Rahul"},{Key=emailID,Value="rahul@example.com"},{Key=phoneNo,Value="+919999999999"},{Key=Place,Value="Pune"}]' \
  --key-name my-keypair
```

If the SCP/Tag Policy denies creation due to missing tags, this command will return an `AccessDenied` style error when tags are omitted.

---

## AWS CLI: Attempting to launch WITHOUT tags (example)

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --count 1 \
  --key-name my-keypair
```

**Expected result when enforcement active:** Command fails with an error; capture its output and include it in the screenshots / report.

---

## Testing & Validation Checklist (intern should complete)

* [ ] Create tag policy in a sandbox OU/account (do not start in root production OU).
* [ ] Attach tag policy and check compliance view (Tag Editor / Resource Groups) for noncompliant resources.
* [ ] Create SCP in sandbox OU to deny `ec2:RunInstances` when `aws:RequestTag/<key>` is null or empty.
* [ ] Test launching EC2 with all tags — should succeed.
* [ ] Test launching EC2 without tags — should fail; capture CLI/console outputs.
* [ ] Document any discovered message text and suggested remediation steps.

---

## Troubleshooting tips

* If the console still allows untagged creation, check:

  * Is the SCP attached to the account/OU? SCPs are only effective if attached to the OU/account where the principal runs.
  * Is `aws:RequestTag/<key>` the correct condition key used in the request? Some tooling or older SDKs may not pass request tags in the same way.
  * Are you testing as an IAM user/role that is governed by the SCP? Root credentials are not blocked by IAM policies but are still subject to SCPs attached to the account.
* Use CloudTrail to see the exact API call and request context; this helps debug missing tag keys in the request.

---

## Example Screenshots (mock images)

Place actual screenshots in an `assets/` folder and update references below before final submission.

* `img/tag-policy-create.png` — "Tag policy creation in AWS Organizations (management account)."
* `img/tag-policy-attach.png` — "Attaching the tag policy to OU/account."
* `img/success-with-tags.png` — "EC2 launch succeeded with required tags present."
* `img/failure-missing-tags.png` — "EC2 launch blocked when required tags are missing — console error shown."

---

## Optional: Automated remediation & detection

* Use **AWS Config** managed rule `required-tags` to detect resources missing tags, and trigger a Lambda to automatically notify or tag resources.
* For strict prevention at creation time, prefer SCP + Tag Policy approach.

---

## Appendix — Helpful CLI commands

List tag-editor compliance in an account (example using resourcegroupstaggingapi):

```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=Name,Values=Rahul
```

Check CloudTrail events for RunInstances calls (example):

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances
```

---

## Final notes for the intern

1. Always test policies in an isolated sandbox OU first.
2. Use clear tagging standards and document them for the team.
3. Remember there are multiple enforcement mechanisms; choose the combination that fits your org (Tag Policy + SCP is the strongest for org-wide enforcement).

---

*Prepared by: Intern exercise skeleton — update this file with screenshots, real ARNs, and exact outputs before handing in.*
