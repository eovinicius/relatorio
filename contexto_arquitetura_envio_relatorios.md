# Contexto de arquitetura --- Envio de relatórios PDF por e-mail

> Documento de transferência de contexto para continuidade da discussão
> em outro assistente/ferramenta (TA).
>
> **Idioma e estilo preferidos:** português brasileiro, comunicação
> direta, didática e objetiva. Usar exemplos em .NET/C# quando fizer
> sentido. Evitar teoria excessiva e não repetir decisões já tomadas sem
> necessidade.

## 1. Objetivo da iniciativa

A plataforma já permite que um especialista consulte um cliente,
gere/obtenha um relatório em PDF e faça o download para depois
compartilhar manualmente por e-mail ou WhatsApp.

A iniciativa é evoluir essa experiência com duas novas possibilidades,
sem remover o download atual:

1.  **Envio manual por e-mail:** o especialista aciona uma ação na
    plataforma e o sistema envia o PDF diretamente para o e-mail do
    cliente.
2.  **Envio automático mensal:** o especialista ativa uma opção/checkbox
    no perfil do cliente para que o sistema gere e envie o relatório por
    e-mail mensalmente.

O download manual existente continua disponível e independente dos dois
novos fluxos.

### Regras funcionais importantes

-   O especialista pode ativar ou desativar a preferência de envio
    mensal do cliente.
-   O envio manual por e-mail pode acontecer mesmo que o cliente **não**
    esteja inscrito no envio automático.
-   O envio manual **não conta** como o envio automático daquele mês.
-   Um envio manual não deve impedir nem bloquear o envio automático
    mensal.
-   O volume estimado é inferior a **1.000 clientes**.
-   O agendamento exato ainda não foi definido.
-   A preferência mensal será persistida em uma tabela no banco já
    existente.
-   A API de negócio existente é proprietária dos dados e das regras de
    negócio.
-   Já existe uma API de e-mail que pode ser consumida por HTTP.
-   Atualmente a geração do PDF está dentro do BFF, mas foi considerado
    viável extraí-la para uma biblioteca .NET compartilhada.
-   A decisão inicial é **não utilizar SQS**, mantendo a solução simples
    para o volume esperado. A decisão pode ser revista se volume,
    duração, necessidade de retentativas ou requisitos operacionais
    crescerem.

## 2. Cenário atual

-   Existe um BFF que recebe a solicitação da plataforma.
-   O BFF consulta uma API existente para obter os dados necessários ao
    relatório.
-   A geração do PDF ocorre atualmente no BFF.
-   O especialista baixa o PDF e o compartilha manualmente com o
    cliente.
-   Existe uma API de e-mail já implementada, que poderá ser chamada
    tanto pelo BFF quanto pelo novo Worker.

## 3. Arquitetura proposta

A proposta é separar responsabilidades e reaproveitar a geração do PDF,
mantendo a API existente como dona dos dados.

### Componentes

  -----------------------------------------------------------------------
  Componente                          Responsabilidade
  ----------------------------------- -----------------------------------
  **BFF existente**                   Atender as requisições da
                                      plataforma para download e envio
                                      manual por e-mail. Também encaminha
                                      a ativação/desativação da
                                      preferência mensal à API.

  **API de negócio existente**        Persistir e consultar preferências,
                                      listar clientes inscritos, fornecer
                                      os dados do relatório e controlar
                                      os registros/status dos envios. É a
                                      dona do banco e das regras de
                                      negócio.

  **Reports.PdfGenerator**            Biblioteca .NET compartilhada para
                                      gerar o PDF a partir dos dados
                                      recebidos. Retorna `byte[]` ou
                                      `Stream`.

  **Reports.Worker**                  Nova aplicação .NET independente,
                                      responsável pelo processamento
                                      mensal dos envios automáticos.

  **EventBridge Scheduler**           Disparar a execução mensal do
                                      Worker.

  **API de e-mail existente**         Enviar e-mails e anexos PDF.
                                      Consumida via HTTP pelo BFF e pelo
                                      Worker.

  **Banco existente**                 Armazenar a preferência mensal e,
                                      se adotado, o controle/histórico
                                      operacional dos envios.
  -----------------------------------------------------------------------

### Visão lógica

