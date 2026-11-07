# Atividade individual 2: Mensageria

Infraestrutura de Software · ADS · CESAR School · 2026.2  
Lucas Paguetti Pereira

**Caminho escolhido:** A (ElasticMQ local em Docker, sem conta na AWS).  
**Vídeo:** ...

Um produtor publica pedidos de guia de transporte do Rota Vital em uma fila compatível com o Amazon SQS, e os
consumidores retiram, processam e confirmam. Serve para observar que a mensagem só sai da fila com a confirmação,
que vários consumidores dividem o trabalho, que a fila padrão entrega pelo menos uma vez e que uma falha faz a
mensagem voltar depois do visibility timeout.

## Ambiente usado

| Ferramenta | Versão |
| --- | --- |
| macOS (Apple Silicon) | arm64 |
| Docker Engine (Docker Desktop) | 29.7.2 |
| ElasticMQ (`softwaremill/elasticmq-native`) | 1.7.1 |
| AWS CLI | 2.37.0 |
| Python / boto3 | 3.12.14 / 1.43.126 |

## Como rodar

Pré-requisitos: Docker aberto, AWS CLI v2 e Python 3.12 instalados.

**Terminal 1: subir a fila.**

```bash
docker run --rm -p 9324:9324 softwaremill/elasticmq-native
```

**Terminal 2: criar a fila e o ambiente Python.** As credenciais são falsas: o ElasticMQ aceita qualquer valor, mas
a CLI exige que existam.

```bash
cd atividade2-mensageria
export AWS_ACCESS_KEY_ID=x AWS_SECRET_ACCESS_KEY=x AWS_DEFAULT_REGION=us-east-1
aws sqs create-queue --queue-name guias --attributes VisibilityTimeout=15 \
    --endpoint-url http://localhost:9324
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Um consumidor.** Em um terminal rode o consumidor; no Terminal 2, o produtor:

```bash
python consumidor.py
```

```bash
python produtor.py
```

**Dois consumidores.** Pare o consumidor com Ctrl+C, abra dois terminais com `python consumidor.py` (ative o `.venv`
em cada um) e rode o produtor de novo.

**Entre uma rodada e outra,** limpe a fila e espere pelo menos 10 segundos antes de subir os consumidores:

```bash
aws sqs purge-queue --queue-url http://localhost:9324/000000000000/guias \
    --endpoint-url http://localhost:9324
```

São dois motivos, os dois vistos na prática. Sem o `purge`, a mensagem inválida da rodada anterior continua
circulando. Sem a espera, o `receive_message` do consumidor que foi parado continua aberto no servidor por até 10
segundos (o `WaitTimeSeconds`), e a fila pode entregar mensagens a esse consumidor que já não existe. Elas não se
perdem, mas só voltam depois do visibility timeout.

**Opcional: DLQ.**

```bash
aws sqs create-queue --queue-name guias-dlq --endpoint-url http://localhost:9324
aws sqs create-queue --queue-name guias-com-dlq --attributes file://dlq-atributos.json \
    --endpoint-url http://localhost:9324
