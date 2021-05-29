## Introdução ao Hystrix

# 1. Introdução
Um sistema distribuído típico consiste em muitos serviços colaborando juntos.

Esses serviços estão sujeitos a falhas ou atrasos nas respostas. Se um serviço falhar, pode afetar outros serviços, afetando o desempenho e, possivelmente, tornando outras partes do aplicativo inacessíveis ou, na pior das hipóteses, derrubar todo o aplicativo.

Claro, existem soluções disponíveis que ajudam a tornar os aplicativos resilientes e tolerantes a falhas - uma dessas estruturas é Hystrix.

A biblioteca da estrutura Hystrix ajuda a controlar a interação entre os serviços, fornecendo tolerância a falhas e tolerância à latência. Ele melhora a resiliência geral do sistema, isolando os serviços com falha e interrompendo o efeito em cascata das falhas.

Nesta série de postagens, começaremos examinando como a Hystrix vem ao resgate quando um serviço ou sistema falha e o que a Hystrix pode realizar nessas circunstâncias.

# 2. Exemplo Simples
A forma como o Hystrix fornece tolerância a falhas e latência é isolar e agrupar chamadas para serviços remotos.

Neste exemplo simples, envolvemos uma chamada no método run() do HystrixCommand:

```
class CommandHelloWorld extends HystrixCommand<String> {

    private String name;

    CommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() {
        return "Hello " + name + "!";
    }
}
```

e executamos a chamada da seguinte forma:

```
@Test
public void givenInputBobAndDefaultSettings_whenCommandExecuted_thenReturnHelloBob(){
    assertThat(new CommandHelloWorld("Bob").execute(), equalTo("Hello Bob!"));
}
```

3. Configuração do Maven
Para usar Hystrix em projetos Maven, precisamos ter a dependência hystrix-core e rxjava-core do Netflix no projeto pom.xml:

```
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-core</artifactId>
    <version>1.5.4</version>
</dependency>
```

A versão mais recente sempre pode ser encontrada aqui:
https://search.maven.org/classic/#search%7Cga%7C1%7Ca%3A%22rxjava-core%22

```
<dependency>
    <groupId>com.netflix.rxjava</groupId>
    <artifactId>rxjava-core</artifactId>
    <version>0.20.7</version>
</dependency>
```

# 4. Configurando o serviço remoto
Vamos começar simulando um exemplo do mundo real.

No exemplo abaixo, a classe RemoteServiceTestSimulator representa um serviço em um servidor remoto. Possui um método que responde com uma mensagem após um determinado período de tempo. Podemos imaginar que essa espera é uma simulação de um processo demorado no sistema remoto, resultando em um atraso na resposta ao serviço de chamada:

```
class RemoteServiceTestSimulator {

    private long wait;

    RemoteServiceTestSimulator(long wait) throws InterruptedException {
        this.wait = wait;
    }

    String execute() throws InterruptedException {
        Thread.sleep(wait);
        return "Success";
    }
}
```

E aqui está nosso cliente de amostra que chama o RemoteServiceTestSimulator.

A chamada para o serviço é isolada e agrupada no método run() de um HystrixCommand. É esse invólucro que fornece a resiliência que mencionamos acima:

```
class RemoteServiceTestCommand extends HystrixCommand<String> {

    private RemoteServiceTestSimulator remoteService;

    RemoteServiceTestCommand(Setter config, RemoteServiceTestSimulator remoteService) {
        super(config);
        this.remoteService = remoteService;
    }

    @Override
    protected String run() throws Exception {
        return remoteService.execute();
    }
}
```

A chamada é executada chamando o método execute() em uma instância do objeto RemoteServiceTestCommand.

O teste a seguir demonstra como isso é feito:

```
@Test
public void givenSvcTimeoutOf100AndDefaultSettings_whenRemoteSvcExecuted_thenReturnSuccess()
  throws InterruptedException {

    HystrixCommand.Setter config = HystrixCommand
      .Setter
      .withGroupKey(HystrixCommandGroupKey.Factory.asKey("RemoteServiceGroup2"));
    
    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(100)).execute(),
      equalTo("Success"));
}
```

Até agora, vimos como envolver chamadas de serviço remoto no objeto HystrixCommand. Na seção abaixo, vamos ver como lidar com uma situação em que o serviço remoto começa a se deteriorar.