``` text
                         ┌─────────────────────┐
                         │    Plataforma       │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      BFF atual      │
                         └──────┬────────┬─────┘
                                │        │
                    consulta    │        │ gera PDF
                    dados       │        ▼
                                │  ┌──────────────────────┐
                                └─▶│ Reports.PdfGenerator │
                                   └──────────────────────┘
                                             │
                              envio manual   │
                                             ▼
                                   ┌──────────────────┐
                                   │   API de e-mail  │
                                   └──────────────────┘


 ┌─────────────────────┐       ┌─────────────────────┐
 │ EventBridge Scheduler│──────▶│    Reports.Worker   │
 └─────────────────────┘       └──────────┬──────────┘
                                          │
                             consulta / reserva / atualiza
                                          │
                                          ▼
                                ┌─────────────────────┐
                                │   API de negócio    │
                                └──────────┬──────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ Banco existente │
                                  └─────────────────┘

 Reports.Worker ───────▶ Reports.PdfGenerator
 Reports.Worker ───────▶ API de e-mail
```

O desenho é conceitual: as chamadas reais do BFF e do Worker para a API
devem ser definidas conforme os contratos existentes.

## 4. Reports.Worker: aplicação nova ou parte da API?

**Decisão/recomendação:** criar o Worker como uma aplicação .NET
separada da API. Ele pode ficar na mesma solution e, dependendo da
organização do código, no mesmo repositório, mas deve ter ciclo de
execução e responsabilidade próprios.

Exemplo de organização:

``` text
Reports.sln
├── Reports.Api                 # API existente ou projeto de API relacionado
├── Reports.Bff                 # BFF existente
├── Reports.Worker              # Nova aplicação de processamento mensal
└── Reports.PdfGenerator        # Biblioteca compartilhada de geração de PDF
```

A API continua atendendo requisições HTTP. O Worker executa o
processamento em background e chama a API por HTTP. O Worker **não deve
acessar diretamente o banco** da API.

### Hospedagem sugerida

-   **EventBridge Scheduler** dispara a execução mensal.
-   **ECS Fargate Task** executa o Worker como tarefa sob demanda e
    encerra quando concluir.
-   Se já houver infraestrutura ECS disponível, essa é uma opção
    coerente com o ambiente.
-   **Lambda** também pode ser considerada se o processamento for curto,
    simples e couber confortavelmente nos limites de execução e memória.

A preferência discutida foi EventBridge Scheduler + ECS Fargate Task,
condicionada à infraestrutura existente. Ainda não foi tomada uma
decisão definitiva de hospedagem.

## 5. Biblioteca compartilhada de geração de PDF

Criar uma biblioteca, por exemplo `Reports.PdfGenerator`, para retirar a
responsabilidade de geração de PDF do BFF e permitir que BFF e Worker
usem a mesma implementação.

### Responsabilidades

-   Receber os dados já consultados pela aplicação chamadora.
-   Montar o conteúdo do relatório.
-   Gerar o PDF.
-   Retornar o conteúdo como `byte[]` ou `Stream`.

### O que não deve fazer

A biblioteca deve permanecer focada em geração de documento. Não deve:

-   Consultar banco de dados.
-   Fazer chamadas HTTP.
-   Enviar e-mails.
-   Ter dependência de AWS.
-   Conhecer o BFF ou o Worker.

Assim, BFF e Worker obtêm os dados por meio da API e passam esses dados
à biblioteca.

Exemplo conceitual:

``` csharp
public interface IReportPdfGenerator
{
    byte[] Generate(ReportData data);
}
```

A assinatura real pode ser síncrona ou assíncrona conforme a biblioteca
de PDF e a implementação. Se o PDF for grande, pode ser interessante
trabalhar com `Stream` para evitar cópias desnecessárias em memória.

## 6. Fluxos funcionais

### Fluxo A --- Download manual existente

1.  O especialista acessa o cliente na plataforma.
2.  Solicita o download do relatório.
3.  O BFF recebe a requisição.
4.  O BFF consulta a API de negócio para obter os dados do relatório.
5.  O BFF passa os dados para `Reports.PdfGenerator`.
6.  A biblioteca gera o PDF e retorna os bytes/stream.
7.  O BFF responde ao browser com `Content-Type: application/pdf` e os
    headers adequados para download.
8.  O especialista baixa o arquivo e compartilha por conta própria, por
    e-mail ou WhatsApp.

