# Knative Serving v1.22.1 images (linux/amd64)

Pulled via Harbor proxy-cache `harbor.devops-labs.ru/gcr-proxy`, retagged to:

`harbor.idp.ecpk.test/core/knative/<name>:v1.22.1`

## Load & push to Tech Harbor

```bash
for f in *.tar; do docker load -i "$f"; done
for n in activator autoscaler controller webhook queue autoscaler-hpa migrate cleanup net-istio-controller net-istio-webhook; do
  docker push "harbor.idp.ecpk.test/core/knative/${n}:v1.22.1"
done
```

See `MANIFEST.txt` for source digests.
