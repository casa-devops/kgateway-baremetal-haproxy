# kgateway em Kubernetes bare-metal atrás de um HAProxy externo

Manifests e configuração do vídeo **"Gateway API em Kubernetes bare-metal: kgateway atrás de um HAProxy externo (caso real)"**.

[kgateway](https://kgateway.dev/docs/envoy/latest/about/overview/) v2.4.3 (Gateway API v1.6.1, canal standard) em um cluster on-premises sem cloud provider. Um par de HAProxy 2.8 com keepalived distribui o tráfego entre os workers, ao lado de um HAProxy Ingress Controller já existente.

Todos os IPs, hostnames e domínios são genéricos: substitua pelos do seu ambiente.

| Valor | Uso |
| --- | --- |
| `192.0.2.10` | VIP do keepalived (lb1/lb2) |
| `192.0.2.11`–`.13` | control plane (`cp1`–`cp3`) |
| `192.0.2.21`–`.23` | workers (`worker1`–`worker3`) |
| `kgatewayhttpcheck.example.corp` | hostname do health check |

## Fluxo de tráfego

```
cliente -> VIP 192.0.2.10:8081 (HAProxy, round robin)
        -> workerN:30081 (Service NodePort, externalTrafficPolicy: Cluster)
        -> Envoy (Gateway http-gateway, listener 8081)
        -> HTTPRoute httpcheck (match pelo header Host)
        -> Service httpcheck:80 -> pods nginx (ConfigMap, /healthz = OK)
```

## Arquivos

| Arquivo | Conteúdo |
| --- | --- |
| `values/kgateway-values.yaml` | Override do chart do controller (`image.tag`) |
| `manifests/kgateway-http-gateway.yaml` | `GatewayParameters` (NodePort 30081, 3 réplicas, um proxy por worker) + `Gateway` `http-gateway` |
| `manifests/kgateway-httpchck.yaml` | App de health check: ConfigMap, Deployment nginx (2 réplicas) e Service `httpcheck` |
| `manifests/http-route.yaml` | `HTTPRoute` `httpcheck` para o hostname dedicado |
| `haproxy/haproxy.cfg` | Configuração do HAProxy externo (idêntica nos dois LBs) |

## Instalação

Os CRDs precisam existir **antes** do controller. São dois conjuntos distintos, e ambos são necessários.

```sh
# 1. CRDs da Gateway API (v1.6.1, canal standard)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

# 2. CRDs do kgateway (mesma versão do controller)
helm upgrade -i kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds \
  --version v2.4.3 -n kgateway-system --create-namespace

# 3. Controller
helm upgrade -i kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.4.3 -n kgateway-system -f values/kgateway-values.yaml

kubectl get gatewayclass   # kgateway  ACCEPTED=True
```

> O chart `kgateway-crds` entrega os CRDs em `templates/`, não em `crds/`: `helm uninstall kgateway-crds` remove os CRDs **e todos os recursos desses tipos**.

Gateway, app de health check e rota:

```sh
kubectl apply -f manifests/kgateway-http-gateway.yaml -f manifests/kgateway-httpchck.yaml -f manifests/http-route.yaml
kubectl -n kgateway-system rollout status deploy/http-gateway
kubectl -n kgateway-system get pods -l gateway-proxy=http-gateway -o wide   # um por worker
curl -H 'Host: kgatewayhttpcheck.example.corp' http://192.0.2.21:30081/healthz   # OK
```

## Pontos de atenção

- **Não existe DaemonSet nem `hostPort`/`hostNetwork`** para o proxy: o kgateway sempre cria um Deployment + Service por Gateway. Aqui se usa NodePort com porta fixa e `topologySpreadConstraints` por hostname para ter um proxy por worker. Um 4º worker **não** ganha um 4º proxy: aumente `replicas`.
- **`nodeTaintsPolicy: Honor` é obrigatório** com control plane com taint. Sem ele, os nós com taint contam como domínios vazios, o pod de surge do rolling update fica em Pending e o rollout trava.
- **`backendRef` sem `namespace` resolve para o namespace da rota.** Mantenha rota e backend no mesmo namespace, ou use um `ReferenceGrant`.
- **O health check usa sintaxe do HAProxy ≥ 2.2** (`http-check send`). A forma 1.8 `option httpchk GET / HTTP/1.1\r\nHost:\ …` é erro fatal no 2.8 e derruba **todos** os frontends no restart.
- Com `externalTrafficPolicy: Cluster`, um check OK significa "o nó alcança um proxy", não "o proxy deste nó está saudável". Use `Local` se precisar dessa precisão.

## Deploy do haproxy.cfg

Valide um arquivo candidato no servidor antes de tocar no arquivo em produção. Comece pelo LB em standby e deixe por último o que detém o VIP.

```sh
scp haproxy/haproxy.cfg lb1:/tmp/haproxy.cfg.candidate
ssh lb1 'haproxy -c -f /tmp/haproxy.cfg.candidate'
ssh lb1 'sudo cp -p /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak-$(date +%Y%m%d%H%M%S) \
  && sudo cp /tmp/haproxy.cfg.candidate /etc/haproxy/haproxy.cfg \
  && sudo haproxy -c -q -f /etc/haproxy/haproxy.cfg \
  && sudo systemctl restart haproxy && systemctl is-active haproxy'
# repita no lb2; os dois devem terminar com o mesmo sha256sum
```
