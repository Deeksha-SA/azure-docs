---
title: Customize CoreDNS for Azure Kubernetes Service (AKS)
description: Learn how to customize CoreDNS to add subdomains or extend custom DNS endpoints using Azure Kubernetes Service (AKS)
services: container-service
author: palma21
ms.topic: article
ms.date: 03/15/2019
ms.author: jpalma

#Customer intent: As a cluster operator or developer, I want to learn how to customize the CoreDNS configuration to add sub domains or extend to custom DNS endpoints within my network
---

# Customize CoreDNS with Azure Kubernetes Service

Azure Kubernetes Service (AKS) uses the [CoreDNS][coredns] project for cluster DNS management and resolution with all *1.12.x* and higher clusters. Previously, the kube-dns project was used. This kube-dns project is now deprecated. For more information about CoreDNS customization and Kubernetes, see the [official upstream documentation][corednsk8s].

As AKS is a managed service, you cannot modify the main configuration for CoreDNS (a *CoreFile*). Instead, you use a Kubernetes *ConfigMap* to override the default settings. To see the default AKS CoreDNS ConfigMaps, use the `kubectl get configmaps --namespace=kube-system coredns -o yaml` command.

This article shows you how to use ConfigMaps for basic customization options of CoreDNS in AKS. This approach differs from configuring CoreDNS in other contexts such as using the CoreFile. Verify the version of CoreDNS you are running as the configuration values may change between versions.

> [!NOTE]
> `kube-dns` offered different [customization options][kubednsblog] via a Kubernetes config map. CoreDNS is **not** backwards compatible with kube-dns. Any customizations you previously used must be updated for use with CoreDNS.

## Before you begin

This article assumes that you have an existing AKS cluster. If you need an AKS cluster, see the AKS quickstart [using the Azure CLI][aks-quickstart-cli], [using Azure PowerShell][aks-quickstart-powershell], or [using the Azure portal][aks-quickstart-portal].

When creating a configuration like the examples below, your names in the *data* section must end in either *.server* or *.override*. This naming convention is defined in the default AKS CoreDNS Configmap which you can view using the `kubectl get configmaps --namespace=kube-system coredns -o yaml` command.

## What is supported/unsupported

All built-in CoreDNS plugins are supported. No add-on/third party plugins are supported.

## Rewrite DNS

One scenario you have is to perform on-the-fly DNS name rewrites. In the following example, replace `<domain to be written>` with your own fully qualified domain name. Create a file named `corednsms.yaml` and paste the following example configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  test.server: | # you may select any name here, but it must end with the .server file extension
    <domain to be rewritten>.com:53 {
    log
    errors
    rewrite stop {
      name regex (.*)\.<domain to be rewritten>.com {1}.default.svc.cluster.local
      answer name (.*)\.default\.svc\.cluster\.local {1}.<domain to be rewritten>.com
    }
    forward . /etc/resolv.conf # you can redirect this to a specific DNS server such as 10.0.0.10, but that server must be able to resolve the rewritten domain name
    }
```

> [!IMPORTANT]
> If you redirect to a DNS server, such as the CoreDNS service IP, that DNS server must be able to resolve the rewritten domain name.

Create the ConfigMap using the [kubectl apply configmap][kubectl-apply] command and specify the name of your YAML manifest:

```console
kubectl apply -f corednsms.yaml
```

To verify the customizations have been applied, use the [kubectl get configmaps][kubectl-get] and specify your *coredns-custom* ConfigMap:

```
kubectl get configmaps --namespace=kube-system coredns-custom -o yaml
```

Now force CoreDNS to reload the ConfigMap. The [kubectl delete pod][kubectl delete] command isn't destructive and doesn't cause down time. The `kube-dns` pods are deleted, and the Kubernetes Scheduler then recreates them. These new pods contain the change in TTL value.

```console
kubectl delete pod --namespace kube-system -l k8s-app=kube-dns
```

> [!Note]
> The command above is correct. While we're changing `coredns`, the deployment is under the **kube-dns** label.

## Custom forward server

If you need to specify a forward server for your network traffic, you can create a ConfigMap to customize DNS. In the following example, update the `forward` name and address with the values for your own environment. Create a file named `corednsms.yaml` and paste the following example configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  test.server: | # you may select any name here, but it must end with the .server file extension
    <domain to be rewritten>.com:53 {
        forward foo.com 1.1.1.1
    }
```

