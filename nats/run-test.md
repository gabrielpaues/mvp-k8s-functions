# Terminal 1 - Start subscriber
kubectl apply -f nats-sub-pod.yaml
kubectl logs -f nats-sub

# Terminal 2 - Publish message
kubectl apply -f nats-pub-pod.yaml
kubectl logs nats-pub

# Cleanup
kubectl delete pod nats-sub nats-pub