# 5. Trabalho com serviço remoto e programação defensiva
5.1. Programação defensiva com tempo limite
É uma prática geral de programação definir tempos limite para chamadas para serviços remotos.

Vamos começar examinando como definir o tempo limite no HystrixCommand e como isso ajuda a fazer um curto-circuito:

```
@Test
public void givenSvcTimeoutOf5000AndExecTimeoutOf10000_whenRemoteSvcExecuted_thenReturnSuccess()
  throws InterruptedException {

    HystrixCommand.Setter config = HystrixCommand
      .Setter
      .withGroupKey(HystrixCommandGroupKey.Factory.asKey("RemoteServiceGroupTest4"));

    HystrixCommandProperties.Setter commandProperties = HystrixCommandProperties.Setter();
    commandProperties.withExecutionTimeoutInMilliseconds(10_000);
    config.andCommandPropertiesDefaults(commandProperties);

    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(500)).execute(),
      equalTo("Success"));
}
```

No teste acima, estamos atrasando a resposta do serviço definindo o tempo limite para 500 ms. Também estamos configurando o tempo limite de execução no HystrixCommand para 10.000 ms, permitindo tempo suficiente para o serviço remoto responder.

Agora vamos ver o que acontece quando o tempo limite de execução é menor que a chamada de tempo limite do serviço:

```
@Test(expected = HystrixRuntimeException.class)
public void givenSvcTimeoutOf15000AndExecTimeoutOf5000_whenRemoteSvcExecuted_thenExpectHre()
  throws InterruptedException {

    HystrixCommand.Setter config = HystrixCommand
      .Setter
      .withGroupKey(HystrixCommandGroupKey.Factory.asKey("RemoteServiceGroupTest5"));

    HystrixCommandProperties.Setter commandProperties = HystrixCommandProperties.Setter();
    commandProperties.withExecutionTimeoutInMilliseconds(5_000);
    config.andCommandPropertiesDefaults(commandProperties);

    new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(15_000)).execute();
}
```

Observe como baixamos a barra e definimos o tempo limite de execução para 5.000 ms.

Esperamos que o serviço responda em 5.000 ms, enquanto configuramos o serviço para responder após 15.000 ms. Se você perceber quando executar o teste, o teste será encerrado após 5.000 ms em vez de esperar 15.000 ms e lançará uma HystrixRuntimeException.

Isso demonstra como o Hystrix não espera mais do que o tempo limite configurado por uma resposta. Isso ajuda a tornar o sistema protegido pelo Hystrix mais responsivo.

Nas seções a seguir, veremos como definir o tamanho do pool de threads que evita que as threads se esgotem e discutiremos seus benefícios.

### 5.2 Programação defensiva com pool limitado de threads
Definir tempos limite para chamada de serviço não resolve todos os problemas associados aos serviços remotos.

Quando um serviço remoto começa a responder lentamente, um aplicativo típico continuará a chamar esse serviço remoto.

O aplicativo não sabe se o serviço remoto está íntegro ou não e novos threads são gerados toda vez que uma solicitação chega. Isso fará com que threads em um servidor que já esteja com problemas sejam usados.

Não queremos que isso aconteça, pois precisamos desses threads para outras chamadas remotas ou processos em execução em nosso servidor e também queremos evitar o aumento da utilização da CPU.

Vamos ver como definir o tamanho do pool de threads no HystrixCommand:

```
@Test
public void givenSvcTimeoutOf500AndExecTimeoutOf10000AndThreadPool_whenRemoteSvcExecuted
  _thenReturnSuccess() throws InterruptedException {

    HystrixCommand.Setter config = HystrixCommand
      .Setter
      .withGroupKey(HystrixCommandGroupKey.Factory.asKey("RemoteServiceGroupThreadPool"));

    HystrixCommandProperties.Setter commandProperties = HystrixCommandProperties.Setter();
    commandProperties.withExecutionTimeoutInMilliseconds(10_000);
    config.andCommandPropertiesDefaults(commandProperties);
    config.andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
      .withMaxQueueSize(10)
      .withCoreSize(3)
      .withQueueSizeRejectionThreshold(10));

    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(500)).execute(),
      equalTo("Success"));
}
```