As in the previous examples, create the ConfigMap using the [kubectl apply configmap][kubectl-apply] command and specify the name of your YAML manifest. Then, force CoreDNS to reload the ConfigMap using the [kubectl delete pod][kubectl delete] for the Kubernetes Scheduler to recreate them:

```console
kubectl apply -f corednsms.yaml
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns
```

## Use custom domains

You may want to configure custom domains that can only be resolved internally. For example, you may want to resolve the custom domain *puglife.local*, which isn't a valid top-level domain. Without a custom domain ConfigMap, the AKS cluster can't resolve the address.

In the following example, update the custom domain and IP address to direct traffic to with the values for your own environment. Create a file named `corednsms.yaml` and paste the following example configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  puglife.server: | # you may select any name here, but it must end with the .server file extension
    puglife.local:53 {
        errors
        cache 30
        forward . 192.11.0.1  # this is my test/dev DNS server
    }
```

As in the previous examples, create the ConfigMap using the [kubectl apply configmap][kubectl-apply] command and specify the name of your YAML manifest. Then, force CoreDNS to reload the ConfigMap using the [kubectl delete pod][kubectl delete] for the Kubernetes Scheduler to recreate them:

```console
kubectl apply -f corednsms.yaml
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns
```

## Stub domains

CoreDNS can also be used to configure stub domains. In the following example, update the custom domains and IP addresses with the values for your own environment. Create a file named `corednsms.yaml` and paste the following example configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  test.server: | # you may select any name here, but it must end with the .server file extension
    abc.com:53 {
        errors
        cache 30
        forward . 1.2.3.4
    }
    my.cluster.local:53 {
        errors
        cache 30
        forward . 2.3.4.5
    }

```

As in the previous examples, create the ConfigMap using the [kubectl apply configmap][kubectl-apply] command and specify the name of your YAML manifest. Then, force CoreDNS to reload the ConfigMap using the [kubectl delete pod][kubectl delete] for the Kubernetes Scheduler to recreate them:

```console
kubectl apply -f corednsms.yaml
kubectl delete pod --namespace kube-system --selector k8s-app=kube-dns
```

## Hosts plugin

As all built-in plugins are supported this means that the CoreDNS [Hosts][coredns hosts] plugin is available to customize as well:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom # this is the name of the configmap you can overwrite with your changes
  namespace: kube-system
data:
    test.override: | # you may select any name here, but it must end with the .override file extension
          hosts { 
              10.0.0.1 example1.org
              10.0.0.2 example2.org
              10.0.0.3 example3.org
              fallthrough
          }
```

## Troubleshooting

For general CoreDNS troubleshooting steps, such as checking the endpoints or resolution, see [Debugging DNS Resolution][coredns-troubleshooting].

To enable DNS query logging, apply the following configuration in your coredns-custom ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  log.override: | # you may select any name here, but it must end with the .override file extension
        log
```

After you apply the configuration changes, use the `kubectl logs` command to view the CoreDNS debug logging. For example:

```console
kubectl logs --namespace kube-system --selector k8s-app=kube-dns
```

## Next steps

This article showed some example scenarios for CoreDNS customization. For information on the CoreDNS project, see [the CoreDNS upstream project page][coredns].

To learn more about core network concepts, see [Network concepts for applications in AKS][concepts-network].

<!-- LINKS - external -->
[kubednsblog]: https://www.danielstechblog.io/using-custom-dns-server-for-domain-specific-name-resolution-with-azure-kubernetes-service/
[coredns]: https://coredns.io/
[corednsk8s]: https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns
[dnscache]: https://coredns.io/plugins/cache/
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get
[kubectl delete]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete
[coredns hosts]: https://coredns.io/plugins/hosts/
[coredns-troubleshooting]: https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

<!-- LINKS - internal -->
[concepts-network]: concepts-network.md
[aks-quickstart-cli]: ./learn/quick-kubernetes-deploy-cli.md
[aks-quickstart-portal]: ./learn/quick-kubernetes-deploy-portal.md
[aks-quickstart-powershell]: ./learn/quick-kubernetes-deploy-powershell.md
