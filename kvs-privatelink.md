# Kinesis Video Streams over AWS PrivateLink (interface VPC endpoint)

Question: can a client inside our VPC (`vpc-0b2dbe5eee7426e00`, us-east-1) call the KVS data-plane
`PutMedia` API through an interface VPC endpoint instead of the SkyPath internet-egress proxy?

Labels used below:
- **Documented** — stated in the official AWS doc linked at the bottom.
- **Inference** — my reasoning, needs to be verified by testing.

## Short answer

Yes. The KVS interface endpoint covers the data plane, including `PutMedia`. But private DNS is
VPC-wide, and this VPC is shared by QA, UAT and prod, so "create it only for UAT" is not possible
in this VPC (see "Impact on our VPC").

## Answers

| Question | Answer | Label |
|----------|--------|-------|
| Is `PutMedia` (data plane) supported, not just control plane? | Yes. Supported: control plane (KVS API), ingestion/retrieval data plane (Media API: `PutMedia`, `GetMedia`), playback data plane (Archived Media API). | Documented |
| Endpoint service name in us-east-1 | `com.amazonaws.us-east-1.kinesisvideo` | Documented |
| Private DNS name | `*.kinesisvideo.us-east-1.amazonaws.com` (wildcard) | Documented |
| us-east-1 available? | Yes, listed in the Region availability table | Documented |
| Does the `GetDataEndpoint` hostname resolve to the endpoint? | The doc does not mention `GetDataEndpoint` by name. The data endpoints it returns (e.g. `s-xxxxxxxx.kinesisvideo.us-east-1.amazonaws.com`) fall under the wildcard private DNS name, so they should resolve to the endpoint's private IPs. Verify with `dig` from inside the VPC. | Inference |
| Is private DNS required? | Yes. Calling the endpoint's own `vpce-...` DNS names returns errors; only the standard KVS hostname with private DNS enabled works. | Documented |
| Excluded APIs | All of KVS WebRTC (signaling, STUN, TURN, media, control plane). Do not route WebRTC traffic through the endpoint. | Documented |
| Extra quota | 500 requests/second per account per VPC endpoint, across ingestion and playback APIs; excess returns `ClientLimitExceededException`. Other KVS quotas still apply. | Documented |
| Prerequisites | VPC `enableDnsHostnames` + `enableDnsSupport`; subnets in each AZ used; SG allowing inbound 443 from clients. | Documented |

## Impact on our VPC

- **Documented**: enabling private DNS attaches a private hosted zone to the whole VPC. Every
  resource in that VPC resolves the KVS hostname to the endpoint. AWS calls this an
  all-or-nothing decision per VPC; splitting apps between endpoint and public path needs separate VPCs.
- **Observed**: `describe-vpc-endpoints` on `vpc-0b2dbe5eee7426e00` shows QA, UAT and prod
  endpoints in the same VPC (`qa-67327-*`, `uat-67327-*`, `prod-datadog-*`, `SkyPath-Internet-Egress-2`).
- **Inference**: an endpoint created "only in the UAT Terraform" would switch KVS traffic for QA
  and prod in this VPC too.

Check before choosing this route:
1. Does anything in this VPC use KVS WebRTC? If yes, it breaks when private DNS is on.
2. Is it acceptable for prod KVS calls in this VPC to move to the endpoint at the same time, and
   does 500 TPS per endpoint cover the whole VPC's volume?

If either is a no, route 2 (SkyPath allows `*.kinesisvideo.us-east-1.amazonaws.com`) has a
smaller blast radius.

**Inference**: Amazon Connect's own media streaming into KVS runs service-side and does not go
through our VPC, so it is not affected. Only our in-VPC callers (Lambda, consumer service) change path.

## Rollback

Modify the endpoint and clear "Enable private DNS name". The KVS hostname resolves to public IPs
again; clients switch after their next DNS lookup. (Documented)

## Verification steps

```bash
# 1. Service exists and private DNS name
aws ec2 describe-vpc-endpoint-services \
  --filters Name=service-name,Values=com.amazonaws.us-east-1.kinesisvideo \
  --region us-east-1 \
  --query 'ServiceDetails[*].{ServiceName:ServiceName,PrivateDnsName:PrivateDnsName,AZs:AvailabilityZones}'

# 2. After creating the endpoint, from inside the VPC:
aws kinesisvideo get-data-endpoint --stream-name <test-stream> --api-name PUT_MEDIA --region us-east-1
dig +short <hostname from step 2>   # should return private IPs from the endpoint ENIs
```

## Source

- [Access Amazon Kinesis Video Streams using AWS PrivateLink](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/vpc-interface-endpoints.html) (checked 2026-10-26)
