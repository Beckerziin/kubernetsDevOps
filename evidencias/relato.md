# Relato do grupo

Integrantes: João, Ian Siveris
Data: 15/10/2026
Distribuição Linux: não se aplica (Windows 11 Home, PowerShell nativo)
Python / Docker / Minikube: Python 3.12.10 / Docker 29.8.2 (Docker Desktop 4.95.0, WSL 2) / Minikube v1.39.0 (Kubernetes v1.37.0, kubectl v1.37.0, containerd 2.3.4)

## Etapas

| Etapa | Comando e evidência | Resultado esperado | Resultado observado e explicação |
|---|---|---|---|
| Testes | `python -m unittest discover -s tests -v` → `evidencias/testes.txt` | 8 testes OK | 8 testes executados, `OK`. Incluem os casos da versão quebrada (`/healthz` 200, `/readyz` e `/api/status` 503), que comprovam o comportamento simulado. Não dependem do cluster. |
| Publicação v1 | `docker build ... -t entrega-local:v1`, `minikube image load` (`evidencias/image-load.txt`), `apply` de namespace/ConfigMap/Service, `render.py v1 --replicas 2`, `apply --dry-run=server`, `apply`, `rollout status`, `get pods -o wide` (`evidencias/v1-pods.txt`) | 2 Pods prontos; API v1 | `deployment "entrega" successfully rolled out`. Dois Pods `1/1 Running`, 0 reinícios: `entrega-5c8cd478b5-57spn` (10.244.0.4) e `entrega-5c8cd478b5-fh6zp` (10.244.0.3), ambos no nó `devops-k8s`. O dry-run apenas validou o manifesto na API (`created (server dry run)`); quem publicou foi o `apply` seguinte. |
| Acesso e verificação v1 | `kube.ps1 port-forward service/entrega 8080:80`; `curl.exe /api/status` e `/readyz` (`evidencias/v1-api.txt`); `smoke.py --version v1` (`evidencias/v1-status.json`); captura `evidencias/v1-pagina.jpg` | API v1, `ready: true`, `SMOKE OK` | `/api/status` → `{"version": "v1", "message": "Entrega v1 publicada pelo grupo.", "pod": "entrega-5c8cd478b5-57spn", "ready": true}`; `/readyz` → `HTTP 200 {"ready": true}`; smoke → `SMOKE OK`. A página mostra versão v1, a mensagem do ConfigMap e a instância `entrega-5c8cd478b5-57spn`, que é um dos dois Pods criados na publicação v1. O port-forward escolheu esse Pod e manteve a sessão nele. |
| Escala | `render.py v1 --replicas 3`, `apply`, `rollout status`, `get pods` (`evidencias/escala-3-pods.txt`) | 3 Pods prontos | `deployment.apps/entrega configured`; os dois Pods existentes (`57spn`, `fh6zp`) foram mantidos e um terceiro, `entrega-5c8cd478b5-nw7s7`, foi criado. Todos `1/1 Running`. A imagem não mudou, então o hash do ReplicaSet (`5c8cd478b5`) continua o mesmo: só mudou a quantidade desejada. |
| Exclusão e recuperação | `get pods -w` em segundo plano (`evidencias/watch-exclusao.txt`); `delete pod` do primeiro da lista (`evidencias/pod-excluido.txt`); estado final (`evidencias/exclusao-final-pods.txt`) | Nome antigo removido; novo Pod criado | Excluído: `entrega-5c8cd478b5-57spn` (o mesmo que respondeu na etapa 3). O watch mostrou `57spn` → `Terminating` → `Completed` e, no mesmo instante, um Pod novo, `entrega-5c8cd478b5-45p5q`: `Pending` → `ContainerCreating` → `Running 0/1` → `Running 1/1` em cerca de 3 s. Estado final: `45p5q` (10.244.0.6), `fh6zp` (10.244.0.3) e `nw7s7` (10.244.0.5), todos `1/1`. |
| ConfigMap | Edição de `k8s/configmap.yaml` (`APP_MESSAGE: "Entrega configurada pelo grupo."`); `apply -f k8s/configmap.yaml`; `curl.exe /api/status` (`evidencias/config-antes-restart.txt`); `rollout restart`; `rollout status`; `get pods -o wide` (`evidencias/config-pods-apos-restart.txt`); `smoke.py --version v1 --message ...` (`evidencias/config-status.json`); captura `evidencias/config-pagina.jpg` | Mensagem muda após recriar Pods | Depois do `apply`, o objeto já continha a mensagem nova (`configmap/entrega-config configured`), mas a API ainda respondia `"Entrega v1 publicada pelo grupo."` (Pod `fh6zp`, iniciado antes). O `rollout restart` substituiu os 3 Pods um a um (novo ReplicaSet `5c6cd7d677`: `kcs7l`, `nllrc`, `kk72s`), criando a revisão 2 do Deployment, com a imagem ainda `entrega-local:v1`. Após isso a API respondeu `"Entrega configurada pelo grupo."`, versão v1, e o smoke imprimiu `SMOKE OK`. |
| Atualização v2 | | 3 Pods v2 prontos | |
| Incidente | | Novo Running 0/1; healthz 200, readyz 503 | |
| Rollback | | v2 restaurada | |
| Pipeline v3 | | Smoke OK e estado final v3 | |

