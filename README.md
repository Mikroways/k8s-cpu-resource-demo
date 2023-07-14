# Jugando con lo recursos en k8s

Este repositorio nos sirve de prueba para verificar como se comporta un cluster
cuando seteamos recursos y limites a nuestros contenedores.

## Creando el cluster

Usarmos la herramienta [kcli](https://kcli.readthedocs.io/en/latest/) de la
siguiente forma:

```
pip install -r requirements.txt
```

Con la librería instalada, procedemos a crear un cluster de la siguiente forma:

```
kcli create kube generic --paramfile mw-cluster.yaml
```

Esperar a que se instale el cluster y listo!

> Notar que mucho se resuelve desde `.envrc`

## Instalamos las dependencias

Solo corremos:

```
helmfile sync
```

## Agregamos taints a un worker

De esta forma, el nodo que se stresse no tendrá ingerencia en el cluster:

```
kubectl taint node mw-cpu-resources-worker-1 stress=:NoSchedule   
kubectl drain mw-cpu-resources-worker-1 \
  --delete-emptydir-data --ignore-daemonsets  
kubectl uncordon mw-cpu-resources-worker-1
```

## Ejemplos

Veremos varios ejemplos que la idea es analizar el comportamiento mediante:

* El comando systemd-gtop en el nodo a stressar
* Grafana + prometheus

Entonces, procedemos a conectar al nodo en una consola:

```
kcli ssh mw-cpu-resources-worker-1
```

En la vm corremos:

```
systemd-cgtop kubepods.slice/
```

Observamos la salida, pero antes tendremos que lanzar una prueba

### Prueba sin asignar recursos

Iniciamos una prueba de stress:

```
kubectl apply -f ejemplos/stress
```

Y ahora sí podremos observar la salida en la consola de `systemd-gtop`. Mientras
analizamos el uso de la CPU y verificamos el comportamiento de cómo
se utiliza la CPU del host entre los varios contenedores:

```
kubectl scale --replicas 4 deploy stress-no-respources
```

Vemos el dashboard de [USE cluster](https://mw-cpu-resources-ctlplane-0/grafana/d/df932f61-6ea5-4299-af64-1d9e5497349c/node-exporter-use-method-cluster?orgId=1&refresh=30s).
Si escalamos aun más, veremos la saturación dispararse.

Vemos como están los cgroups:

```
cd  /sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice
```

Analizamos acá como está el `cpu.weight` de cada pod.

Luego, escalamos de nuevo a 1 replica, y creamos otro contenedor con 500m cpu de
requerimientos y vemos qué sucede:

```
k apply -f ejemplos/stress-req/stress-500m.yaml
```

Continuamos analizando el uso de la CPU, ahora desde la vista del nodo usando
[grafana](https://mw-cpu-resources-ctlplane-0/grafana/d/200ac8fdbfbb74b39aff88118e4d1c2c/kubernetes-compute-resources-node-pods?orgId=1&refresh=10s&var-datasource=default&var-cluster=&var-node=mw-cpu-resources-worker-1&from=now-15m&to=now).

Analicemos el `cpu.weight` del nuevo pod, que ahora, no será un pod de QoS
BestEffort, sino Bursteable:

```
cd /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice
```

### Prueba con request de CPU únicamente

Corremos la prueba con:

```
kubectl apply -f ejemplos/stress-req
```

Y ahora sí podremos observar la salida en la consola de `systemd-gtop`. Mientras
observamos el uso de la CPU, podemos escalar y verificar como se distribuye la
CPU por **peso** según el request realizado. **_No hemos usado límite_**.

```
kubectl delete -f ejemplos/stress-req
```

Acá se pueden escalar los contenedores y ver que como se especifican
requerimientros, no se puede exceder el máximo de CPU del nodo.

### Prueba con request y limites

Ahora veremos el ejemplo con los límites:

```
kubectl delete -f ejemplos/stress-req-limit
```

Acá tenemos que analizar el throttling, que es la cantidad de ciclos de CPU que
se limita el uso por períodos de tiempo de CPU a los procesos.

# Referencias

Estas pruebas se han realizado a partir de las siguientes referencias:

* [CPU and memory management on K8S with cgroups
  v2](https://linuxera.org/cpu-memory-management-kubernetes-cgroupsv2/)
* [CPU requests and limits in
  K8S](https://community.ops.io/danielepolencic/cpu-requests-and-limits-in-kubernetes-ock)
* [Why you should keep using CPU limits on
  K8S](https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61)
* [For the love of god, stop using CPU limits on Kubernetes
  (updated)](https://home.robusta.dev/blog/stop-using-cpu-limits)
* [Layer-by-Layer Cgroup in
  Kubernetes](https://medium.com/geekculture/layer-by-layer-cgroup-in-kubernetes-c4e26bda676c)