**Não participa desse fluxo:** Worker, EventBridge e API de e-mail.

### Fluxo B --- Envio manual por e-mail

1.  O especialista clica em "Enviar relatório por e-mail".

2.  O BFF recebe uma requisição, por exemplo:

    ``` http
    POST /clients/{clientId}/report/email
    ```

3.  O BFF consulta a API de negócio para obter os dados do relatório.

4.  O BFF gera o PDF usando `Reports.PdfGenerator`.

5.  O BFF chama a API de e-mail existente via HTTP, enviando
    destinatário, assunto, corpo e anexo PDF.

6.  A API de e-mail processa o envio.

7.  O BFF retorna à plataforma o resultado da solicitação.

**Regras:**

-   Esse fluxo não depende de o cliente estar inscrito no envio mensal.
-   O envio manual não altera nem substitui o envio automático do mês.
-   O envio manual pode ocorrer várias vezes, conforme a regra de
    negócio.
-   O Worker não participa desse fluxo.

Exemplo conceitual de chamada à API de e-mail:

``` text
BFF
 ├─ consulta dados na API de negócio
 ├─ gera PDF pela biblioteca compartilhada
 └─ POST para API de e-mail
      ├─ destinatário
      ├─ assunto
      ├─ corpo
      └─ anexo PDF
```

É recomendável avaliar se a API de e-mail aceita uma chave de
idempotência. Um timeout após a chamada não significa necessariamente
que o e-mail não foi enviado: o serviço pode ter concluído o envio e a
resposta ter se perdido.

### Fluxo C --- Ativar/desativar envio mensal

1.  O especialista marca ou desmarca o checkbox no perfil do cliente.
2.  O BFF recebe a alteração.
3.  O BFF chama a API de negócio.
4.  A API valida a solicitação e persiste a preferência no banco.
5.  A plataforma apresenta o estado atualizado da preferência.

A API de negócio é responsável pela persistência; BFF não deve escrever
diretamente no banco.

### Fluxo D --- Envio automático mensal

1.  O EventBridge Scheduler dispara a execução na data/horário
    configurados.
2.  O Scheduler inicia o `Reports.Worker` (a opção preliminar é uma ECS
    Fargate Task).
3.  O Worker consulta a API de negócio para obter os clientes com
    preferência mensal ativa.
4.  A listagem deve ser paginada para não depender de retornar todos os
    clientes em uma única resposta.
5.  Para cada cliente, o Worker executa o processamento individual:
    -   Revalida se a preferência ainda está ativa.
    -   Verifica se já existe um envio automático concluído ou em
        andamento para aquele cliente e mês de referência.
    -   Reserva o processamento de forma atômica.
    -   Consulta os dados necessários ao relatório na API.
    -   Gera o PDF usando `Reports.PdfGenerator`.
    -   Chama a API de e-mail com o PDF anexado.
    -   Atualiza o status do processamento por meio da API.
6.  Se um cliente falhar, o Worker registra a falha e continua com os
    demais.
7.  Ao terminar, a execução do Worker é encerrada.

Endpoint de listagem sugerido, apenas como exemplo:

``` http
GET /report-preferences/enabled?page=1&pageSize=100
```

A API também pode expor operações específicas para reservar, concluir e
falhar um envio. Os contratos definitivos ainda precisam ser desenhados.

## 7. Persistência sugerida

A preferência do cliente e o controle de cada tentativa/envio são
conceitos diferentes. Foi sugerido separar em duas tabelas.

### 7.1. `ClientReportPreference`

Armazena a configuração de envio mensal.

  -----------------------------------------------------------------------
  Campo                               Finalidade
  ----------------------------------- -----------------------------------
  `Id`                                Identificador do registro

  `ClientId`                          Identificador do cliente;
                                      idealmente único se houver uma
                                      preferência por cliente

  `IsEnabled`                         Indica se o envio mensal está ativo

  `CreatedAt`                         Data de criação

  `UpdatedAt`                         Data da última alteração

  `UpdatedBy`                         Identificador do especialista que
                                      alterou a preferência
  -----------------------------------------------------------------------

### 7.2. `ReportEmailDispatch`

