# To use with Kubernetes

Use a `Deployment`

```shell
kubectl apply -f https://raw.githubusercontent.com/evillgenius75/rook-aks/master/perf-testing/manifests/readperf.yaml
deployment.extensions "fio-tester-read" created
service "fiotools-read" created

kubectl apply -f https://raw.githubusercontent.com/evillgenius75/rook-aks/master/perf-testing/manifests/writeperf.yaml
deployment.extensions "fio-tester-write" created
service "fiotools-write" created

kubectl get svc fiotools-read
kubectl get svc fiotools-write
```

Access the Output

`kubectl port-forward service/fiotools-read  8001:8001`
Visit http://localhost:8001

`kubectl port-forward service/fiotools-write  8000:8000`
Visit http://localhost:8000

OR

Visit http://<Service-EXTERNAL_IP>:[8000|8001] as long as the firewall allows `8000 and 8001` to the workers.

> Note, you can change the `ENV` variables to submit a new job. Just provide a new job url to `REMOTEFILES` and update the name of `JOBFILES` to the name of the `.fio` file and provide and optional new `PLOTNAME`

```shell
env:
  - name: REMOTEFILES
    value: "https://gist.githubusercontent.com/wallnerryan/06cb07d3d8bee67af025a60a88da053f/raw/a46d97f30b79c2a2a6b42333e7114d85e84c450f/editablejob.fio"
  - name: JOBFILES
    value: editablejob.fio
  - name: PLOTNAME
    value: editablejob
```
