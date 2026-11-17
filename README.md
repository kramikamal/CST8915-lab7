# CST8915 Lab 7: Introduction to Kubernetes Basics

- **Student Name**: Kamal Krami
- **Student ID**: 041273436
- **Course**: CST8915 Full-stack Cloud-native Development
- **Semester**: Fall 2026

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/S2xGUsxABB0)

## What I Did in This Lab

In this lab, I created an AKS cluster with one node, and I practised creating Kubernetes resources in two ways: with `kubectl` commands and with YAML files.

- **Pod:** I created `my-pod` with `kubectl run`, and then with the file `my-pod.yaml`.
- **Deployment:** I created `my-deployment` with `kubectl create deployment`, and then with `my-deployment.yaml` (3 replicas).
- **Service:** I exposed the Deployment with `kubectl expose`, and then with `my-service.yaml` (type LoadBalancer).
- **Pet Store:** I deployed the full Algonquin Pet Store with one command: `kubectl apply -f algonquin-pet-store-all-in-one.yaml`. This file has eight resources separated by `---`.

I noticed that `kubectl` commands are faster for small tests, but YAML files are better because I can save them in GitHub and use them again.

## My RabbitMQ Test

To understand the RabbitMQ problem, I did a small test:

1. I placed 3 orders in the store.
2. I checked the queue with this command:
   ```bash
   kubectl exec rabbitmq-5d4b559c94-q2v9t -- rabbitmqctl list_queues name durable messages
   ```
   The result was `order_queue  true  3`, so there were 3 messages in the queue.
3. I deleted the RabbitMQ Pod with `kubectl delete pod rabbitmq-5d4b559c94-q2v9t`. Kubernetes created a new Pod with a different name: `rabbitmq-5d4b559c94-kdlqf`.
4. I checked the queue again in the new Pod, and it was **empty**. The 3 orders were lost.
5. I placed 1 new order, and the queue showed `order_queue  true  1`. This means the order-service connected to the new RabbitMQ Pod.

So, the application continued to work, but the old orders did not come back.

## RabbitMQ Analysis

### Is RabbitMQ stateless or stateful?

There is no doubt that RabbitMQ is a **stateful** application. A stateless application, like the product-service, does not keep important data, so Kubernetes can replace it at any time. RabbitMQ is different, because its job is to keep the messages until another service reads them. If RabbitMQ loses its data, the orders are lost.

In this lab, RabbitMQ runs as a **Deployment**, which is normally used for stateless applications.

### What does it mean to run RabbitMQ without persistent storage?

In the YAML file, the RabbitMQ Deployment does not have any volume. This means that RabbitMQ saves its data inside the container. The [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/volumes/) explains that when a container stops, the files that were created inside it are lost.

To be more specific, this means that:

- All the orders that are waiting in the queue can be lost after a crash or a restart.
- Even a **durable** queue is not safe here. My queue was `durable = true`, but the messages were still lost, because the "disk" was inside the Pod.
- The customer sees "order placed", but the order will never be processed, and no error is shown.

### What happens when the RabbitMQ Pod is deleted or restarted?

When I deleted the Pod, the Deployment created a new one right away, because it wants to keep 1 replica. However, the new Pod started empty, without the queue and without the messages. In other words, Kubernetes brought back the Pod, but not the data.

### Potential solutions

I found three possible solutions in my research:

**First, use a StatefulSet with a PersistentVolume.** A [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) is made for applications that need to keep their data. Each Pod gets its own storage, and the data stays even if the Pod is deleted. The Pod also keeps the same name, which is important because RabbitMQ saves its data based on its hostname, as the [RabbitMQ Docker image page](https://hub.docker.com/_/rabbitmq) explains. On AKS, this storage can be an [Azure Disk](https://learn.microsoft.com/azure/aks/concepts-storage).

**Second, use the RabbitMQ Cluster Operator.** This is an [official tool from RabbitMQ](https://www.rabbitmq.com/kubernetes/operator/using-operator.html) that creates RabbitMQ on Kubernetes with persistent storage for us, so we don't need to write everything by hand.

**Third, run more than one RabbitMQ node with quorum queues.** [Quorum queues](https://www.rabbitmq.com/docs/quorum-queues) copy the messages to several RabbitMQ nodes, so if one node has a problem, the messages are still safe on the others.

### Does Azure Service Bus solve the problem?

I strongly believe that Azure Service Bus solves the main problem of this lab. [Azure Service Bus](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview) is a managed message service, so it does not run inside our cluster. If a Pod is deleted in AKS, the messages are not affected, because Azure stores them and takes care of the servers for us.

However, it is not a perfect solution. First, the order-service is written for RabbitMQ, so we would need to change the code to use Service Bus. Second, Service Bus has a cost, and the application would depend on Azure.

Overall, given all these points, it is clear that RabbitMQ needs persistent storage in Kubernetes, or it should be replaced by a managed service like Azure Service Bus.

## Challenges and Learnings

My main challenge was the VM size. Like in Lab 6, the sizes in the instructions were not available for my Azure for Students subscription, so I used `Standard_B2s_v2` in Sweden Central.

Another small challenge was that the `EXTERNAL-IP` of my Services stayed `<pending>` at first. I learned that I need to wait about one minute and run `kubectl get services` again.

The most important thing I learned is that Kubernetes can bring back a Pod very fast, but this does not mean that the data is safe. For applications like RabbitMQ, we need to think about where the data is stored.

## Sources

- [Kubernetes: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes: StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Microsoft Learn: Storage options for applications in AKS](https://learn.microsoft.com/azure/aks/concepts-storage)
- [Docker Hub: RabbitMQ official image](https://hub.docker.com/_/rabbitmq)
- [RabbitMQ: Cluster Kubernetes Operator](https://www.rabbitmq.com/kubernetes/operator/using-operator.html)
- [RabbitMQ: Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [Microsoft Learn: What is Azure Service Bus?](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview)