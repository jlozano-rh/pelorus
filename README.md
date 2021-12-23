# Pelorus

Pelorus es un dashboard de Grafana que puede desplegarse fácilmente en un clúster de OpenShift, y proporciona una visión a nivel organizacional de las cuatro métricas críticas del rendimiento en la entrega de software que conducen a una [transformacion basada en métricas](https://cloud.redhat.com/blog/exploring-a-metrics-driven-approach-to-transformation). Estas cuatro métricas son:

* Tiempo de espera de los cambios (cuanto tarda el código en llegar a producción)
* Frecuencia de despliegue (con qué frecuencia se despliega una aplicación en producción)
* Tiempo promedio de recuperación (cuánto tardan los sistemas en recuperarse de fallos en producción)
* Tasa de fallos ante cambios (porcentaje de despliegues que requieren rollback y/o correcciones)

## Instalación

### Despliegue Inicial

Pelorus se instala a través de Helm Charts. En el primer despliegue, es necesario instalar los operadores de los que depende Pelorus. Luego, se instala el operador que contiene el stack del core de Pelorus y finalmente los `exporters` que son los encargados de recoger los datos. Por defecto, las siguientes instrucciones instalan los operadores en un namespace llamado `pelorus`:

```bash
git clone https://github.com/konveyor/pelorus
cd pelorus
oc create namespace pelorus
helm install operators charts/operators --namespace pelorus
helm install pelorus charts/pelorus --namespace pelorus
```

Para mas información acerca de la instalación, consulte el siguiente enlace: [github.com/pelorus/Install.md](https://github.com/konveyor/pelorus/blob/master/docs/Install.md)

## Desinstalación

Desinstalar Pelorus es muy sencillo:

```bash
helm uninstall pelorus --namespace pelorus
helm uninstall operators --namespace pelorus
```

## Configuración

### Configuración del stack de Pelorus

El stack de Pelorus (Prometheus, Grafana, Thanos, etc.) se puede configurar cambiando el archivo `values.yaml` que se le pasa al Chart de Helm como parametro. Las buenas prácticas indican que es recomendable partir desde una copia del archivo que se proporciona junto con en el repositorio Git de Pelorus, y almacenarlo en su propio repositorio de configuración para mantenerlo seguro y actualizado. Por lo cual, es posible hacer cambios de configuración actualizando unicamente el archivo `values.yaml` y aplicando los cambios de esta manera:

```bash
helm upgrade pelorus charts/pelorus --namespace pelorus --values openshift/values.yaml
```

> **NOTA:** tener en cuenta, que algunos exporters declarados en el archivo `values.yaml` pueden requerir que existan ciertos secretos en OpenShift.

### Configuración de Exporters

Un exporter es una aplicación (hecha en Python) que recopila los datos que extrae de varias herramientas y plataformas y los expone para que puedan ser consumidos por los dashboards de Pelorus. Cada exporter se despliega individualmente junto al core de Pelorus.

Los exporters pueden ser desplegados y configurados a través de una lista (o arrays) declarados dentro de la propiedad `exporters.instances` dentro del archivo `values.yaml`. Algunos de los exporters también requieren la creación de secretos cuando se integran con herramientas y plataformas externas. Para revisar como configurar cada exporter, consulte el siguiente enlace: [github.com/pelorus/Configuration.md](https://github.com/konveyor/pelorus/blob/master/docs/Configuration.md)

> **NOTA:** por la configuracion y arquitectura actual del cluster de pre-producción en OpenShift, los Builds de los exporters fallan cuando hacen referencia a la imagen s2i de python-39. Para solventar esta situacion, es necesario modificar el objeto `BuildConfig` reemplazando la parte del sourceStrategy por lo siguiente:

```yaml
strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: pelorus
        name: 'python-39:latest'
```

> **NOTA2:** la imagen debe exportarse tambien de forma local para que pueda ser referenciada luego por OpenShift. Para lograr esto, es necesario ejecutar el siguiente comando:

```bash
oc import-image python-39 --from registry.access.redhat.com/ubi8/python-39 --confirm --reference-policy local -n pelorus
```

### Configuración del Pipeline en Jenkins para Automatizacion

Debido a que pelorus, de fabrica no soporta una estrategia de builds binarios. Por lo cual, se tomo como solución agregar `metadata.labels` que contengan toda la información que precisa Pelorus.

El unico dato que debe cambiar entre un build y otro, es el hash del ultimo commit, para ello desde el pipeline se invoca la funcion `recordHashPelorus()` para acometer con este propósito.

Sin embargo, también será necesario configurar algunos `metadata.labels` para cada aplicación que serán siempre fijos. Tomar como ejemplo el siguiente fragmento de código y modificar el objeto **BuildConfig** ubicado en el archivo `openshift/template.yaml`:

```yaml
metadata:
  labels:
    pelorus/app.name: appname
    peloruscommitsha: abcdefgh
    pelorusgitprotocol: http
    pelorusgithost: customer-git.com
    pelorusgitproject: backend
    pelorusgitapp: appname
```

También será necesario agregar un label más al objeto **Deployment** o **DeploymentConfig** segun sea el caso. Igualmente, ubicado en el archivo `openshift/template.yaml`. Y agregar la siguiente linea:

```yaml
metadata:
  labels:
    # Esta linea le indica a pelorus que lo debe tener en cuenta para las metricas
    pelorus/app.name: appname
```

> **NOTA:** tener en cuenta que los labels `pelorusgit*` seran concatenados por pelorus formando la URL del repositorio git de la aplicación. Para este caso, por ejemplo, al concatenar todos ellos tendremos la URL: `http://git.com/customer/app.git`. Sera necesario añadirlas junto con el label `pelorus/app.name` para que sea reconocida por pelorus correctamente.

## Acceder al Dashboard

Para poder acceder al Dashboard de Pelorus y poder revisar las métricas, debe hacerse a traves de su ruta (URL). Podemos obtener la URL ejecutando el siguiente comando:

```bash
echo -en "\nhttps://$(oc get route grafana-route -n pelorus -o template --template={{.spec.host}})\n\n"
```

Sin embargo, puede que en ocasiones sea necesario revisar las reglas o queries de Prometheus, para ello el procedimiento es el mismo. Es decir, a traves de la ruta de prometheus podremos acceder a su consola web. Por tanto, para obtener la URL podemos ejectuar el siguiente comando:

```bash
echo -en "\nhttps://$(oc get route prometheus-pelorus -n pelorus -o template --template={{.spec.host}})\n\n"
```