Armazena o controle operacional dos envios, caso seja adotado
histórico/rastreabilidade.

  -----------------------------------------------------------------------
  Campo                               Finalidade
  ----------------------------------- -----------------------------------
  `Id`                                Identificador do
                                      processamento/envio

  `ClientId`                          Cliente destinatário

  `ReferenceMonth`                    Mês de referência do relatório

  `Type`                              Tipo: `Manual` ou `Automatic`

  `Status`                            Estado: `Pending`, `Processing`,
                                      `Sent` ou `Failed`

  `AttemptCount`                      Quantidade de tentativas

  `CreatedAt`                         Data de criação

  `SentAt`                            Data/hora em que a API de e-mail
                                      confirmou o processamento conforme
                                      seu contrato

  `ErrorMessage`                      Último erro, com cuidado para não
                                      armazenar dados sensíveis
  -----------------------------------------------------------------------

Campos adicionais que podem ser úteis:

-   `CorrelationId`
-   `IdempotencyKey`
-   `StartedAt`
-   `CompletedAt`
-   `LastError`
-   `LeaseExpiresAt`

### Regra de unicidade

Para evitar que o envio automático seja duplicado no mesmo mês,
considerar uma restrição única para:

``` text
(ClientId, ReferenceMonth, Type)
```

Essa regra de unicidade deve ser aplicada ao tipo `Automatic` (por
índice filtrado/estratégia equivalente, dependendo do banco). Envios
`Manual` não devem ser limitados por cliente/mês/tipo, pois podem
ocorrer várias vezes e são independentes do automático.

### Observação importante

O histórico de envios não foi colocado como requisito funcional
obrigatório pelo usuário. A tabela `ReportEmailDispatch` foi recomendada
principalmente para:

-   Evitar duplicidade.
-   Controlar tentativas e falhas.
-   Retomar processamentos interrompidos.
-   Ter rastreabilidade operacional.

Se a decisão for não manter histórico, ainda será necessário definir
algum mecanismo confiável de idempotência e controle do processamento
automático.

## 8. Idempotência, concorrência e recuperação

Mesmo com menos de 1.000 clientes e sem SQS, o processamento precisa ser
seguro contra execução duplicada e interrupções.

### Cenários que precisam ser tratados

-   O EventBridge pode iniciar uma nova execução enquanto uma execução
    anterior ainda está ativa.
-   O Worker pode falhar depois de reservar um cliente.
-   A aplicação pode cair depois de chamar a API de e-mail, mas antes de
    persistir o status `Sent`.
-   A API de e-mail pode ter enviado o e-mail e a resposta HTTP se
    perder.
-   A preferência pode ser desativada depois da listagem inicial e antes
    do processamento daquele cliente.

### Estratégias recomendadas

1.  **Reserva atômica:** a API deve oferecer uma operação que garanta
    que apenas uma execução consiga reservar o envio automático daquele
    cliente/mês.
2.  **Revalidação da preferência:** conferir novamente se o envio
    continua habilitado no momento do processamento.
3.  **Lease/expiração:** um registro em `Processing` deve ter prazo de
    validade (`LeaseExpiresAt`) para permitir recuperação se o Worker
    morrer.
4.  **Retomada:** uma execução futura deve conseguir identificar
    registros pendentes ou `Processing` expirados e decidir se podem ser
    retomados.
5.  **Idempotency key:** se possível, enviar uma chave estável para a
    API de e-mail, de modo que uma repetição da mesma solicitação não
    dispare outro e-mail.
6.  **Falha isolada por cliente:** erro em um cliente não deve
    interromper os demais.
7.  **Concorrência limitada:** começar sequencialmente ou com
    paralelismo baixo; aumentar somente após medir tempo total e
    capacidade dos serviços.

Não manter uma transação de banco aberta durante geração do PDF ou
chamadas HTTP. A reserva deve ser persistida em uma operação curta;
chamadas externas acontecem fora da transação; depois o status é
atualizado.

### Limite da garantia de entrega

Não assumir que `Sent` significa que o e-mail chegou à caixa de entrada.
O significado depende do contrato da API de e-mail. Em geral, o sistema
consegue afirmar que o serviço aceitou ou processou a solicitação, não
que o destinatário efetivamente recebeu ou leu a mensagem.

Também não é possível garantir *exactly-once* de ponta a ponta apenas
com uma tabela local, porque existe uma chamada externa ao serviço de
e-mail. Idempotência suportada pelo serviço externo é a melhor proteção
contra duplicação nesse limite.

