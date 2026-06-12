# Vault Enterprise on OpenShift HVD — reference Helm values

This repository contains reference `values.yaml` files for the [vault-helm](https://github.com/hashicorp/vault-helm)
chart that accompany the HashiCorp Validated Design (HVD) **Vault Enterprise on
OpenShift**. Each file maps the HVD's recommendations to one concrete deployment
profile. Start from the profile closest to your environment, replace the
placeholder values, and deploy with Helm or your GitOps tool.

## Files

| File | Platform | Topology | Seal / unseal |
| ---- | -------- | -------- | ------------- |
| `values-onprem-rz.yaml` | Self-managed OpenShift (on-premises) | 6-pod, redundancy zones | Shamir (manual unseal) |
| `values-hsm-rz.yaml` | Self-managed OpenShift | 6-pod, redundancy zones | HSM auto-unseal (PKCS#11, Thales Luna example) |
| `values-awskms-rz.yaml` | OpenShift on AWS (ROSA / self-managed) | 6-pod, redundancy zones | AWS KMS auto-unseal |
| `values-awskms-no-rz.yaml` | OpenShift on AWS (ROSA / self-managed) | 5-pod fallback, no redundancy zones | AWS KMS auto-unseal |

## How they differ

- **Seal / unseal.** The HSM file auto-unseals through a
  PKCS#11 HSM and the AWS KMS files through cloud KMS. Without an HSM or cloud KMS,
  `values-onprem-rz.yaml` falls back to Shamir manual unseal.
  We recommend auto-unseal for production so rescheduled Pods
  return to service without operator action.
- **Topology.** The `-rz` files use the recommended 6-pod redundancy-zones
  topology: 3 failure domains, 1 voter and 1 non-voter per zone, with automatic
  voter replacement by Autopilot. `values-awskms-no-rz.yaml` is the fallback —
  5 voting Pods placed 2-2-1 across 3 zones — for clusters that cannot run
  redundancy zones.
- **Platform.** The AWS files use IAM Roles for Service Accounts (IRSA) so the
  Vault Pods reach AWS KMS without static credentials. The HSM file adds an init
  container that delivers the vendor PKCS#11 library into the Vault Pod, plus the
  HSM client configuration and credentials.

Everything else (resources, storage, TLS, health probes, telemetry, and the
Service and Route topology) is shared across the files. These files deploy the
Vault server only and disable the Agent Injector and CSI provider.

## Adapting to other clouds

`values-awskms-rz.yaml` and `values-awskms-no-rz.yaml` are not AWS-only. The same
cloud KMS auto-unseal and workload identity pattern applies to other managed
OpenShift platforms, such as Azure Red Hat OpenShift (ARO) and OpenShift Dedicated
on GCP. Retarget a file by adapting these settings to your platform, following the
Vault and cloud provider documentation for the exact configuration:

- **Cloud KMS seal.** Replace the AWS KMS seal stanza with your provider's cloud
  KMS seal, such as Azure Key Vault or GCP Cloud KMS, and supply the seal
  configuration that the provider requires.
- **Workload identity.** Replace IRSA with your provider's workload identity
  mechanism, such as Microsoft Entra Workload ID on Azure or Workload Identity
  Federation on GCP, and bind it to the Vault ServiceAccount.
- **Service exposure.** Load balancer behavior is platform-specific, so the AWS
  annotations do not apply elsewhere. Configure the LoadBalancer Service for your
  load balancer stack, and restrict its endpoints to the clients and peer clusters
  that require them.
- **Storage.** Use a low-latency block StorageClass appropriate for the target
  cloud.

We recommend granting the Vault Pods access to cloud APIs such as KMS through workload
identity, so the Pods mount no static credentials.

## Requirements

The reference files have the following requirements:

- OpenShift 4.22+ (Kubernetes 1.35+) and Vault Helm chart v0.33.0+ for the
  redundancy-zones files (`-rz`).
- A Vault Enterprise license.
- The per-profile Secrets and ConfigMaps listed in each file's header comment.

If your platform does not meet these version requirements yet, use
`values-awskms-no-rz.yaml` and migrate later.

## Usage

These files are references, not turnkey configurations. Replace every
`<placeholder>`, the example hostname, and the namespace before deploying, and
update the pinned image digest to your target Vault Enterprise version. The
files assume dedicated Vault nodes; inline comments show the shared-node
adjustments.

For the design rationale behind every setting, refer to the Vault
Enterprise on OpenShift HVD.