No teste acima, estamos configurando o tamanho máximo da fila, o tamanho da fila principal e o tamanho de rejeição da fila. O Hystrix começará a rejeitar as solicitações quando o número máximo de threads atingir 10 e a fila de tarefas atingir o tamanho 10.

O tamanho do núcleo é o número de threads que sempre permanecem ativos no pool de threads.

### 5.3. Programação defensiva com padrão de curto-circuito
No entanto, ainda há uma melhoria que podemos fazer nas chamadas de serviço remoto.

Vamos considerar o caso em que o serviço remoto começou a falhar.

Não queremos continuar disparando solicitações e desperdiçar recursos. O ideal é parar de fazer solicitações por um determinado período de tempo para dar ao serviço tempo para se recuperar antes de retomar as solicitações. Isso é chamado de padrão de curto-circuito.

Vamos ver como Hystrix implementa esse padrão:

```
@Test
public void givenCircuitBreakerSetup_whenRemoteSvcCmdExecuted_thenReturnSuccess()
  throws InterruptedException {

    HystrixCommand.Setter config = HystrixCommand
      .Setter
      .withGroupKey(HystrixCommandGroupKey.Factory.asKey("RemoteServiceGroupCircuitBreaker"));

    HystrixCommandProperties.Setter properties = HystrixCommandProperties.Setter();
    properties.withExecutionTimeoutInMilliseconds(1000);
    properties.withCircuitBreakerSleepWindowInMilliseconds(4000);
    properties.withExecutionIsolationStrategy
     (HystrixCommandProperties.ExecutionIsolationStrategy.THREAD);
    properties.withCircuitBreakerEnabled(true);
    properties.withCircuitBreakerRequestVolumeThreshold(1);

    config.andCommandPropertiesDefaults(properties);
    config.andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
      .withMaxQueueSize(1)
      .withCoreSize(1)
      .withQueueSizeRejectionThreshold(1));

    assertThat(this.invokeRemoteService(config, 10_000), equalTo(null));
    assertThat(this.invokeRemoteService(config, 10_000), equalTo(null));
    assertThat(this.invokeRemoteService(config, 10_000), equalTo(null));

    Thread.sleep(5000);

    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(500)).execute(),
      equalTo("Success"));

    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(500)).execute(),
      equalTo("Success"));

    assertThat(new RemoteServiceTestCommand(config, new RemoteServiceTestSimulator(500)).execute(),
      equalTo("Success"));
}
```

```
public String invokeRemoteService(HystrixCommand.Setter config, int timeout)
  throws InterruptedException {

    String response = null;

    try {
        response = new RemoteServiceTestCommand(config,
          new RemoteServiceTestSimulator(timeout)).execute();
    } catch (HystrixRuntimeException ex) {
        System.out.println("ex = " + ex);
    }

    return response;
}
```

No teste acima, definimos diferentes propriedades do disjuntor. Os mais importantes são:

- O CircuitBreakerSleepWindow definido para 4.000 ms. Isso configura a janela do disjuntor e define o intervalo de tempo após o qual a solicitação ao serviço remoto será retomada;
- O CircuitBreakerRequestVolumeThreshold que é definido como 1 e define o número mínimo de solicitações necessárias antes que a taxa de falha seja considerada.

Com as configurações acima em vigor, nosso HystrixCommand agora será desarmado após duas solicitações com falha. A terceira solicitação nem chegará ao serviço remoto, embora tenhamos definido o atraso do serviço em 500 ms, o Hystrix entrará em curto-circuito e nosso método retornará nulo como resposta.

Em seguida, adicionaremos um Thread.sleep (5000) para cruzar o limite da janela de sono que definimos. Isso fará com que o Hystrix feche o circuito e as solicitações subsequentes serão transmitidas com sucesso.

# 6. Conclusão
Em resumo, o Hystrix foi projetado para:

1 - Fornece proteção e controle sobre falhas e latência de
serviços normalmente acessados ​​pela rede;

2 - Pare a cascata de falhas resultantes de alguns dos serviços estarem inativos;

3 - Falha rápido e se recupera rapidamente;

4 - Degrade-se graciosamente sempre que possível;

5 - Monitoramento em tempo real e alerta do centro de comando sobre falhas
No próximo post, veremos como combinar os benefícios do Hystrix com o framework Spring.