## 9. Retentativas e tratamento de erros

-   Retentar falhas transitórias, como indisponibilidade temporária,
    throttling ou erros de rede.
-   Aplicar backoff exponencial e limitar o número de tentativas.
-   Não retentar indefinidamente erros permanentes, como endereço de
    e-mail inválido ou dados obrigatórios ausentes.
-   Registrar o erro de forma útil para diagnóstico, evitando incluir
    PDF, dados financeiros ou informações pessoais em logs.
-   Continuar o processamento dos próximos clientes depois de uma falha
    individual.
-   Definir como e quando registros `Failed` poderão ser reprocessados.

Sem SQS, o próprio Worker/API precisa implementar o controle de
tentativas e a recuperação. A simplicidade operacional deve ser avaliada
junto com a necessidade de não perder processamentos.

## 10. Por que não usar SQS inicialmente?

A decisão inicial foi não incluir SQS porque:

-   O volume estimado é baixo (menos de 1.000 clientes).
-   A rotina é mensal, não um fluxo contínuo de alto volume.
-   A intenção é manter a primeira versão simples.
-   A API de negócio e o Worker podem controlar o processamento, as
    tentativas e a retomada.

Isso não significa que SQS seja inadequado. Reavaliar se houver:

-   Crescimento relevante do volume.
-   Tempo total de execução muito alto.
-   Necessidade de desacoplar cada envio individual.
-   Necessidade de retentativas e DLQ gerenciadas por mensagem.
-   Necessidade de escalar consumidores independentemente.
-   Requisitos mais fortes de isolamento e recuperação.

## 11. Segurança e observabilidade

Os PDFs podem conter informações financeiras ou pessoais. Considerar:

-   Autorização no BFF para garantir que o especialista pode solicitar o
    envio daquele relatório.
-   Autenticação de serviço entre Worker e API.
-   Proteção dos endpoints de operação interna.
-   Validação do destinatário e das regras de envio.
-   Não registrar o conteúdo do PDF nem dados sensíveis nos logs.
-   Usar TLS nas chamadas HTTP.
-   Garantir que o anexo seja tratado com segurança pela API de e-mail.
-   Definir limites de tamanho para o PDF/anexo.

### Métricas e logs sugeridos

-   Quantidade de clientes encontrados com preferência ativa.
-   Quantidade de envios concluídos.
-   Quantidade de falhas.
-   Quantidade de tentativas.
-   Duração total da execução.
-   Duração por cliente.
-   Quantidade de registros `Processing` expirados/recuperados.
-   `CorrelationId` para rastrear o processamento entre Worker, API e
    serviço de e-mail.

Configurar alarmes para falha da execução, taxa elevada de erros e
processamento que exceda a janela esperada.

## 12. Decisões e pontos ainda em aberto

### Decisões/recomendações já discutidas

-   Manter o download manual atual.
-   Adicionar envio manual por e-mail pelo BFF.
-   Adicionar envio automático mensal por Worker.
-   O envio manual é independente do automático.
-   Preferência mensal persistida no banco existente por meio da API de
    negócio.
-   API de negócio permanece dona dos dados e regras; Worker e BFF não
    acessam o banco diretamente.
-   Extrair geração de PDF para uma biblioteca .NET compartilhada.
-   Criar `Reports.Worker` como aplicação independente da API.
-   Consumir a API de e-mail existente por HTTP.
-   Não usar SQS na primeira versão, considerando o volume e a intenção
    de simplicidade.
-   Considerar EventBridge Scheduler + ECS Fargate Task para a execução
    mensal.

### Questões que precisam de decisão

1.  **Agendamento:** em qual dia e horário do mês o envio será
    executado? Qual fuso horário deve ser considerado?
2.  **Mês de referência:** o relatório enviado representa o mês
    anterior, o mês corrente ou outra janela?
3.  **Janela de envio:** todos os clientes devem receber no mesmo dia ou
    o processamento pode ser distribuído?
4.  **Desativação:** se o cliente desativar a preferência antes do
    processamento, o envio daquele mês deve ser cancelado? A
    recomendação discutida é revalidar a preferência e não enviar se
    estiver desativada.
