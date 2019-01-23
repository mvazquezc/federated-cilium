# Federated Cilium

Federated [Cilium](https://cilium.io) is a reference implementation of deploying and managing Cilium across multiple [Kubernetes](https://kubernetes.io) clusters using [Federation V2](https://github.com/kubernetes-sigs/federation-v2).

The purpose of this repository is provide the required resources to deploy [this demo](https://cilium.readthedocs.io/en/stable/gettingstarted/minikube/#step-2-deploy-the-demo-application) in a federated way.

## Minikube Deployment

Minikube will be used to deploy two Kubernetes clusters, follow the [setup guide](https://kubernetes.io/docs/setup/minikube/) using the appropriated VM driver for your environment.

Once `Minikube` is installed, you can proceed with the creation of two clusters. Both clusters must use CNI as network plugin, otherwise Cilium won't work.

```sh
minikube start --profile cluster1 --network-plugin=cni --extra-config=kubelet.network-plugin=cni --memory=5120 --kubernetes-version v1.12.5 --vm-driver <your_driver>
minikube start --profile cluster2 --network-plugin=cni --extra-config=kubelet.network-plugin=cni --memory=5120 --kubernetes-version v1.12.5 --vm-driver <your-driver>
```

**Note**: If using minikube 0.33+ the following parameter must be added to the parameters above `--enable-default-cni`.

## Federation V2 Deployment

Follow the Federation V2 [user guide](https://github.com/kubernetes-sigs/federation-v2/blob/master/docs/userguide.md) for deploying the Federation V2 Control Plane.

The following sections asume cluster1 and cluster2 were used as Federation's host cluster and member cluster respectively.

## Federated Cilium Deployment

The deployment assumes 2 clusters with the context name of `cluster1` and `cluster2`. 

Use the `kubefed2` tool to federate additional Kubernetes resource types required by `Cilium`.

```sh
kubefed2 federate enable ClusterRoleBinding
kubefed2 federate enable RoleBinding
kubefed2 federate enable StatefulSet
kubefed2 federate enable DaemonSet
```

Create the namespace used by Cilium control-plane:

```sh
kubectl --context=cluster1 create ns cilium-system
```

Deploy the standalone etcd and the Cilium Control Plane to both clusters:

```sh
kubectl --context=cluster1 create -f install/standalone-etcd.yaml
kubectl --context=cluster1 create -f install/cilium.yaml
```

Federate the Cilium custom resource types:

```sh
kubefed2 federate enable CiliumNetworkPolicy
```

**Note**: If you get error message `error: Unable to find api resource named...` Cilium pod is not finished creating all the Cilium CRDs. Wait a few seconds and try again.

You should see `Cilium` pod running on both clusters:

```
for i in 1 2
do 
  kubectl --context cluster$i get pods -n cilium-system
done
```

## Star Wars Demo

Create a namespace and deploy the application pieces in there:

```sh
kubectl --context=cluster1 create ns mos-eisley
kubectl --context=cluster1 -n mos-eisley create -f demo/http-sw-app.yaml
```

### Check Cilium Endpoints

```sh
for i in 1 2;
do
  echo "-----Cluster$i-----"
  CILIUM_POD=$(kubectl --context=cluster$i -n cilium-system get pods -l k8s-app=cilium -o name | awk -F '/' '{print $2'})
  kubectl --context=cluster$i -n cilium-system exec $CILIUM_POD -- cilium endpoint list
done
```

### Request Landing from xwing and tiefighter

In this first iteration, both spacecrafts can request landing on deathstar.

```sh
for i in 1 2;
do
  echo "-----Cluster$i-----"
  XWING_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=xwing -o name | awk -F '/' '{print $2'})
  TIEFIGHTER_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=tiefighter -o name | awk -F '/' '{print $2'})
  echo "XWing requesting landing on deathstar..."
  kubectl --context=cluster$i -n mos-eisley exec $XWING_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
  echo "TieFighter requesting landing on deathstar..."
  kubectl --context=cluster$i -n mos-eisley exec $TIEFIGHTER_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
done
```

### Applying L3 policy to avoid rebels landing on deathstar

StarWars wouldn't have had all those movies if rebels landed on Empire's spacecrafts that easy....

This policy will allow spacecrafts labelled as `org=empire` to request landing on deathstar.

Once the policy is applied, we will try to request landing from XWing and TieFighter again.

```sh
kubectl --context=cluster1 -n mos-eisley create -f demo/sw_l3_l4_policy.yaml
for i in 1 2;
do
  echo "-----Cluster$i-----"
  XWING_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=xwing -o name | awk -F '/' '{print $2'})
  TIEFIGHTER_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=tiefighter -o name | awk -F '/' '{print $2'})
  echo "XWing requesting landing on deathstar... (requesting deathstar.mos-eisley.svc.cluster.local/v1/request-landing)"
  kubectl --context=cluster$i -n mos-eisley exec $XWING_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
  echo "TieFighter requesting landing on deathstar... (requesting deathstar.mos-eisley.svc.cluster.local/v1/request-landing)"
  kubectl --context=cluster$i -n mos-eisley exec $TIEFIGHTER_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
done
```

### Applying L7 policy to avoid empire self-destruction

Did I mention deathstar's exhaust port? - Well, what would happen if I press that button?

```sh
for i in 1 2;
do
  echo "-----Cluster$i-----"
  TIEFIGHTER_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=tiefighter -o name | awk -F '/' '{print $2'})
  echo "TieFighter pilot presses the button... (requesting deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port)"
  kubectl --context=cluster$i -n mos-eisley exec $TIEFIGHTER_POD -- curl -sS -m 2 -XPUT deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port
done
```

Oops! Well.. it wasn't that easy in the movies, so let's protect the exhaust-port so no one can interact with the exhaust port, even if they press the button :D

We're going to update our L3 policy and add an L7 policy, so the only allowed endpoint on deathstar is `/v1/request-landing` and will be only accessible from empire's spacecrafts.

```sh
kubectl --context=cluster1 -n mos-eisley apply -f demo/sw_l3_l4_l7_policy.yaml
for i in 1 2;
do
  echo "-----Cluster$i-----"
  XWING_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=xwing -o name | awk -F '/' '{print $2'})
  TIEFIGHTER_POD=$(kubectl --context=cluster$i -n mos-eisley get pods -l class=tiefighter -o name | awk -F '/' '{print $2'})
  echo "XWing requesting landing on deathstar... (requesting deathstar.mos-eisley.svc.cluster.local/v1/request-landing)"
  kubectl --context=cluster$i -n mos-eisley exec $XWING_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
  echo "TieFighter requesting landing on deathstar... (requesting deathstar.mos-eisley.svc.cluster.local/v1/request-landing)"
  kubectl --context=cluster$i -n mos-eisley exec $TIEFIGHTER_POD -- curl -sS -m 2 -XPOST deathstar.mos-eisley.svc.cluster.local/v1/request-landing
  echo "XWing pilot presses the button... (requesting deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port)"
  kubectl --context=cluster$i -n mos-eisley exec $XWING_POD -- curl -sS -m 2 -XPUT deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port
  echo "TieFighter pilot presses the button... (requesting deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port)"
  kubectl --context=cluster$i -n mos-eisley exec $TIEFIGHTER_POD -- curl -sS -m 2 -XPUT deathstar.mos-eisley.svc.cluster.local/v1/exhaust-port
done
```

### Show current policies

```sh
for i in 1 2;
do
  echo "-----Cluster$i-----"
  CILIUM_POD=$(kubectl --context=cluster$i -n cilium-system get pods -l k8s-app=cilium -o name | awk -F '/' '{print $2'})
  kubectl --context=cluster$i -n cilium-system exec $CILIUM_POD -- cilium policy get
done
```

### Clean up

```sh
kubectl delete ns mos-eisley
```
