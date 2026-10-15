# Relato do grupo

Integrantes: João, Ian Siveris
Distribuição Linux: não se aplica (Windows 11 Home, PowerShell)
Python / Docker / Minikube: Python 3.12.10 / Docker 29.8.2 (Desktop 4.95.0, WSL 2) / Minikube v1.39.0 (Kubernetes v1.37.0)

## Etapas

| Etapa | Comando e evidência | Resultado esperado | Resultado observado e explicação |
|---|---|---|---|
| Testes | `unittest discover` → `testes.txt` | 8 testes OK | 8 testes OK, sem depender do cluster. |
| Publicação v1 | `docker build`, `image load`, `apply`, `rollout status` → `image-load.txt`, `v1-pods.txt` | 2 Pods prontos; API v1 | 2 Pods `1/1 Running` (`57spn`, `fh6zp`). O dry-run só validou; o `apply` publicou. |
| Acesso v1 | `port-forward 8080:80`, `curl`, `smoke.py` → `v1-api.txt`, `v1-status.json`, `v1-pagina.jpg` | API v1, `SMOKE OK` | API `v1`, `ready: true`, `/readyz` 200, `SMOKE OK`. A página mostra o Pod `57spn`. |
| Escala | `render.py --replicas 3`, `apply` → `escala-3-pods.txt` | 3 Pods prontos | Pods antigos mantidos e um novo criado (`nw7s7`). Mesmo ReplicaSet, pois a imagem não mudou. |
| Exclusão e recuperação | `delete pod`, `get pods -w` → `pod-excluido.txt`, `watch-exclusao.txt`, `exclusao-final-pods.txt` | Nome antigo removido; novo Pod criado | `57spn` foi excluído e `45p5q` criado em ~3 s. Voltou a 3 Pods prontos. |
| ConfigMap | edição do `configmap.yaml`, `apply`, `rollout restart` → `config-antes-restart.txt`, `config-status.json`, `config-pagina.jpg` | Mensagem muda após recriar Pods | Após o `apply`, a mensagem continuou antiga. Após o restart (revisão 2, imagem v1), mostrou "Entrega configurada pelo grupo." com `SMOKE OK`. |
| Atualização v2 | | 3 Pods v2 prontos | |
| Incidente | | Novo Running 0/1; healthz 200, readyz 503 | |
| Rollback | | v2 restaurada | |
| Pipeline v3 | | Smoke OK e estado final v3 | |

## Respostas

1. Por que carregar a imagem no Minikube depois de construir no Docker?
   O nó do Minikube tem armazenamento de imagens próprio, separado do Docker do computador. Com `imagePullPolicy: Never`, sem o `image load` o Pod falharia com `ErrImageNeverPull`.
2. Qual a função das portas 8000, 80, 30080 e 8080?
   8000 é a porta da aplicação no contêiner; 80 é a porta do Service no cluster; 30080 é o NodePort (não usado aqui); 8080 é a porta local aberta pelo port-forward.
3. Por que o Pod excluído foi substituído?
   O ReplicaSet mantém as 3 réplicas declaradas (reconciliação). O Pod novo não é o mesmo: tem outro nome, IP e UID.
4. Por que três Pods no mesmo nó não protegem contra a perda da máquina?
   O cluster tem um único nó. Se ele ou o computador parar, os três Pods param juntos. Seriam necessários vários nós.
5. Por que a mensagem não mudou imediatamente após aplicar o ConfigMap?
   A mensagem entra como variável de ambiente, lida só quando o contêiner inicia. Foi preciso recriar os Pods; a imagem não mudou porque a configuração fica fora dela.
6. Tag de imagem e revisão do Deployment são a mesma coisa?
7. Que sinais mostram que a versão quebrada está viva, mas não pronta?
8. O que o timeout fez? O que o rollback restaurou e o que não restaurou?
9. Por que alinhar o arquivo gerado após a reversão?
10. Quais verificações a pipeline realiza? Qual é seu limite?

## Divergências e correções

- **Etapa 1 – `docker info` sem servidor:** o Docker Desktop estava fechado. Depois de aberto, o servidor respondeu e o nó ficou `Ready` (`nodes.txt`).