## Respostas

1. Por que carregar a imagem no Minikube depois de construir no Docker?
   O `docker build` guarda a imagem no armazenamento do Docker do computador. O nó do Minikube é outro ambiente, com seu próprio runtime (containerd) e seu próprio armazenamento de imagens. Antes do `minikube -p devops-k8s image load`, `minikube image ls` não listava `entrega-local`; depois, passou a listar `docker.io/library/entrega-local:v1` (`evidencias/image-load.txt`). Como o Deployment usa `imagePullPolicy: Never`, o kubelet não tenta baixar a imagem de um registro: sem o load, o Pod ficaria em `ErrImageNeverPull`. O load precisa usar o mesmo perfil (`devops-k8s`) do cluster.
2. Qual a função das portas 8000, 80, 30080 e 8080?
   - **8000**: porta em que a aplicação escuta dentro do contêiner (`containerPort: 8000`, nome `http`, no Deployment; `PORT=8000` no Dockerfile).
   - **80**: porta do Service `entrega` dentro do cluster (`port: 80`), que encaminha para `targetPort: http` (8000) dos Pods com `app: entrega`.
   - **30080**: `nodePort` do Service, que publica o Service numa porta do nó. Não foi usado, porque com o driver Docker o nó fica numa rede interna (192.168.49.2).
   - **8080**: porta aberta no computador pelo `port-forward service/entrega 8080:80`; o próprio kubectl mostrou `Forwarding from 127.0.0.1:8080 -> 8000`, ou seja, ele resolve o Service até um Pod e conecta direto na porta do contêiner.
3. Por que o Pod excluído foi substituído?
   O Deployment declara `replicas: 3`, e o ReplicaSet que ele gerencia compara continuamente o estado atual com esse estado desejado (reconciliação). Ao excluir `57spn`, passaram a existir 2 Pods com `app: entrega`; o controlador agiu sobre a diferença e criou outro Pod a partir do mesmo modelo. O Pod novo **não** é o mesmo objeto: tem outro nome (`45p5q`), outro IP (10.244.0.6 em vez de 10.244.0.4), outro UID e começou com idade 0 e 0 reinícios. O Kubernetes não "recupera" o Pod apagado; ele cria uma nova instância equivalente. Isso funciona bem porque a aplicação não guarda estado.
4. Por que três Pods no mesmo nó não protegem contra a perda da máquina?
   Os três Pods aparecem com `NODE devops-k8s`: o cluster do Minikube tem um único nó, que é um contêiner no Docker Desktop deste computador. As réplicas protegem contra a falha de um processo ou de um Pod, mas, se o nó, o Docker Desktop ou o computador parar, os três param juntos e não há outro nó para onde o controlador possa reagendar os Pods. Proteger contra isso exigiria vários nós (de preferência em máquinas ou zonas diferentes) e regras de distribuição dos Pods; e réplicas também não substituem backup.
5. Por que a mensagem não mudou imediatamente após aplicar o ConfigMap?
   O Deployment injeta o ConfigMap como variáveis de ambiente (`envFrom: configMapRef: entrega-config`). Variáveis de ambiente são entregues ao processo apenas quando o contêiner inicia, e a aplicação lê `APP_MESSAGE` nesse momento. Atualizar o objeto ConfigMap muda o dado guardado no cluster, mas não altera o ambiente de processos que já estão rodando, por isso o Pod `fh6zp` continuou com a mensagem antiga. O `rollout restart` adiciona a anotação `kubectl.kubernetes.io/restartedAt` ao modelo dos Pods; isso muda o modelo, cria uma nova revisão e faz o Deployment substituir os Pods, que ao iniciar leem o valor novo. Não foi preciso construir outra imagem porque a mensagem não faz parte da imagem: imagem (código, `v1`) e configuração (ConfigMap) são separadas. A edição ficou salva em `k8s/configmap.yaml`, que a pipeline reaplica.
6. Tag de imagem e revisão do Deployment são a mesma coisa?
7. Que sinais mostram que a versão quebrada está viva, mas não pronta?
8. O que o timeout fez? O que o rollback restaurou e o que não restaurou?
9. Por que alinhar o arquivo gerado após a reversão?
10. Quais verificações a pipeline realiza? Qual é seu limite?

## Divergências e correções

Descreva hipótese, comando de diagnóstico, sinal observado e correção.

- **Etapa 1 – `docker info` sem servidor.** Hipótese: Docker Desktop instalado, mas fechado. Diagnóstico: `docker version` mostrava apenas a seção Client e o erro `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`; `wsl -l -v` não listava a distribuição `docker-desktop`. Correção: abrir o Docker Desktop e aguardar o mecanismo; depois `docker info` respondeu com Server 29.8.2 (OSType linux) e o cluster `devops-k8s` subiu com o nó `Ready` (`evidencias/nodes.txt`).
