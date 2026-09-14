# Best practices: backend standalone cloud-native

Documento normativo para serviços backend independentes, com um único deployável, executados em ambiente cloud-native. A espinha dorsal é o [The Twelve-Factor App](https://12factor.net/). As seções finais complementam o manifesto com práticas que o 12-Factor não cobre com o detalhe exigido hoje: probes, observabilidade, secrets, containers, resiliência e segurança mínima do serviço.

## Contexto e quando aplicar

Um **backend standalone** é um serviço com fronteira de deploy própria: um repositório (ou um artefato) gera uma release, essa release sobe como um conjunto de processos idênticos e o estado durável fica fora do processo. Exemplos: API HTTP/gRPC, worker de fila, consumidor de eventos.

**Cloud-native**, neste documento, significa que o serviço foi desenhado para ser empacotado em container, orquestrado (Kubernetes, Cloud Run, ECS ou equivalente), escalado horizontalmente e observado por sinais externos (logs, métricas, traces, probes).

Aplique este guia quando:

- o serviço tem ciclo de vida próprio (build, release e rollback independentes de outros sistemas);
- a unidade de escala é o processo ou a réplica, não a thread dentro de um monólito compartilhado;
- config, secrets e backing services mudam entre ambientes sem recompilar o código.

Não use este guia como padrão de frontend, de BFF acoplado a um monólito compartilhado, nem de plataforma interna específica. Bibliotecas internas e módulos compartilhados entram só como dependências versionadas, não como cópia de código entre apps.

O 12-Factor continua sendo a base porque força o serviço a ser **declarativo, stateless, configurável por ambiente e descartável**. Sem isso, probes, autoscaling e deploys contínuos viram contorno operacional.

Cada fator abaixo segue o mesmo formato: princípio, prática no backend standalone, o que fazer, o que evitar e, quando ajuda, um exemplo curto.

## Os 12 fatores

### I. Codebase

**Princípio.** Uma codebase rastreada em controle de versão, muitos deploys.

**Prática.** Um app standalone tem uma codebase que produz um único tipo de deployável. Vários ambientes (dev, staging, prod) são deploys da mesma linha de código, em releases diferentes. Código compartilhado entre serviços vira pacote versionado, não pasta copiada.

**Faça**

- Um repositório (ou um módulo publicável) por app deployável.
- Tag ou SHA imutável em toda release.
- Dependência de código compartilhado via registro interno ou módulo versionado.

**Evite**

- Vários apps sem relação no mesmo repositório sem fronteira de build clara.
- Copiar bibliotecas entre repositórios.
- Branches de longa duração que viram “a versão de produção”.

### II. Dependencies

**Princípio.** Declare e isole as dependências.

**Prática.** Tudo que o processo precisa para subir está no manifesto e no lockfile. A imagem ou o runtime não assume pacotes “já instalados” no host. O CI instala a partir do lockfile, não do resolver solto.

**Faça**

- Manifesto explícito (`package.json`, `go.mod`, `pom.xml`, `requirements.txt`, `Cargo.toml`).
- Lockfile commitado e usado no CI (`package-lock.json`, `go.sum`, `poetry.lock`).
- Dependências de sistema declaradas na imagem (multi-stage), não no nó do cluster.

**Evite**

- `apt-get` / pacotes globais como pré-requisito não documentado do host.
- Instalar latest sem pin no pipeline de produção.
- Vendoring informal fora do gerenciador da linguagem.

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY app.jar app.jar
USER 65532
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### III. Config

**Princípio.** Armazene a config no ambiente.

**Prática.** Tudo que muda entre deploys (URLs, feature flags, limites, credenciais) fica fora do artefato. O binário é o mesmo em staging e produção. Secrets não são “config de arquivo no repo”; entram por variável de ambiente ou volume injetado em runtime.

**Faça**

- Config via env ou arquivo montado no container, gerado pelo orquestrador.
- Nomes estáveis (`DATABASE_URL`, `PORT`, `LOG_LEVEL`).
- Validação de config na subida: falhe rápido se obrigatório faltar.

**Evite**

- `application-prod.yml` com senha commitado.
- Build args com secret gravados na imagem.
- Defaults de produção hardcoded no código.

```env
PORT=8080
LOG_LEVEL=info
DATABASE_URL=postgres://user:pass@db:5432/app
REDIS_URL=redis://cache:6379/0
```

### IV. Backing services

**Princípio.** Trate backing services como recursos anexados.

**Prática.** Banco, cache, fila, SMTP, object storage e APIs de terceiros são recursos endereçáveis. O código fala com eles por URL e credencial. Trocar o provedor de fila ou o host do banco é mudança de config, não de release de código.

**Faça**

- Acesso só por interface e connection string.
- Timeouts e credenciais por recurso, não globais sem dono.
- Degradação explícita quando um recurso anexado não é crítico (cache) versus crítico (banco primário).

**Evite**

- Caminho local de disco como “banco” em produção.
- Hostname de infra hardcoded.
- Tratar um sidecar obrigatório como se fosse memória do processo.

### V. Build, release, run

**Princípio.** Separe rigorosamente build e run.

**Prática.** **Build** produz um artefato imutável (imagem, binário, JAR). **Release** é esse artefato mais a config daquele ambiente. **Run** apenas inicia processos da release. Produção não instala dependências, não compila e não “dá um jeito” no container.

**Faça**

- Pipeline: test → build da imagem com digest → deploy daquele digest.
- Rollback = apontar o runtime para uma release anterior, sem rebuild.
- Identidade da release visível (`GIT_SHA`, `IMAGE_DIGEST`).

**Evite**

- `npm install` ou `go build` no entrypoint de produção.
- Mutar a imagem depois do push.
- Hot-patch em pod/container como procedimento normal.

### VI. Processes

**Princípio.** Execute o app como um ou mais processos stateless.

**Prática.** Qualquer réplica deve poder morrer sem perder dado de negócio. Sessão, upload e cache de escrita ficam em backing services. Disco local é scratch: some no restart.

**Faça**

- Estado de sessão em cache externo ou token autossuficiente.
- Uploads diretos para object storage.
- Locks e sequências no banco ou no coordenador, não em memória de uma réplica.

**Evite**

- Sticky session como requisito de correção.
- Escrever arquivo local e esperar que a próxima request acerte o mesmo pod.
- Cron in-process que só “funciona” com réplica única.

### VII. Port binding

**Princípio.** Exporte serviços via port binding.

**Prática.** O processo escuta uma porta e fala o protocolo (HTTP, gRPC, AMQP se for o caso). O roteamento (Ingress, Service, load balancer) aponta para essa porta. O app não espera que um Apache/Nginx no host “hospede” a aplicação.

**Faça**

- Bind em `0.0.0.0` e porta vinda de `PORT`.
- Um processo, uma porta de serviço. Métricas podem ir na mesma porta em path dedicado ou em porta auxiliar documentada.
- Health na mesma superfície HTTP que o orquestrador alcança.

**Evite**

- Porta fixa sem override por env.
- Depender de um servidor web do host para o app subir.
- Unix socket local como único modo em ambiente container.

```env
PORT=8080
```

### VIII. Concurrency

**Princípio.** Escale via o modelo de processos.

**Prática.** A unidade de escala é a réplica. CPU e memória por processo são limites explícitos. Trabalho pesado (CPU, I/O longo, fan-out) vai para worker com a mesma release, consumindo a mesma família de backing services.

**Faça**

- HPA ou equivalente por CPU, memória ou métrica de negócio (lag da fila).
- Separar API e worker quando o perfil de carga for distinto.
- Limitar concorrência interna (pool de conexões, semáforo de I/O) para caber no cgroup.

**Evite**

- Vertical scaling como único plano.
- Fila em memória compartilhada entre requests como substituto de broker.
- Um único processo fazendo API síncrona e batch pesado sem isolamento.

### IX. Disposability

**Princípio.** Maximize a robustez com startup rápido e shutdown gracioso.

**Prática.** O processo sobe em segundos, aceita tráfego só quando ready e, ao receber `SIGTERM`, para de aceitar work novo, drena requests e fecha conexões. Jobs de fila devem ser interrompíveis ou idempotentes: a mensagem volta ao broker se o ack não ocorreu.

**Faça**

- Handler de `SIGTERM` / `SIGINT` com timeout de drain menor que `terminationGracePeriodSeconds`.
- Timeout de request menor que o idle timeout do load balancer.
- Startup enxuto: conexão lazy ou pool pequeno no boot, não “esquentar o mundo”.

**Evite**

- Ignorar sinais e deixar o orquestrador matar no `SIGKILL`.
- Shutdown que espera job de minutos.
- Init que baixa dependências da internet.

### X. Dev/prod parity

**Princípio.** Mantenha dev, staging e produção o mais parecidos possível.

**Prática.** A distância que importa é de **backing services e runtime**, não de “framework leve no laptop”. Use a mesma família de banco, fila e broker. Containers reduzem o gap de SO e de dependências nativas. Migrations são as mesmas artefatos em todos os ambientes.

**Faça**

- Compose ou cluster local com os mesmos tipos de serviço (Postgres, Redis, Kafka, etc.).
- Migrations versionadas aplicadas pela mesma release.
- Feature flags para diferença de comportamento, não forks de código por ambiente.

**Evite**

- SQLite no laptop e Postgres em produção sem disciplina de SQL.
- “Funciona na minha máquina” com pacotes globais.
- Pular staging em mudanças de schema ou de contrato.

### XI. Logs

**Princípio.** Trate logs como streams de eventos.

**Prática.** O processo escreve eventos em stdout/stderr. Não roteia arquivo, não rotaciona, não envia da aplicação para cinco destinos. O coletor da plataforma agrega, indexa e retém. Prefira JSON com campos estáveis e um id de correlação por request ou mensagem.

**Faça**

- Um evento por linha, estruturado.
- `level`, `msg`, `ts`, `service`, `release`, `correlation_id` (ou `trace_id`).
- Erros com tipo e stack só quando houver exceção; evite logar payload sensível.

**Evite**

- Escrever em `/var/log/app.log` dentro do container.
- Biblioteca de shipper obrigatória no processo (salvo exigência da plataforma).
- `console.log` sem estrutura em produção.

```json
{"ts":"2026-09-14T21:00:00.000Z","level":"info","service":"orders-api","release":"sha-a1b2c3d","correlation_id":"00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01","msg":"order_accepted","order_id":"ord_123"}
```

### XII. Admin processes

**Princípio.** Rode tarefas de admin como processos pontuais.

**Prática.** Migration, backfill e reprocessamento usam o **mesmo artefato da release** que está (ou vai entrar) em produção. Dispare por Job de CI, Kubernetes Job ou tarefa one-off do PaaS. Não exponha `/admin/migrate` autenticado de qualquer jeito como rotina.

**Faça**

- `migrate` como comando do mesmo binário/imagem.
- Job com backoff e idempotência.
- Auditoria de quem rodou o one-off e contra qual release.

**Evite**

- SSH no pod para “rodar um script”.
- Endpoint HTTP permanente para DDL.
- Migration escrita só no README, fora do artefato.

## Extensões cloud-native

O 12-Factor não detalha o contrato com o orquestrador nem a telemetria moderna. As práticas abaixo são obrigatórias para um backend standalone operável.

### Health checks

Publique três sinais distintos. O orquestrador decide se reinicia, se envia tráfego ou se ainda espera o boot.

- **Liveness:** o processo está vivo. Sem dependência externa. Falha implica restart.
- **Readiness:** o processo pode receber tráfego. Falha se o backing service crítico estiver inacessível ou o pool estiver saturado.
- **Startup:** cobre boot lento (JVM, warmup). Evita que o liveness mate o pod cedo demais.

**Faça**

- Paths baratos (`/health/live`, `/health/ready`).
- Ready falha fechado quando o banco primário não responde.
- Cache ou provedor não crítico não derruba liveness.

**Evite**

- Um único `/health` que pinga tudo e também serve de liveness.
- Ready que faz query pesada ou chama terceiros com timeout longo.

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  periodSeconds: 5
startupProbe:
  httpGet:
    path: /health/live
    port: 8080
  failureThreshold: 30
  periodSeconds: 2
```

### Observabilidade

Logs sozinhos não operam o serviço. Instrumente os três sinais e propague contexto.

- **Métricas RED** no caminho síncrono: rate, errors, duration por rota ou método RPC.
- **USE** em recurso: utilização, saturação, erros de pool, fila, CPU e memória.
- **Tracing** distribuído com contexto W3C Trace Context ou equivalente; o `correlation_id` do log é o `trace_id` (ou inclui ele).
- Cardinalidade controlada: não crie label com id de usuário ou de pedido.

**Faça**

- Endpoint de métricas scrapeável (`/metrics`) ou push padronizado pela plataforma.
- Span na borda (HTTP/gRPC) e nos clients de backing services.
- Dashboards e alertas em sintoma (latência, erro, saturação), não em “pod restartou uma vez”.

**Evite**

- Métrica nova por cliente ou por SKU sem agregação.
- Trace amostrado em 0% em produção “para não gastar”.
- Log de debug ligado por padrão.

### Secrets

Config não secreta pode ir em ConfigMap ou env. Segredo (senha, token, chave privada) tem ciclo de vida próprio: injeção em runtime, rotação e auditoria.

**Faça**

- Secret Manager, CSI driver ou env injetado pelo orquestrador.
- Rotação sem rebuild da imagem.
- Escopo mínimo: um secret por propósito, readable só pelo workload.

**Evite**

- Secret no git, mesmo no branch privado.
- Secret em layer da imagem ou em `ARG` de build.
- Reuso da mesma credencial em todos os ambientes.

### Containers

A unidade de deploy é a imagem. Ela deve ser reproduzível, pequena e sem privilégio.

**Faça**

- Multi-stage: compile no stage de build, rode no runtime mínimo.
- Pin da imagem base por digest.
- Usuário não-root, filesystem read-only quando o app permitir, um processo por container.
- `USER` numérico e capabilities dropadas no pod.

**Evite**

- `:latest` em produção.
- Root + shell de debug na imagem final como hábito.
- Vários processos supervisionados por um init caseiro sem necessidade (API + sidecar de log + cron no mesmo container).

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

### Resiliência

Falha de dependência é o caso normal. O serviço define timeouts, limites de retry e semântica de escrita.

**Faça**

- Timeout em todo I/O (HTTP client, DB, gRPC). Sem timeout infinito.
- Retry só em erro transitório, com jitter e teto de tentativas.
- Circuit breaker ou bulkhead quando o downstream satura o pool.
- Escritas idempotentes: `Idempotency-Key` ou chave de negócio única.
- Consumo de fila com ack após persistir; poison message vai para DLQ.

**Evite**

- Retry cego em `POST` não idempotente.
- Cascata: um timeout interno maior que o do cliente.
- Esconder erro 5xx de dependência como 200 com body vazio.

### Segurança mínima do serviço

A borda termina TLS. O processo corre com o menor privilégio e a superfície de API é explícita.

**Faça**

- TLS no Ingress / load balancer; mTLS entre serviços se a malha exigir.
- Service account e role só com as permissões do workload.
- Authn/authz em toda rota que não seja probe ou métrica interna.
- Desligar debug, actuator aberto e profiler em produção.

**Evite**

- Porta de admin pública.
- Credencial de cloud com `*` no IAM “para o app funcionar”.
- Stack trace ou connection string em resposta HTTP.

## Checklist rápido

Use na review de um serviço novo ou de um lift-and-shift.

- [ ] Uma codebase, um artefato, muitos deploys; SHA ou digest na release.
- [ ] Dependências no manifesto e no lockfile; CI instala pelo lock.
- [ ] Config e secrets fora da imagem; validação no boot.
- [ ] Backing services só por URL/credencial anexada.
- [ ] Pipeline separado: build imutável → release → run; sem compile em produção.
- [ ] Processo stateless; disco local é scratch.
- [ ] Bind em `PORT`; probes alcançáveis nessa superfície.
- [ ] Escala por réplica; worker separado se o perfil de carga exigir.
- [ ] `SIGTERM` drena requests; start rápido; jobs idempotentes.
- [ ] Mesma família de backing services em dev e prod; migrations na release.
- [ ] Logs em stdout, JSON, com `trace_id` / `correlation_id`.
- [ ] Admin e migrate como Job da mesma imagem, não endpoint permanente.
- [ ] Liveness, readiness e startup com responsabilidades distintas.
- [ ] Métricas RED/USE e tracing com cardinalidade controlada.
- [ ] Secrets rotacionáveis; nunca no git nem na imagem.
- [ ] Imagem mínima, non-root, base pinada, um processo.
- [ ] Timeouts, retry com jitter, circuit breaker, idempotência em escrita.
- [ ] TLS na borda, least privilege, sem debug em produção.

## Referências

- [The Twelve-Factor App](https://12factor.net/)
- [CNCF Cloud Native Definition](https://www.cncf.io/about/who-we-are/)
- [Kubernetes: Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- [Google SRE: Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