FILA=guias-com-dlq python consumidor.py
FILA=guias-com-dlq python produtor.py
```

## Código

`consumidor.py` e `produtor.py` são os do enunciado, com uma única mudança: o nome da fila pode vir da variável de
ambiente `FILA`, para rodar o experimento da DLQ. Sem a variável, o nome continua sendo `guias`.

## Resultados

Tudo abaixo vem dos arquivos em [`evidencias/`](evidencias/).

**Um consumidor.** As 20 guias foram geradas e a mensagem inválida reapareceu a cada 15 segundos:

| Horário | `ApproximateReceiveCount` |
| --- | --- |
| 20:00:41 | 1 |
| 20:00:56 | 2 |
| 20:01:11 | 3 |
| 20:01:26 | 4 |

Três segundos depois do envio, a fila mostrava `ApproximateNumberOfMessages: 0` e
`ApproximateNumberOfMessagesNotVisible: 1`: a mensagem inválida continuava na fila, só que invisível.

**Dois consumidores.**

| Rodada | Consumidor A | Consumidor B |
| --- | --- | --- |
| 1 | 10 guias (R-001, R-003, R-005...) | 10 guias (R-002, R-004, R-006...) |
| 2 | 10 guias (ímpares) | 10 guias (pares) |
| 3 | 10 guias (pares) | 10 guias (ímpares) |

**Consumidor que cai.** Matei um consumidor com `kill -9` enquanto o `receive_message` dele estava aberto. A R-001
foi entregue a ele e não processada. As outras 19 saíram às 20:03:07, e a R-001 voltou sozinha às 20:03:22, depois
da R-020. No fim, 20 guias distintas foram geradas.

**DLQ (opcional).** Com `maxReceiveCount` 3, a mensagem inválida falhou três vezes e foi para a `guias-dlq`, com a
origem registrada em `DeadLetterQueueSourceArn`. A fila de origem ficou vazia. Usei visibility timeout de 5 segundos
nessa fila só para encurtar a espera.

## Respostas

**1. Por que o consumidor só apaga a mensagem depois de processar? O que aconteceria se apagasse antes?**

Porque apagar é a confirmação de que o trabalho terminou. Se o consumidor cair no meio, a mensagem continua na fila
e volta depois do visibility timeout, para outro consumidor tentar. Vi isso ao matar um consumidor: a R-001 ficou com
ele, não foi processada e voltou 15 segundos depois, e nenhuma guia se perdeu.

Se apagasse antes, uma falha entre apagar e processar perderia a mensagem para sempre: o hospital teria feito o
pedido, a guia nunca sairia e nada avisaria ninguém.

**2. Com dois consumidores, como as mensagens se dividiram? A ordem foi mantida? O que a documentação do SQS diz sobre ordem nas filas padrão?**

Dividiram-se quase por igual: 10 para cada um nas três rodadas, com um ficando com as ímpares e o outro com as
pares. Isso não é uma garantia, pois em um teste anterior, sem log guardado, deu 9 e 11. Cada consumidor recebeu as
suas mensagens em ordem crescente, e no conjunto a ordem ficou praticamente a de envio.

Mas a ordem não é garantida. No teste do consumidor que caiu, a R-001 foi processada depois da R-020. Além disso, o
ElasticMQ roda em um único processo local, enquanto o SQS real é distribuído. A documentação diz que, nas filas
padrão, "messages may occasionally arrive out of order" e que elas fazem "a best-effort attempt to maintain the
order in which messages are sent". Ou seja, tentam manter a ordem, sem prometer. Quem precisa de ordem usa fila FIFO.

**3. A fila padrão garante entrega pelo menos uma vez, e a mesma mensagem pode chegar duas vezes. O que isso exige do consumidor? Como vocês o tornariam idempotente?**

Exige que processar a mesma mensagem duas vezes tenha o mesmo efeito de processar uma. No Rota Vital isso importa:
uma guia duplicada pode significar dois transportes para a mesma bolsa.

Eu usaria o `requisicaoId` como chave, porque ele identifica o pedido de verdade, e o `MessageId` muda se o produtor
reenviar. A tabela de guias teria `UNIQUE (requisicao_id)` e o consumidor faria
`INSERT ... ON CONFLICT (requisicao_id) DO NOTHING`. Na segunda entrega nada novo é criado, e o consumidor apaga a
mensagem normalmente. Deixaria essa regra no banco, e não em um `SELECT` antes do `INSERT`, porque dois consumidores
poderiam fazer o `SELECT` ao mesmo tempo e os dois achariam que a guia ainda não existe.

**4. Expliquem o visibility timeout com o que viram na mensagem inválida. Para que serve uma DLQ?**

Quando um consumidor recebe uma mensagem, ela não sai da fila: fica invisível para os outros durante o visibility
timeout, 15 segundos aqui. Se não for apagada nesse prazo, volta a ficar visível. A mensagem inválida mostrou isso:
o `processar` lança erro, o `delete_message` nunca roda, e ela voltou às 20:00:41, 20:00:56, 20:01:11 e 20:01:26,
com o `ApproximateReceiveCount` subindo de 1 a 4. Nesse meio-tempo a fila a contava como `NotVisible`.

O problema é que uma mensagem assim nunca vai dar certo e ficaria voltando sem parar. A DLQ é uma fila separada para
onde ela vai depois de um número máximo de tentativas, o `maxReceiveCount`. No meu teste, com 3, ela saiu da fila
principal depois da terceira falha. Assim ela para de gastar processamento e fica guardada para alguém investigar.

**5. No Rota Vital, qual a diferença entre o hospital chamar a emissão de guia diretamente (síncrono) e passar por uma fila (assíncrono)? O que se ganha e o que se perde?**

No síncrono, o hospital chama a emissão e espera: recebe a guia na hora, mas se o serviço estiver fora do ar ou
lento, o pedido falha ou trava junto.

Na fila, o hospital só publica o pedido e segue. Ganha-se desacoplamento (o pedido fica guardado mesmo com o serviço
fora do ar), absorção de picos, escala simples (basta subir mais consumidores, como fiz com dois) e nova tentativa
automática.

Perde-se a resposta imediata: o hospital recebe "pedido recebido", não a guia, e precisa consultar o status ou ser
avisado depois. O sistema também fica mais complexo, com fila, DLQ, monitoramento e consumidor idempotente. E uma
fila única não tem prioridade: no Rota Vital, um pedido de urgência `ALTA` esperaria atrás dos outros, então seria
preciso uma fila por nível de urgência. Para sangue, eu registraria o pedido de forma síncrona, para o hospital ter
a confirmação na hora, e faria a emissão da guia pela fila.

## Evidências

| Arquivo | O que mostra |
| --- | --- |
| [`01-fila-no-ar`](evidencias/01-fila-no-ar.txt) | ElasticMQ no ar e fila criada com visibility timeout de 15 s |
| [`02-um-consumidor`](evidencias/02-um-consumidor.txt) | 20 guias e a mensagem inválida voltando a cada 15 s |
| [`03-dois-consumidores`](evidencias/03-dois-consumidores.txt) | Divisão entre dois consumidores, em três rodadas |
| [`04-consumidor-caiu-mensagem-voltou`](evidencias/04-consumidor-caiu-mensagem-voltou.txt) | Consumidor morto, mensagem preservada e fora de ordem |
| [`05-opcional-dlq`](evidencias/05-opcional-dlq.txt) | Opcional: mensagem inválida indo para a DLQ |

Cada arquivo `.txt` traz o log da execução. O horário no início das linhas das saídas do consumidor foi acrescentado
pelo script de captura, e não pelo `consumidor.py`. O `.png` de mesmo nome é uma imagem gerada a partir desse log,
para leitura rápida; não é uma captura de tela.

## Fontes

- [Filas padrão do SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html)
- [Visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Filas de mensagens mortas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [ElasticMQ](https://github.com/softwaremill/elasticmq)

## Uso de IA

Declaração conforme a PO-CESAR-021.

- **Ferramenta:** Claude Code (modelo Claude Opus 5.5).
- **Finalidade:** instalar as ferramentas, executar os passos no meu computador e coletar os logs; investigar o caso das
  mensagens atrasadas depois de parar um consumidor; montar o experimento opcional da DLQ; gerar as imagens a partir dos
  logs; redigir este README e o rascunho das respostas; preparar o roteiro do vídeo.
- **Como validei:** ...