---
layout: post
title:  "Fix metrics server startup Kubernetes"
date:   2022-09-19 18:00:00 +0200
categories: kubernetes metrics-server
---

My goal was to deploy [metrics-server][metrics-server] in Kubernetes. After deploying it, it was not able to start. The error was
```
E1229 07:09:05.013998 1 summary.go:97] error while getting metrics summary from Kubelet node2(192.168.x.x:10250): Get https://192.168.x.x:10250/metrics/resource/: x509: cannot validate certificate for 192.168.x.x because it doesn't contain any IP SANs
```

It looks pretty clear. The configured certificate does not allow IPs as hostnames.

# How to fix

I found a [fix][fix-url] in Github but it didn't work just following the steps described so I did following steps.

1. Install some tools (Ubuntu)
    ```
    sudo apt install golang-cfssl jq
    ```
2. Generate Certificate Signing Request (CSR) file
    ```
    cat <<EOF | jq -r . | cfssl genkey - | cfssljson -bare kubelet-server
    {
      "hosts": [
        "master1",
        "node2",
        "node3",
        "node4",
        "192.168.30.12",
        "192.168.30.11",
        "192.168.30.15",
        "192.168.30.16"
      ],
      "CN": "system:node:kubelet-server",
      "name": [
        {
          "C": "ES",
          "ST": "Spain",
          "L": "Malaga",
          "O": "system:nodes",
          "OU": "system:nodes"
        }
      ]
    }
    EOF
    ```
3. Create CSR in kubernetes
    ```
    cat <<EOF | kubectl create -f -
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: kubelet-server
    spec:
      request: $(cat kubelet-server.csr | base64 | tr -d '\n')
      signerName: kubernetes.io/kubelet-serving
      groups:
      - system:nodes
      - system:authenticated
      usages:
      - digital signature
      - key encipherment
      - server auth
    EOF
    ```
4. Approve the request generated in kubernetes
    ```
    kubectl certificate approve kubelet-server
    ```
5. Download the generated certificate after approving the CSR in kubernetes
    ```
    kubectl get csr kubelet-server -o jsonpath='{.status.certificate}' | base64 --decode > kubelet-server.pem
    ```
    This command places in ```kubelet-server.pem``` file the public and private key of the certificate. Move the private key to another file ```kubelet-server-key.pem```
6. Copy files ```kubelet-server.pem``` and ```kubelet-server-key.pem``` to all the nodes of the kubernetes cluster (masters and workers) to the path ```/var/lib/kubernetes/pki```
7. Verify that kubelet (in master nodes) uses file ```/etc/kubernetes/kubelet-config.yaml```. To verify it, run below command
    ```
    sudo ps aux | grep kubelet
    ```
    And the output should look like below snippet
    ```
    root 2020195 5.6 0.3 2160136 121488 ? Ssl Sep18 73:58 /opt/bin/kubelet --logtostderr=true --v=2 --node-ip=192.168.30.16 --hostname-override=node4 --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/etc/kubernetes/kubelet-config.yaml --kubeconfig=/etc/kubernetes/kubelet.conf --container-runtime=remote --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --runtime-cgroups=/systemd/system.slice --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --volume-plugin-dir=/var/lib/kubelet/volumeplugins
    ```
    The important flag is ```--config=/etc/kubernetes/kubelet-config.yaml```
8. Configure kubelet to read the public and private key we placed previously in ```/var/lib/kubernetes/pki```
    ```
    echo "tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet-server-key.pem" >> /etc/kubernetes/kubelet-config.yaml
    echo "tlsCertFile: /var/lib/kubelet/pki/kubelet-server.pem" >> /etc/kubernetes/kubelet-config.yaml
    ```
9. Restart kubelet
    ```
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```

Repeat steps 7-9 for all master nodes in the kubernetes cluster.

[metrics-server]: https://github.com/kubernetes-sigs/metrics-server
[fix-url]: https://github.com/kubernetes-sigs/metrics-server/issues/146#issuecomment-459239615