5.  **Alteração de e-mail:** qual endereço será usado: o endereço atual
    cadastrado no momento do processamento ou um snapshot do endereço no
    momento da inscrição?
6.  **Falhas:** qual política de tentativas, intervalo e reprocessamento
    será adotada?
7.  **API de e-mail:** suporta chave de idempotência? Como informa
    sucesso e falha? Aceita PDF em Base64, multipart ou outro contrato?
8.  **Histórico:** há necessidade de histórico visível para o
    especialista/cliente ou a tabela de dispatch será somente
    operacional?
9.  **Hospedagem:** ECS Fargate Task ou Lambda? A recomendação depende
    da infraestrutura já existente e do tempo real de execução.
10. **Autorização:** como o Worker se autentica na API e quais endpoints
    serão exclusivos para comunicação serviço-a-serviço?
11. **Retenção:** por quanto tempo manter registros de dispatch e
    mensagens de erro?
12. **Duplicidade:** como tratar uma falha ambígua em que a API de
    e-mail pode ter enviado, mas o Worker não recebeu a confirmação?

## 13. Contratos de API a desenhar

Os caminhos abaixo são exemplos para orientar a discussão, não contratos
já existentes ou aprovados.

### Preferência mensal

``` http
PUT /clients/{clientId}/report-preference
Content-Type: application/json

{
  "isEnabled": true
}
```

Consulta:

``` http
GET /clients/{clientId}/report-preference
```

### Listagem para o Worker

``` http
GET /report-preferences/enabled?page=1&pageSize=100
```

A resposta deve trazer os identificadores necessários para processar
cada cliente, sem expor dados além do necessário.

### Reserva de envio automático

Exemplo conceitual:

``` http
POST /report-email-dispatches/automatic/claim
Content-Type: application/json

{
  "clientId": "client-id",
  "referenceMonth": "2026-11"
}
```

A operação deve ser atômica e informar se o Worker conseguiu reservar o
processamento ou se ele já existe/está sendo executado.

### Atualização de resultado

Exemplos conceituais:

``` http
POST /report-email-dispatches/{dispatchId}/sent
```

``` http
POST /report-email-dispatches/{dispatchId}/failed
Content-Type: application/json

{
  "errorCode": "EMAIL_PROVIDER_UNAVAILABLE",
  "errorMessage": "..."
}
```

Os endpoints finais devem refletir os padrões da API existente e evitar
criar operações excessivamente granulares sem necessidade.

## 14. Direção de implementação sugerida

Uma sequência pragmática para evoluir a solução:

1.  Confirmar contratos e comportamento da API de e-mail, especialmente
    idempotência e anexos.
2.  Extrair a geração de PDF do BFF para `Reports.PdfGenerator`,
    mantendo o download funcionando.
3.  Implementar o envio manual por e-mail no BFF, reutilizando a
    biblioteca.
4.  Implementar persistência e consulta da preferência mensal na API de
    negócio.
5.  Definir o modelo mínimo de controle/idempotência dos envios
    automáticos.
6.  Criar o `Reports.Worker` e implementar processamento de um cliente
    por vez.
7.  Implementar listagem paginada, reserva atômica, revalidação da
    preferência e atualização de status via API.
8.  Configurar EventBridge Scheduler e a execução do Worker na
    infraestrutura escolhida.
9.  Adicionar logs, métricas, alarmes e testes de falha/duplicidade.
10. Validar o comportamento com poucos clientes antes de habilitar para
    todos.

## 15. Instruções para continuidade da conversa

Ao continuar este trabalho:

-   Trate este documento como o contexto consolidado da discussão, mas
    diferencie claramente **decisões tomadas**, **recomendações** e
    **pontos em aberto**.
-   Não assuma que os exemplos de endpoints, nomes de tabelas ou campos
    já existem no sistema.
-   Não introduza SQS sem explicar qual problema concreto ele resolve em
    relação ao volume e aos requisitos atuais.
-   Preserve a separação entre envio manual e envio automático.
-   Preserve a API de negócio como proprietária do banco.
-   Priorize uma solução simples, mas não ignore idempotência, falhas
    ambíguas e recuperação do Worker.
-   Use exemplos em .NET/C# e AWS quando forem úteis.
-   Antes de propor uma arquitetura mais complexa, valide se há um
    requisito que realmente a justifique.
