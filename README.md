# Amazon Connect: API-only contact + user-defined Segment Attribute round-trip

Goal: create a contact purely via API (no real phone call) in a UAT Connect instance, write the
user-defined Segment Attribute `v_streamArn` with `UpdateContact`, and read it back with
`DescribeContact` (`Contact.SegmentAttributes`). `v_streamArn` is already registered as a
predefined attribute in the instance. Channel and state of the contact don't matter.

SDK: `aws-sdk-go-v2`, `service/connect` v1.203.0 (also checked v1.209.0).

Labels used below:
- **Documented** — stated in official AWS docs (linked).
- **Not documented** — I could not find it in the docs.
- **Inference** — my reasoning, needs to be verified by testing or AWS Support.

## What was tried with CreateContact

IAM and VPC endpoint permissions are confirmed working.

| # | Request | Result |
|---|---------|--------|
| 1 | `Channel=VOICE`, `InitiationMethod=TRANSFER` | `ValidationException: Create contact is only supported for external audio VOICE contacts` |
| 2 | Same, plus `SegmentAttributes {"connect:Subtype": {"ValueString": "connect:ExternalAudio"}}` | `ValidationException: One or more of the input parameters are invalid` |
| 3 | `Channel=TASK`, `InitiationMethod=API` (no other optional fields) | `ValidationException: Contact flow missing` |
| 4 | `Channel=TASK`, `InitiationMethod=API`, `InitiateAs=COMPLETED` | not run yet |

`CreateContactInput` has no `ContactFlowId` / `TaskTemplateId` field.

## Q1. TASK: what does "Contact flow missing" mean?

- **Documented**: for TASK the only supported initiation method is `API`; contacts created this way
  get the sub-type `connect:ExternalTask`. The request has no flow parameter.
  ([CreateContact](https://docs.aws.amazon.com/connect/latest/APIReference/API_CreateContact.html))
- **Not documented**: this error message, and any instance-level setting such as "default flow for
  API-created / external tasks".
- **Inference**: without `InitiateAs`, the service may treat the task as an active contact that must
  be routed, so it looks for a flow.

## Q2. Does `InitiateAs=COMPLETED` avoid the flow requirement?

- **Documented**: only TASK contacts can be initiated as `COMPLETED`; valid values are
  `CONNECTED_TO_USER | COMPLETED`; contact AWS Support before using it for bulk import, otherwise
  requests may be throttled or fail.
  ([CreateContact](https://docs.aws.amazon.com/connect/latest/APIReference/API_CreateContact.html))
- **Not documented**: whether `COMPLETED` skips the flow, and any extra required fields for it.
  Attempt #4 answers this faster than asking.

## Q3. Can UpdateContact set a user-defined Segment Attribute on a COMPLETED contact?

- **Documented**:
  - The page says user-defined contact information can be added or updated for both ongoing and
    completed contacts.
  - `SegmentAttributes`: Contact Expiry and user-defined attributes (String–String) that are defined
    in predefined attributes can be updated with UpdateContact.
    ([UpdateContact](https://docs.aws.amazon.com/connect/latest/APIReference/API_UpdateContact.html))
  - `ValueList` and `ValueArn` are only supported for system-defined attributes, so use
    `ValueString` for `v_streamArn`.
    ([boto3 update_contact](https://docs.aws.amazon.com/boto3/latest/reference/services/connect/client/update_contact.html))
- **Caveat**: the older AWS CLI v1 page still says Contact Expiry is the only segment attribute
  UpdateContact can update. That is probably stale text, but it means user-defined support was added
  later — verify against the pinned SDK version (v1.203.0) by testing.
  ([CLI v1 update-contact](https://docs.aws.amazon.com/goto/aws-cli/connect-2017-08-08/UpdateContact))
- **Not documented**: the exact combination "COMPLETED contact + user-defined segment attribute" is
  only implied by the page-level statement. Whether `DescribeContact` returns it was not verified.

## Q4. StartTaskContact as the fallback

- **Documented**: pass exactly one of `ContactFlowId`, `QuickConnectId`, `TaskTemplateId`; if only
  `TaskTemplateId` is given and the template has no flow, the request fails.
  ([StartTaskContact](https://docs.aws.amazon.com/connect/latest/APIReference/API_StartTaskContact.html))
  A minimal flow that only disconnects + `ContactFlowId` should work.
- **Not verified (inference)**: IAM needs `connect:StartTaskContact` on the instance and the
  contact-flow ARN. Check the Amazon Connect page of the Service Authorization Reference.
- After the flow disconnects, the contact is completed, so writing `v_streamArn` depends on Q3.

## About the two VOICE failures (inference)

- **#2**: docs say VOICE/TRANSFER contacts already *have* the `connect:ExternalAudio` sub-type, and
  CreateContact's `SegmentAttributes` docs only show `connect:ContactExpiry`. Passing
  `connect:Subtype` yourself is the likely cause of "invalid parameters".
- **#1**: "only supported for external audio" most likely means the instance doesn't have the
  Contact Lens external voice integration enabled
  ([contact-lens-integration](https://docs.aws.amazon.com/connect/latest/adminguide/contact-lens-integration.html)).
  The TASK path is more direct.

## Suggested verification order

1. Run attempt #4 (`TASK` + `API` + `InitiateAs=COMPLETED`).
2. `UpdateContact` with `SegmentAttributes {"v_streamArn": {"ValueString": "..."}}`.
3. `DescribeContact` and check `Contact.SegmentAttributes`.
4. If step 1 still fails on the flow, switch to `StartTaskContact` with a disconnect-only flow.

## Results

_To be filled in after testing._
